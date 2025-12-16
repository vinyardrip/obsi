```
#!/usr/bin/env bash
# =================================================================
# СКРИПТ: Обновление локальной DDEV из Продакшена (v3.4)
# Логика:
# 1. Бэкап текущей локалки в файл -> db_local (для истории).
# 2. Бэкап текущей локалки в Snapshot (для быстрой страховки).
# 3. Импорт базы с прода (из db_prod).
# 4. Авто-запуск сайта в браузере.
# =================================================================

set -e

# --- ⚙️ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DB_LOCAL_PATH="$HOME/projects/work/db_local"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"

# --- Цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО ---
printf "%s--- Обновление локальной БД с продакшена ---%s\n" "${BLUE}" "${NC}"

# 1. Проверка Devilbox
if [ -d "$DEVILBOX_PATH_ROOT" ]; then
    (cd "$DEVILBOX_PATH_ROOT" && docker-compose down >/dev/null 2>&1)
fi

# 2. Выбор CMS
echo "1. WordPress (WP-CLI - безопасно)"
echo "2. Другое (Joomla / OpenCart / PHP)"
read -p "Тип [1]: " PROJECT_CHOICE
PROJECT_CHOICE=${PROJECT_CHOICE:-1}
case "$PROJECT_CHOICE" in
    1) CMS_TYPE="wordpress";;
    *) CMS_TYPE="php";;
esac

# 3. Настройки проекта
read -p "Имя проекта DDEV: " PROJECT_NAME
[ -z "$PROJECT_NAME" ] && { echo "Ошибка имени"; exit 1; }

DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
[ ! -d "$DDEV_PROJECT_PATH" ] && { echo "Папка проекта не найдена"; exit 1; }

# Проверка запуска
if ! ddev describe "$PROJECT_NAME" | grep -q "OK"; then
    read -p "Проект остановлен. Запустить? (y/n): " CONFIRM_START
    if [ "$CONFIRM_START" = "y" ]; then ddev start -p "$PROJECT_NAME"; else exit 1; fi
fi

# Домены
read -p "Доменная зона прода (.by): " PROD_TLD
PROD_TLD=${PROD_TLD:-"by"}; RAW_DOMAIN="${PROJECT_NAME}.${PROD_TLD//./}"
LOCAL_URL="https://${PROJECT_NAME}.lo"

# Протокол
read -p "Протокол прода (1=https, 2=http) [1]: " PROTOCOL_CHOICE
if [ "$PROTOCOL_CHOICE" = "2" ]; then PROD_URL="http://${RAW_DOMAIN}"; else PROD_URL="https://${RAW_DOMAIN}"; fi

# Файл источника (ПРОД)
read -p "Файл из db_prod (без .sql) [${PROJECT_NAME}]: " DB_FILENAME_BASE
DB_FILENAME_BASE=${DB_FILENAME_BASE:-${PROJECT_NAME}}
INPUT_FILE_PATH="${DB_PROD_PATH}/${DB_FILENAME_BASE}.sql"
[ ! -f "$INPUT_FILE_PATH" ] && { echo "Файл прода не найден: $INPUT_FILE_PATH"; exit 1; }

# =================================================================
#  🚀 ПРОЦЕСС
# =================================================================

cd "$DDEV_PROJECT_PATH"

printf "\n%sШаг 1: Бэкап в файл (в папку db_local)...%s\n" "${BLUE}" "${NC}"
mkdir -p "$DB_LOCAL_PATH"
BACKUP_FILE="${DB_LOCAL_PATH}/${PROJECT_NAME}_before_refresh_$(date +%Y-%m-%d).sql.gz"
ddev export-db --gzip=true --file="$BACKUP_FILE"
printf "✔ Сохранен файл: %s\n" "$BACKUP_FILE"

printf "\n%sШаг 2: Страховочный Snapshot (для быстрого отката)...%s\n" "${BLUE}" "${NC}"
ddev delete-snapshot auto_before_refresh >/dev/null 2>&1 || true
ddev snapshot --name "auto_before_refresh"
printf "✔ Снимок создан. Откат, если что: ddev restore-snapshot auto_before_refresh\n"

printf "\n%sВНИМАНИЕ: Локальная БД будет заменена данными из:%s\n%s\n" "${RED}" "${NC}" "$INPUT_FILE_PATH"
read -p "Погнали? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then echo "Отмена."; exit 1; fi

if [ "$CMS_TYPE" = "wordpress" ]; then
    # --- WORDPRESS ---
    printf "\n%sШаг 3: Импорт и WP-CLI замена...%s\n" "${BLUE}" "${NC}"
    ddev import-db --file="$INPUT_FILE_PATH"
    ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root"
else
    # --- ДРУГИЕ ---
    printf "\n%sШаг 3: Подготовка файла и Импорт...%s\n" "${BLUE}" "${NC}"
    TEMP_SQL="/tmp/${PROJECT_NAME}_refresh.sql"
    sed "s|${PROD_URL}|${LOCAL_URL}|g" "$INPUT_FILE_PATH" > "$TEMP_SQL"
    ddev import-db --file="$TEMP_SQL"
    rm "$TEMP_SQL"
fi

printf "\n%s🎉 Готово! 🎉%s\n" "${GREEN}" "${NC}"
printf "Локальный сайт '%s' успешно обновлен данными с продакшена.\n" "$PROJECT_NAME"
printf "Он доступен по адресу: %s%s%s\n" "${YELLOW}" "${LOCAL_URL}" "${NC}"
ddev launch

exit 0
```

```



## 📦 Обновление локалки из продакшена
Команда: `./refresh-local.sh` (запускать из терминала)

Перед первым запуском:
- `chmod +x refresh-local.sh`
- Проверить пути в шапке скрипта (DB_PROD_PATH, DB_LOCAL_PATH, DDEV_PROJECTS_BASE_PATH)
- Убедиться, что дамп `имя_проекта.sql` лежит в `db_prod`

```
ddev-refresh
```

Порядок действий скрипта:
1. Останавливает Devilbox (если запущен)
2. Спрашивает тип CMS и имя проекта
3. Делает бэкап текущей локальной БД в `db_local`
4. Импортирует прод-дамп
5. Заменяет прод-URL на локальный (`*.lo`) нужным способом
6. Открывает обновлённый сайт в браузере

Готово! Локалка = продакшен без ручной возни.