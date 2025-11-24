
```bash
#!/usr/bin/env bash
# =================================================================
# УНИВЕРСАЛЬНЫЙ скрипт для подготовки локальной DDEV-БД к выгрузке на продакшен.
# ВЕРСИЯ 3.0: Поддерживает WordPress, Joomla, OpenCart и другие PHP-проекты.
#
# Что делает:
# - Спрашивает имя и тип проекта.
# - Выбирает правильный метод замены URL в зависимости от CMS.
# - Экспортирует базу данных в готовый для продакшена файл.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
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
printf "%s--- Подготовка БД для продакшена ---%s\n" "${BLUE}" "${NC}"

# === НОВЫЙ БЛОК: Предотвращение конфликта портов ===
if [ -d "$DEVILBOX_PATH_ROOT" ]; then
    printf "\n%s--- Проверка состояния Devilbox ---%s\n" "${BLUE}" "${NC}"
    printf "Чтобы избежать конфликта портов, Devilbox будет остановлен...\n"
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

# 1. Запрос ключевой информации
read -p "1. Введите имя проекта DDEV (например, grandmebel): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ ! -d "$DDEV_PROJECT_PATH" ]; then
  printf "\n%sОШИБКА: Директория проекта не найдена по пути:%s\n  %s\n" "${RED}" "${NC}" "$DDEV_PROJECT_PATH"; exit 1
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

read -p "2. Выберите протокол для продакшена (1=https, 2=http) [1]: " PROTOCOL_CHOICE
PROTOCOL_CHOICE=${PROTOCOL_CHOICE:-1}
case "$PROTOCOL_CHOICE" in
    1) PROD_URL="https://${RAW_DOMAIN}" ;;
    2) PROD_URL="http://${RAW_DOMAIN}" ;;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac

LOCAL_URL="https://${PROJECT_NAME}.lo"
CURRENT_DATE=$(date +%Y-%m-%d)
OUTPUT_FILENAME="${PROJECT_NAME}_${CURRENT_DATE}_from_local.sql"
OUTPUT_FILE_PATH="${DB_PROD_PATH}/${OUTPUT_FILENAME}"

printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Проект DDEV:        ${GREEN}${PROJECT_NAME} (${CMS_TYPE})${NC}"
echo "Локальный URL (что меняем): ${YELLOW}${LOCAL_URL}${NC}"
echo "Продакшен URL (на что меняем): ${GREEN}${PROD_URL}${NC}"
echo "Готовый файл будет сохранен как: ${GREEN}${OUTPUT_FILE_PATH}${NC}"
echo "---------------------------"
read -p "Продолжить экспорт? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# =================================================================
#  🚀 Безопасный экспорт
# =================================================================

mkdir -p "$DB_PROD_PATH"

# Сохраняем текущую директорию, чтобы вернуться в нее в конце
ORIGINAL_DIR=$(pwd)
cd "$DDEV_PROJECT_PATH"

if [ "$CMS_TYPE" = "wordpress" ]; then
    # --- Путь для WordPress ---
    function cleanup {
      printf "\n%sШаг 3: Возвращение URL на локальные значения...%s\n" "${BLUE}" "${NC}"
      ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root" >/dev/null 2>&1
      printf "✔ Локальная база данных восстановлена.\n"
      cd "$ORIGINAL_DIR"
    }
    trap cleanup EXIT

    printf "\n%sШаг 1: Временная замена URL на продакшен...%s\n" "${BLUE}" "${NC}"
    ddev exec "wp search-replace '$LOCAL_URL' '$PROD_URL' --all-tables --allow-root"
    printf "✔ URL в локальной базе временно изменены.\n"

    printf "\n%sШаг 2: Экспорт подготовленной базы данных...%s\n" "${BLUE}" "${NC}"
    ddev export-db --gzip=false > "$OUTPUT_FILE_PATH"
    printf "✔ Экспорт успешно завершен.\n"
else
    # --- Путь для Joomla, OpenCart и других PHP проектов ---
    printf "\n%sШаг 1: Экспорт базы данных 'как есть'...%s\n" "${BLUE}" "${NC}"
    # Создаем временный файл для "грязного" дампа
    TEMP_SQL_FILE="/tmp/${PROJECT_NAME}_export_raw_$(date +%s).sql"
    ddev export-db --gzip=false > "$TEMP_SQL_FILE"
    printf "✔ База данных экспортирована во временный файл.\n"

    printf "\n%sШаг 2: Замена URL в SQL-файле...%s\n" "${BLUE}" "${NC}"
    # Копируем временный файл в конечный и сразу делаем замену
    sed "s|${LOCAL_URL}|${PROD_URL}|g" "$TEMP_SQL_FILE" > "$OUTPUT_FILE_PATH"
    rm "$TEMP_SQL_FILE" # Очищаем временный файл
    printf "✔ Замена URL выполнена, итоговый файл создан.\n"
fi

printf "\n%s🎉 Готово! 🎉%s\n" "${GREEN}" "${NC}"
printf "Дамп базы данных, готовый для продакшена, сохранен в:\n%s%s%s\n" "${YELLOW}" "${OUTPUT_FILE_PATH}" "${NC}"

# cleanup вызовется автоматически для WP, для остальных просто выходим
exit 0

```

```
ddev-export
```