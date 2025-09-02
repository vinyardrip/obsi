
```bash
#!/usr/bin/env bash
# =================================================================
# Централизованный скрипт для развертывания PHP-проекта на DDEV.
# ВЕРСИЯ 6.0: "Умные имена" - минимум вопросов, максимум автоматизации.
# - Автоматически определяет имя проекта и домен.
# - Автоматически генерирует "умный" wp-config.php.
# - Автоматически очищает старые данные проекта.
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
printf "%s--- Развертывание нового проекта на DDEV ---%s\n" "${BLUE}" "${NC}"

# 1. Задаем всего один главный вопрос
read -p "1. Введите основное имя проекта/домена (например, grandmebel): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

read -p "   Доменная зона продакшена (.by, .com, etc) [.by]: " PROD_TLD
PROD_TLD=${PROD_TLD:-"by"} # .by по умолчанию
RAW_DOMAIN="${PROJECT_NAME}.${PROD_TLD//./}" # Убираем точку, если пользователь ее ввел

# Проверка и очистка старых данных DDEV
printf "\n%sПроверяем наличие старых данных для проекта '%s'...%s\n" "${BLUE}" "${PROJECT_NAME}" "${NC}"
if ddev describe "$PROJECT_NAME" >/dev/null 2>&1; then
    printf "\n%sВНИМАНИЕ: DDEV уже содержит данные для проекта '%s'.%s\n" "${YELLOW}" "$PROJECT_NAME" "${NC}"
    read -p "Удалить все существующие данные DDEV и начать заново? (y/n): " CONFIRM_DELETE
    if [ "$CONFIRM_DELETE" = "y" ]; then
        ddev delete "$PROJECT_NAME" -y
        printf "✔ Старые данные DDEV успешно удалены.\n\n"
    else
        printf "%sРазвертывание отменено.%s\n" "${RED}" "${NC}"; exit 1
    fi
else
    printf "✔ Старых данных не найдено, можно продолжать.\n\n"
fi

# 2. Формирование и проверка пути к проекту
DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ ! -d "$DDEV_PROJECT_PATH" ]; then
  printf "\n%sОШИБКА: Директория проекта не найдена по пути:%s\n  %s\n" "${RED}" "${NC}" "${DDEV_PROJECT_PATH}"; exit 1
fi

# 3. Запрос остальной информации
read -p "2. Имя файла БД из '${DB_PROD_PATH}' (без .sql) [${PROJECT_NAME}]: " DB_FILENAME_BASE
DB_FILENAME_BASE=${DB_FILENAME_BASE:-${PROJECT_NAME}} # Используем имя проекта по умолчанию
SOURCE_DB_FILE="${DB_PROD_PATH}/${DB_FILENAME_BASE}.sql"
if [ ! -f "$SOURCE_DB_FILE" ]; then
  printf "\n%sОШИБКА: Файл базы данных не найден:%s %s\n" "${RED}" "${NC}" "$SOURCE_DB_FILE"; exit 1;
fi

read -p "3. Выберите протокол для продакшена (1=https, 2=http) [1]: " PROTOCOL_CHOICE
PROTOCOL_CHOICE=${PROTOCOL_CHOICE:-1}
case "$PROTOCOL_CHOICE" in
    1) OLD_DOMAIN="https://${RAW_DOMAIN}" ;;
    2) OLD_DOMAIN="http://${RAW_DOMAIN}" ;;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac

# 4. Настройки проекта DDEV
read -p "4. Тип проекта DDEV [wordpress]: " PROJECT_TYPE; PROJECT_TYPE=${PROJECT_TYPE:-wordpress}
read -p "5. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION; DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}
read -p "6. Тип БД (mysql, mariadb) [mysql]: " DB_TYPE; DB_TYPE=${DB_TYPE:-mysql}
read -p "7. Версия БД (например, 5.7) [5.7]: " DB_VERSION; DB_VERSION=${DB_VERSION:-5.7}

# --- Сводка ---
NEW_URL="https://${PROJECT_NAME}.lo"
printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Имя проекта:      ${GREEN}${PROJECT_NAME}${NC}"; echo "Путь к проекту:   ${GREEN}${DDEV_PROJECT_PATH}${NC}"
echo "Тип проекта:      ${GREEN}${PROJECT_TYPE}${NC}"; echo "Версия PHP:       ${GREEN}${DDEV_PHP_VERSION}${NC}"
echo "База данных:      ${GREEN}${DB_TYPE}:${DB_VERSION}${NC}"; echo "Файл БД:          ${GREEN}${SOURCE_DB_FILE}${NC}"
echo "Prod URL:         ${YELLOW}${OLD_DOMAIN}${NC}"; echo "Local URL:        ${GREEN}${NEW_URL}${NC}"
echo "---------------------------"
read -p "Все верно? Начинаем? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# =================================================================
#  🚀 Шаги автоматизации
# =================================================================

cd "$DDEV_PROJECT_PATH"

printf "\n%sШаг 1: Конфигурация DDEV...%s\n" "${BLUE}" "${NC}"
ddev config --project-name "$PROJECT_NAME" --project-type "$PROJECT_TYPE" \
  --php-version "$DDEV_PHP_VERSION" --database "${DB_TYPE}:${DB_VERSION}" \
  --docroot "" --webserver-type apache-fpm --project-tld "lo"
printf "✔ Конфигурация создана.\n"

printf "\n%sШаг 2: Запуск проекта...%s\n" "${BLUE}" "${NC}"
ddev start -y
printf "✔ Проект успешно запущен.\n"

if [ "$PROJECT_TYPE" = "wordpress" ] && [ -f "wp-config.php" ]; then
    printf "\n%sШаг 3: Автоматическая настройка wp-config.php...%s\n" "${BLUE}" "${NC}"
    mv wp-config.php wp-config.original.php
    printf "✔ Оригинальный wp-config.php сохранен как wp-config.original.php\n"
    DB_CREDS=$(grep -E "define\(\s*'DB_(NAME|USER|PASSWORD|HOST)'," wp-config.original.php)
    DB_PREFIX=$(grep '$table_prefix' wp-config.original.php)
    AUTH_KEYS=$(grep -A 8 "AUTH_KEY" wp-config.original.php)
    cat << EOF > wp-config.php
<?php
/** This file was automatically generated by a DDEV deployment script. */
if (getenv('IS_DDEV_PROJECT') == 'true') {
    define('DB_NAME', 'db');
    define('DB_USER', 'db');
    define('DB_PASSWORD', 'db');
    define('DB_HOST', 'db');
    define('DB_CHARSET', 'utf8mb4');
    define('DB_COLLATE', '');
    define('WP_HOME', '${NEW_URL}');
    define('WP_SITEURL', '${NEW_URL}');
    if (empty(\$_SERVER['HTTPS'])) { \$_SERVER['HTTPS'] = 'on'; }
} else {
    /** Production Environment Configuration */
    ${DB_CREDS}
    define('DB_CHARSET', 'utf8mb4');
    define('DB_COLLATE', '');
    define('WP_HOME', '${OLD_DOMAIN}');
    define('WP_SITEURL', '${OLD_DOMAIN}');
}
/** Authentication Unique Keys and Salts. */
${AUTH_KEYS}
/** WordPress Database Table prefix. */
${DB_PREFIX}
define('WP_DEBUG', false);
/* That's all, stop editing! Happy publishing. */
if ( ! defined('ABSPATH') ) { define('ABSPATH', __DIR__ . '/'); }
require_once ABSPATH . 'wp-settings.php';
EOF
    printf "✔ Новый 'умный' wp-config.php успешно создан.\n"
fi

printf "\n%sШаг 4: Импорт базы данных...%s\n" "${BLUE}" "${NC}"
ddev import-db --file="$SOURCE_DB_FILE"
printf "✔ База данных импортирована.\n"

if [ "$PROJECT_TYPE" = "wordpress" ]; then
    printf "\n%sШаг 5: Замена URL в базе данных...%s\n" "${BLUE}" "${NC}"
    ddev wp search-replace "$OLD_DOMAIN" "$NEW_URL" --all-tables --allow-root
    printf "✔ URL успешно заменены.\n"
fi

printf "\n%s🎉 Проект успешно развернут! 🎉%s\n" "${GREEN}" "${NC}"
printf "Сайт доступен по адресу: %s%s%s\n" "${YELLOW}" "${NEW_URL}" "${NC}"
ddev launch

exit 0 # Корректное завершение скрипта