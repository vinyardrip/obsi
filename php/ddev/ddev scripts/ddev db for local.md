
```bash
#!/usr/bin/env bash
# =================================================================
# Скрипт для "освежения" локального сайта данными с продакшена.
# ВЕРСИЯ 2.0: "Безопасное обновление" с обязательным бэкапом.
#
# Что делает:
# 1. ОБЯЗАТЕЛЬНО создает резервную копию текущей локальной БД.
# 2. Импортирует дамп с продакшена, затирая старую локальную БД.
# 3. Корректно заменяет продакшен-URL на локальные с помощью WP-CLI.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DB_LOCAL_PATH="$HOME/projects/work/db_local"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"

# --- Утилиты и цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО РАБОТЫ ---
printf "%s--- Обновление локальной БД с продакшена ---%s\n" "${BLUE}" "${NC}"

# 1. Запрос ключевой информации
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

# ---- НОВЫЙ ОБЯЗАТЕЛЬНЫЙ ШАГ: РЕЗЕРВНОЕ КОПИРОВАНИЕ ----
printf "\n%sШаг 1: Создание резервной копии текущей локальной базы данных...%s\n" "${BLUE}" "${NC}"
mkdir -p "$DB_LOCAL_PATH"
CURRENT_DATE=$(date +%Y-%m-%d)
BACKUP_FILENAME="${PROJECT_NAME}_${CURRENT_DATE}_local_backup.sql"
BACKUP_FILE_PATH="${DB_LOCAL_PATH}/${BACKUP_FILENAME}"

ddev export-db --gzip=false > "$BACKUP_FILE_PATH"
printf "✔ Резервная копия успешно сохранена в:\n%s%s%s\n" "${YELLOW}" "${BACKUP_FILE_PATH}" "${NC}"
# ---- КОНЕЦ НОВОГО ШАГА ----

# --- Сводка и финальное подтверждение ---
printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Проект для обновления:      ${GREEN}${PROJECT_NAME}${NC}"
echo "Источник данных (прод):       ${GREEN}${INPUT_FILE_PATH}${NC}"
printf "\n%sВНИМАНИЕ: Текущая локальная база данных проекта '%s' будет ПОЛНОСТЬЮ ПЕРЕЗАПИСАНА!%s\n" "${RED}" "$PROJECT_NAME" "${NC}"
echo "Резервная копия вашей старой локальной БД уже создана."
echo "---------------------------"
read -p "Продолжить обновление? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi


printf "\n%sШаг 2: Импорт продакшен-дампа в DDEV...%s\n" "${BLUE}" "${NC}"
ddev import-db --file="$INPUT_FILE_PATH"
printf "✔ База данных с продакшена импортирована.\n"

printf "\n%sШаг 3: Замена URL на локальные значения...%s\n" "${BLUE}" "${NC}"
ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root"
printf "✔ URL успешно заменены. Ваш локальный сайт обновлен.\n"

printf "\n%s🎉 Готово! 🎉%s\n" "${GREEN}" "${NC}"
printf "Локальный сайт '%s' успешно обновлен данными с продакшена.\n" "$PROJECT_NAME"
printf "Он доступен по адресу: %s%s%s\n" "${YELLOW}" "${LOCAL_URL}" "${NC}"
ddev launch

# Возвращаемся в исходную директорию
cd - > /dev/null

exit 0