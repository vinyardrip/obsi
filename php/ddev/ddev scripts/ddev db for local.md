
```bash
#!/usr/bin/env bash
# =================================================================
# УНИВЕРСАЛЬНЫЙ скрипт для "освежения" локального сайта данными с продакшена.
# ВЕРСИЯ 3.1: Добавлена автоматическая остановка Devilbox для предотвращения конфликтов.
#
# Что делает:
# 1. ОБЯЗАТЕЛЬНО останавливает Devilbox.
# 2. Создает резервную копию текущей локальной БД.
# 3. В зависимости от типа CMS, выбирает правильный метод замены URL.
# 4. Импортирует дамп с продакшена, затирая старую локальную БД.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DB_LOCAL_PATH="$HOME/projects/work/db_local"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
# Добавляем путь к Devilbox для проверки
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"

# --- Утилиты и цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО РАБОТЫ ---
printf "%s--- Обновление локальной БД с продакшена ---%s\n" "${BLUE}" "${NC}"

# === НОВЫЙ БЛОК: Предотвращение конфликта портов ===
if [ -d "$DEVILBOX_PATH_ROOT" ]; then
    printf "\n%s--- Проверка состояния Devilbox ---%s\n" "${BLUE}" "${NC}"
    printf "Чтобы избежать конфликта портов, Devilbox будет остановлен...\n"
    # Выполняем команду в под-оболочке, чтобы не менять текущую директорию
    (cd "$DEVILBOX_PATH_ROOT" && docker-compose down >/dev/null 2>&1)
    printf "✔ Devilbox успешно остановлен.\n"
fi
# === КОНЕЦ НОВОГО БЛОКА ===

# === ШАГ 1: ВЫБОР ТИПА ПРОЕКТА ===
printf "\n%s--- Выбор типа проекта ---%s\n" "${BLUE}" "${NC}"
echo "1. WordPress (безопасная замена URL через WP-CLI)"
echo "2. Joomla (замена URL в SQL-файле)"
echo "3. OpenCart (замена URL в SQL-файле)"
echo "4. Другой PHP проект с БД (замена URL в SQL-файле)"
read -p "Выберите тип проекта [1]: " PROJECT_CHOICE
PROJECT_CHOICE=${PROJECT_CHOICE:-1}

case "$PROJECT_CHOICE" in
    1) CMS_TYPE="wordpress";;
    2) CMS_TYPE="joomla";;
    3) CMS_TYPE="opencart";;
    4) CMS_TYPE="php";;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac
printf "✔ Выбран тип: %s\n" "$CMS_TYPE"

# --- ОБЩИЕ НАСТРОЙКИ ПРОЕКТА ---
read -p "1. Введите имя проекта DDEV (например, grandmebel): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ ! -d "$DDEV_PROJECT_PATH" ]; then
  printf "\n%sОШИБКА: Директория проекта не найдена по пути:%s\n  %s\n" "${RED}" "${NC}" "${DDEV_PROJECT_PATH}"; exit 1
fi

read -p "   Доменная зона продакшена (.by, .com, etc) [.by]: " PROD_TLD
PROD_TLD=${PROD_TLD:-"by"}; RAW_DOMAIN="${PROJECT_NAME}.${PROD_TLD//./}"

printf "Проверяем статус проекта '%s'...\n" "$PROJECT_NAME"
if ! ddev describe "$PROJECT_NAME" | grep -q "OK"; then
    printf "\n%sВНИМАНИЕ: Проект '%s' не запущен.%s\n" "${YELLOW}" "$PROJECT_NAME" "${NC}"
    read -p "Запустить его сейчас? (y/n): " CONFIRM_START
    if [ "$CONFIRM_START" = "y" ]; then ddev start -p "$PROJECT_NAME"; else printf "%sОперация отменена.%s\n" "${RED}" "${NC}"; exit 1; fi
fi
printf "✔ Проект запущен и готов к работе.\n"

read -p "2. Имя файла БД из '${DB_PROD_PATH}' (без .sql) [${PROJECT_NAME}]: " DB_FILENAME_BASE
DB_FILENAME_BASE=${DB_FILENAME_BASE:-${PROJECT_NAME}}
INPUT_FILE_PATH="${DB_PROD_PATH}/${DB_FILENAME_BASE}.sql"
if [ ! -f "$INPUT_FILE_PATH" ]; then
  printf "\n%sОШИБКА: Файл базы данных с продакшена не найден:%s %s\n" "${RED}" "${NC}" "$INPUT_FILE_PATH"; exit 1;
fi

read -p "3. Выберите протокол продакшена (1=https, 2=http) [1]: " PROTOCOL_CHOICE
PROTOCOL_CHOICE=${PROTOCOL_CHOICE:-1}
case "$PROTOCOL_CHOICE" in
    1) PROD_URL="https://${RAW_DOMAIN}" ;;
    2) PROD_URL="http://${RAW_DOMAIN}" ;;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac

LOCAL_URL="https://${PROJECT_NAME}.lo"

# =================================================================
#  🚀 Процесс обновления
# =================================================================

# Переходим в директорию проекта для корректной работы ddev
cd "$DDEV_PROJECT_PATH"

# ---- ОБЯЗАТЕЛЬНОЕ РЕЗЕРВНОЕ КОПИРОВАНИЕ ----
printf "\n%sШаг 1: Создание резервной копии текущей локальной базы данных...%s\n" "${BLUE}" "${NC}"
mkdir -p "$DB_LOCAL_PATH"
CURRENT_DATE=$(date +%Y-%m-%d)
BACKUP_FILENAME="${PROJECT_NAME}_${CURRENT_DATE}_local_backup.sql"
BACKUP_FILE_PATH="${DB_LOCAL_PATH}/${BACKUP_FILENAME}"
ddev export-db --gzip=false > "$BACKUP_FILE_PATH"
printf "✔ Резервная копия успешно сохранена в:\n%s%s%s\n" "${YELLOW}" "${BACKUP_FILE_PATH}" "${NC}"

# --- Сводка и финальное подтверждение ---
printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Проект для обновления:      ${GREEN}${PROJECT_NAME} (${CMS_TYPE})${NC}"
echo "Источник данных (прод):       ${GREEN}${INPUT_FILE_PATH}${NC}"
printf "\n%sВНИМАНИЕ: Текущая локальная база данных проекта '%s' будет ПОЛНОСТЬЮ ПЕРЕЗАПИСАНА!%s\n" "${RED}" "$PROJECT_NAME" "${NC}"
echo "Резервная копия вашей старой локальной БД уже создана."
echo "---------------------------"
read -p "Продолжить обновление? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# === УСЛОВНАЯ ЛОГИКА ОБНОВЛЕНИЯ ===
if [ "$CMS_TYPE" = "wordpress" ]; then
    # --- Путь для WordPress ---
    printf "\n%sШаг 2: Импорт продакшен-дампа в DDEV...%s\n" "${BLUE}" "${NC}"
    ddev import-db --file="$INPUT_FILE_PATH"
    printf "✔ База данных с продакшена импортирована.\n"

    printf "\n%sШаг 3: Замена URL на локальные значения (WP-CLI)...%s\n" "${BLUE}" "${NC}"
    ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root"
    printf "✔ URL успешно заменены. Ваш локальный сайт обновлен.\n"
else
    # --- Путь для Joomla, OpenCart и других PHP проектов ---
    printf "\n%sШаг 2: Подготовка и замена URL в SQL-файле...%s\n" "${BLUE}" "${NC}"
    TEMP_SQL_FILE="/tmp/${PROJECT_NAME}_refresh_$(date +%s).sql"
    cp "$INPUT_FILE_PATH" "$TEMP_SQL_FILE"
    sed -i "s|${PROD_URL}|${LOCAL_URL}|g" "$TEMP_SQL_FILE"
    printf "✔ Временный SQL-файл с замененными URL подготовлен.\n"

    printf "\n%sШаг 3: Импорт подготовленного дампа в DDEV...%s\n" "${BLUE}" "${NC}"
    ddev import-db --file="$TEMP_SQL_FILE"
    rm "$TEMP_SQL_FILE" # Очищаем временный файл
    printf "✔ База данных импортирована и обновлена.\n"
fi

printf "\n%s🎉 Готово! 🎉%s\n" "${GREEN}" "${NC}"
printf "Локальный сайт '%s' успешно обновлен данными с продакшена.\n" "$PROJECT_NAME"
printf "Он доступен по адресу: %s%s%s\n" "${YELLOW}" "${LOCAL_URL}" "${NC}"
ddev launch

# Возвращаемся в исходную директорию
cd - > /dev/null

exit 0