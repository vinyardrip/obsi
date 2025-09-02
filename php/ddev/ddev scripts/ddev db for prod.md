
```bash
#!/usr/bin/env bash
# =================================================================
# Скрипт для подготовки локальной DDEV-базы данных к выгрузке на продакшен.
# ВЕРСИЯ 2.3: Исправлена логика выполнения путем перехода в директорию проекта.
#
# Что делает:
# 1. Спрашивает имя проекта, переходит в его директорию.
# 2. Временно меняет URL в локальной БД на продакшен-URL.
# 3. Экспортирует базу данных в готовый для продакшена файл.
# 4. Немедленно возвращает локальные URL обратно.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"

# --- Утилиты и цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО РАБОТЫ ---
printf "%s--- Подготовка БД для продакшена ---%s\n" "${BLUE}" "${NC}"

# 1. Запрос ключевой информации
read -p "1. Введите имя проекта DDEV (например, grandmebel): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

# Проверяем наличие директории проекта
DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ ! -d "$DDEV_PROJECT_PATH" ]; then
  printf "\n%sОШИБКА: Директория проекта не найдена по пути:%s\n  %s\n" "${RED}" "${NC}" "${DDEV_PROJECT_PATH}"; exit 1
fi

read -p "   Доменная зона продакшена (.by, .com, etc) [.by]: " PROD_TLD
PROD_TLD=${PROD_TLD:-"by"}
RAW_DOMAIN="${PROJECT_NAME}.${PROD_TLD//./}"

# 2. Проверка, запущен ли проект (можно делать извне)
printf "Проверяем статус проекта '%s'...\n" "$PROJECT_NAME"
if ! ddev describe "$PROJECT_NAME" | grep -q "OK"; then
    printf "\n%sВНИМАНИЕ: Проект '%s' не запущен.%s\n" "${YELLOW}" "$PROJECT_NAME" "${NC}"
    read -p "Запустить его сейчас? (y/n): " CONFIRM_START
    if [ "$CONFIRM_START" = "y" ]; then
        ddev start -p "$PROJECT_NAME"
    else
        printf "%sОперация отменена.%s\n" "${RED}" "${NC}"; exit 1
    fi
fi
printf "✔ Проект запущен и готов к работе.\n"

read -p "2. Выберите протокол для продакшена (1=https, 2=http) [1]: " PROTOCOL_CHOICE
PROTOCOL_CHOICE=${PROTOCOL_CHOICE:-1}
case "$PROTOCOL_CHOICE" in
    1) PROD_URL="https://${RAW_DOMAIN}" ;;
    2) PROD_URL="http://${RAW_DOMAIN}" ;;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac

# 3. Формируем переменные и пути
LOCAL_URL="https://${PROJECT_NAME}.lo"
CURRENT_DATE=$(date +%Y-%m-%d)
OUTPUT_FILENAME="${PROJECT_NAME}_${CURRENT_DATE}_from_local.sql"
OUTPUT_FILE_PATH="${DB_PROD_PATH}/${OUTPUT_FILENAME}"

# --- Сводка и подтверждение ---
printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Проект DDEV:        ${GREEN}${PROJECT_NAME}${NC}"
echo "Директория проекта: ${GREEN}${DDEV_PROJECT_PATH}${NC}"
echo "Локальный URL (что меняем): ${YELLOW}${LOCAL_URL}${NC}"
echo "Продакшен URL (на что меняем): ${GREEN}${PROD_URL}${NC}"
echo "Готовый файл будет сохранен как:"
echo "${GREEN}${OUTPUT_FILE_PATH}${NC}"
printf "\n%sБудет использован безопасный метод 'wp search-replace'.%s\n" "${GREEN}" "${NC}"
echo "---------------------------"
read -p "Продолжить экспорт? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# =================================================================
#  🚀 Безопасный экспорт
# =================================================================

# Сохраняем текущую директорию, чтобы вернуться в нее в конце
ORIGINAL_DIR=$(pwd)
# ПЕРЕХОДИМ В ДИРЕКТОРИЮ ПРОЕКТА - ЭТО КЛЮЧЕВОЕ ИЗМЕНЕНИЕ
cd "$DDEV_PROJECT_PATH"

# Функция для возврата URL, которая будет вызвана при любом завершении скрипта
function cleanup {
  printf "\n%sШаг 3: Возвращение URL на локальные значения...%s\n" "${BLUE}" "${NC}"
  # Теперь, когда мы в нужной директории, команда простая
  ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root" >/dev/null 2>&1
  printf "✔ Локальная база данных восстановлена.\n"
  # Возвращаемся в исходную директорию
  cd "$ORIGINAL_DIR"
}

# Перехватываем выход из скрипта и вызываем cleanup
trap cleanup EXIT

printf "\n%sШаг 1: Временная замена URL на продакшен...%s\n" "${BLUE}" "${NC}"
# Теперь, когда мы в нужной директории, команда простая
ddev exec "wp search-replace '$LOCAL_URL' '$PROD_URL' --all-tables --allow-root"
printf "✔ URL в локальной базе временно изменены.\n"

printf "\n%sШаг 2: Экспорт подготовленной базы данных...%s\n" "${BLUE}" "${NC}"
# Указываем абсолютный путь для сохранения файла
ddev export-db --gzip=false > "$OUTPUT_FILE_PATH"
printf "✔ Экспорт успешно завершен.\n"

printf "\n%s🎉 Готово! 🎉%s\n" "${GREEN}" "${NC}"
printf "Дамп базы данных, готовый для продакшена, сохранен в:\n%s%s%s\n" "${YELLOW}" "${OUTPUT_FILE_PATH}" "${NC}"

# Функция cleanup будет вызвана автоматически при выходе
exit 0