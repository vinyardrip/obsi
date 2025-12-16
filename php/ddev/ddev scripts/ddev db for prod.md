---
tags:
  - ddev
  - script
  - database
  - bash
  - devops
date: 2023-10-25
---

# 🚀 Скрипт: Экспорт DDEV БД на Продакшен (v3.1)

Этот скрипт автоматически готовит базу данных к выгрузке на живой сайт.
**Ключевая особенность:**
1. Для **WordPress** использует `wp search-replace` (сохраняет сериализацию) + делает **авто-снапшот** перед стартом (страховка).
2. Для **OpenCart/Joomla/PHP** делает замену через `sed` в SQL-файле.
3. Останавливает **Devilbox**, чтобы не было конфликта портов.

### Код скрипта (`ddev-export.sh`)

```bash
#!/usr/bin/env bash
# =================================================================
# УНИВЕРСАЛЬНЫЙ скрипт для подготовки локальной DDEV-БД к выгрузке на продакшен.
# ВЕРСИЯ 3.1: + Страховочный Snapshot перед заменой URL
#
# Что делает:
# - Останавливает Devilbox (если есть).
# - Делает мгновенный снимок БД (для WP) на случай сбоя.
# - Меняет локальные домены (.lo) на продакшен (.by/.com) правильным методом.
# - Экспортирует .sql файл.
# - Возвращает локальную базу в исходное состояние.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"

# --- Цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО РАБОТЫ ---
printf "%s--- Подготовка БД для продакшена ---%s\n" "${BLUE}" "${NC}"

# === БЛОК 0: Предотвращение конфликта портов ===
if [ -d "$DEVILBOX_PATH_ROOT" ]; then
    printf "\n%s--- Проверка состояния Devilbox ---%s\n" "${BLUE}" "${NC}"
    printf "Чтобы избежать конфликта портов, Devilbox будет остановлен...\n"
    (cd "$DEVILBOX_PATH_ROOT" && docker-compose down >/dev/null 2>&1)
    printf "✔ Devilbox успешно остановлен.\n"
fi

# === ШАГ 1: ВЫБОР ТИПА ПРОЕКТА ===
printf "\n%s--- Выбор типа проекта ---%s\n" "${BLUE}" "${NC}"
echo "1. WordPress (безопасная замена URL через WP-CLI + Snapshot)"
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

# 2. Запрос ключевой информации
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
echo "Локальный URL:      ${YELLOW}${LOCAL_URL}${NC}"
echo "Продакшен URL:      ${GREEN}${PROD_URL}${NC}"
echo "Файл сохранения:    ${GREEN}${OUTPUT_FILE_PATH}${NC}"
echo "---------------------------"
read -p "Продолжить экспорт? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# =================================================================
#  🚀 ЭКСПОРТ
# =================================================================

mkdir -p "$DB_PROD_PATH"

# Сохраняем текущую директорию
ORIGINAL_DIR=$(pwd)
cd "$DDEV_PROJECT_PATH"

if [ "$CMS_TYPE" = "wordpress" ]; then
    # --- WORDPRESS (WP-CLI + SNAPSHOT) ---
    
    # Функция очистки (вызывается при выходе)
    function cleanup {
      printf "\n%sШаг 3: Возвращение URL на локальные значения...%s\n" "${BLUE}" "${NC}"
      ddev exec "wp search-replace '$PROD_URL' '$LOCAL_URL' --all-tables --allow-root" >/dev/null 2>&1
      printf "✔ Локальная база данных восстановлена.\n"
      cd "$ORIGINAL_DIR"
    }
    trap cleanup EXIT

    # 0. СТРАХОВКА (Snapshot)
    printf "\n%sШаг 0: Создание страховочного снимка (Snapshot)...%s\n" "${BLUE}" "${NC}"
    ddev delete-snapshot auto_pre_export >/dev/null 2>&1 || true
    ddev snapshot --name "auto_pre_export"
    printf "✔ Снимок 'auto_pre_export' создан (на случай аварии).\n"

    # 1. Замена
    printf "\n%sШаг 1: Замена URL на продакшен (search-replace)...%s\n" "${BLUE}" "${NC}"
    ddev exec "wp search-replace '$LOCAL_URL' '$PROD_URL' --all-tables --allow-root"
    printf "✔ URL временно изменены.\n"

    # 2. Экспорт
    printf "\n%sШаг 2: Экспорт SQL файла...%s\n" "${BLUE}" "${NC}"
    ddev export-db --gzip=false > "$OUTPUT_FILE_PATH"
    printf "✔ Экспорт завершен.\n"

else
    # --- ДРУГИЕ CMS (SED / SQL TEXT REPLACE) ---
    
    printf "\n%sШаг 1: Экспорт базы данных 'как есть'...%s\n" "${BLUE}" "${NC}"
    # Временный файл
    TEMP_SQL_FILE="/tmp/${PROJECT_NAME}_export_raw_$(date +%s).sql"
    ddev export-db --gzip=false > "$TEMP_SQL_FILE"
    printf "✔ Экспорт во временный файл.\n"

    printf "\n%sШаг 2: Замена URL в SQL-файле (sed)...%s\n" "${BLUE}" "${NC}"
    # Подмена текста
    sed "s|${LOCAL_URL}|${PROD_URL}|g" "$TEMP_SQL_FILE" > "$OUTPUT_FILE_PATH"
    rm "$TEMP_SQL_FILE"
    printf "✔ Замена URL выполнена.\n"
fi

printf "\n%s🎉 Готово! Файл сохранен: %s%s\n" "${GREEN}" "${OUTPUT_FILE_PATH}" "${NC}"

exit 0