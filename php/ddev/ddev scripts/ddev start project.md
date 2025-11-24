
```bash
#!/usr/bin/env bash
# =================================================================
# УНИВЕРСАЛЬНЫЙ скрипт для развертывания существующего проекта на DDEV.
# ВЕРСИЯ 2.1: Добавлена автоматическая остановка Devilbox.
#
# Что делает:
# 1. Останавливает Devilbox для предотвращения конфликта портов.
# 2. Спрашивает тип и имя проекта.
# 3. Настраивает DDEV.
# 4. Импортирует базу данных с прода, выполняя необходимые для CMS замены.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DB_PROD_PATH="$HOME/projects/work/db_prod"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
# Добавляем путь к Devilbox
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"

# --- Утилиты и цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 🚀 НАЧАЛО РАБОТЫ ---
printf "%s--- Развертывание существующего проекта на DDEV ---%s\n" "${BLUE}" "${NC}"

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
echo "1. WordPress"
echo "2. Joomla"
echo "3. OpenCart"
echo "4. Чистый PHP/HTML (с базой данных)"
read -p "Выберите тип проекта [1]: " PROJECT_CHOICE
PROJECT_CHOICE=${PROJECT_CHOICE:-1}

case "$PROJECT_CHOICE" in
    1) DDEV_PROJECT_TYPE="wordpress"; CMS_TYPE="wordpress";;
    2) DDEV_PROJECT_TYPE="php";       CMS_TYPE="joomla";;
    3) DDEV_PROJECT_TYPE="php";       CMS_TYPE="opencart";;
    4) DDEV_PROJECT_TYPE="php";       CMS_TYPE="php";;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac
printf "✔ Выбран тип CMS: %s (для DDEV будет использован тип: %s)\n" "$CMS_TYPE" "$DDEV_PROJECT_TYPE"

# --- ОБЩИЕ НАСТРОЙКИ ПРОЕКТА ---
read -p "1. Введите имя папки проекта (например, my-joomla-site): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

DDEV_PROJECT_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ ! -d "$DDEV_PROJECT_PATH" ]; then
  printf "\n%sОШИБКА: Директория проекта не найдена по пути:%s\n  %s\n" "${RED}" "${NC}" "${DDEV_PROJECT_PATH}"; exit 1
fi

if ddev describe "$PROJECT_NAME" >/dev/null 2>&1; then
    printf "\n%sВНИМАНИЕ: DDEV уже содержит данные для проекта '%s'.%s\n" "${YELLOW}" "$PROJECT_NAME" "${NC}"
    read -p "Удалить все существующие данные DDEV и начать заново? (y/n): " CONFIRM_DELETE
    if [ "$CONFIRM_DELETE" = "y" ]; then
        ddev delete "$PROJECT_NAME" -y; printf "✔ Старые данные DDEV успешно удалены.\n\n"
    else
        printf "%sРазвертывание отменено.%s\n" "${RED}" "${NC}"; exit 1
    fi
fi

read -p "2. Имя файла БД из '${DB_PROD_PATH}' (без .sql): " DB_FILENAME_BASE
if [ -z "$DB_FILENAME_BASE" ]; then printf "%sОШИБКА: Имя файла БД не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi
SOURCE_DB_FILE="${DB_PROD_PATH}/${DB_FILENAME_BASE}.sql"
if [ ! -f "$SOURCE_DB_FILE" ]; then
  printf "\n%sОШИБКА: Файл базы данных не найден:%s %s\n" "${RED}" "${NC}" "$SOURCE_DB_FILE"; exit 1;
fi

read -p "3. Домен продакшен-сайта (например, my-joomla-site.by): " RAW_DOMAIN
if [ -z "$RAW_DOMAIN" ]; then printf "%sОШИБКА: Домен не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi
read -p "   Выберите протокол для него (1=https, 2=http) [1]: " PROTOCOL_CHOICE
PROTOCOL_CHOICE=${PROTOCOL_CHOICE:-1}
case "$PROTOCOL_CHOICE" in
    1) OLD_DOMAIN="https://${RAW_DOMAIN}" ;;
    2) OLD_DOMAIN="http://${RAW_DOMAIN}" ;;
    *) printf "%sОШИБКА: Неверный выбор.%s\n" "${RED}" "${NC}"; exit 1 ;;
esac

read -p "4. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION; DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}
read -p "5. Тип БД (mysql, mariadb) [mysql]: " DB_TYPE; DB_TYPE=${DB_TYPE:-mysql}
read -p "6. Версия БД (например, 5.7) [5.7]: " DB_VERSION; DB_VERSION=${DB_VERSION:-"5.7"}

NEW_URL="https://${PROJECT_NAME}.lo"

printf "\n%s--- Проверьте параметры ---%s\n" "${BLUE}" "${NC}"
echo "Имя проекта:      ${GREEN}${PROJECT_NAME}${NC}"
echo "Старый URL:       ${YELLOW}${OLD_DOMAIN}${NC}"
echo "Новый URL:        ${GREEN}${NEW_URL}${NC}"
# ... (можно добавить вывод других параметров)
read -p "Все верно? Начинаем? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Операция отменена.\n"; exit 1; fi

# =================================================================
#  🚀 Шаги автоматизации
# =================================================================

cd "$DDEV_PROJECT_PATH"

printf "\n%sШаг 1: Конфигурация DDEV...%s\n" "${BLUE}" "${NC}"
ddev config --project-name "$PROJECT_NAME" --project-type "$DDEV_PROJECT_TYPE" \
  --php-version "$DDEV_PHP_VERSION" --database "${DB_TYPE}:${DB_VERSION}" \
  --docroot "" --webserver-type apache-fpm --project-tld "lo"
printf "✔ Конфигурация создана.\n"

printf "\n%sШаг 2: Запуск проекта...%s\n" "${BLUE}" "${NC}"
ddev start -y
printf "✔ Проект успешно запущен.\n"

if [ "$CMS_TYPE" = "wordpress" ] && [ -f "wp-config.php" ]; then
    printf "\n%sШаг 3: Автоматическая настройка wp-config.php...%s\n" "${BLUE}" "${NC}"
    mv wp-config.php wp-config.original.php
    DB_CREDS=$(grep -E "define\(\s*'DB_(NAME|USER|PASSWORD|HOST)'," wp-config.original.php)
    DB_PREFIX=$(grep '$table_prefix' wp-config.original.php)
    AUTH_KEYS=$(grep -A 8 "AUTH_KEY" wp-config.original.php)
    cat << EOF > wp-config.php
<?php
/** This file was automatically generated by a DDEV deployment script. */
if (getenv('IS_DDEV_PROJECT') == 'true') {
    define('DB_NAME', 'db'); define('DB_USER', 'db'); define('DB_PASSWORD', 'db'); define('DB_HOST', 'db');
    define('DB_CHARSET', 'utf8mb4'); define('DB_COLLATE', '');
    define('WP_HOME', '${NEW_URL}'); define('WP_SITEURL', '${NEW_URL}');
    if (empty(\$_SERVER['HTTPS'])) { \$_SERVER['HTTPS'] = 'on'; }
} else { /* Production/Original Settings */ ${DB_CREDS}; define('DB_CHARSET', 'utf8mb4'); define('DB_COLLATE', ''); }
${AUTH_KEYS}; ${DB_PREFIX}; define('WP_DEBUG', false);
if ( ! defined('ABSPATH') ) { define('ABSPATH', __DIR__ . '/'); }
require_once ABSPATH . 'wp-settings.php';
EOF
    printf "✔ Новый 'умный' wp-config.php успешно создан.\n"

elif [ "$CMS_TYPE" = "joomla" ] && [ -f "configuration.php" ]; then
    printf "\n%sШаг 3: Автоматическая настройка configuration.php...%s\n" "${BLUE}" "${NC}"
    sed -i.bak -e "s|public \$host = .*|public \$host = 'db';|" \
               -e "s|public \$user = .*|public \$user = 'db';|" \
               -e "s|public \$password = .*|public \$password = 'db';|" \
               -e "s|public \$db = .*|public \$db = 'db';|" \
               -e "s|public \$live_site = .*|public \$live_site = '${NEW_URL}';|" \
               "configuration.php"
    printf "✔ Файл configuration.php обновлен для DDEV.\n"
elif [ "$CMS_TYPE" = "opencart" ]; then
    printf "\n%sШаг 3: Автоматическая настройка config.php...%s\n" "${BLUE}" "${NC}"
    if [ -f "config.php" ]; then
        sed -i.bak -e "s/'DB_HOSTNAME', '.*'/'DB_HOSTNAME', 'db'/" \
                   -e "s/'DB_USERNAME', '.*'/'DB_USERNAME', 'db'/" \
                   -e "s/'DB_PASSWORD', '.*'/'DB_PASSWORD', 'db'/" \
                   -e "s/'DB_DATABASE', '.*'/'DB_DATABASE', 'db'/" "config.php"
        printf "✔ Корневой config.php обновлен.\n"
    fi
    if [ -f "admin/config.php" ]; then
        sed -i.bak -e "s/'DB_HOSTNAME', '.*'/'DB_HOSTNAME', 'db'/" \
                   -e "s/'DB_USERNAME', '.*'/'DB_USERNAME', 'db'/" \
                   -e "s/'DB_PASSWORD', '.*'/'DB_PASSWORD', 'db'/" \
                   -e "s/'DB_DATABASE', '.*'/'DB_DATABASE', 'db'/" "admin/config.php"
        printf "✔ Файл admin/config.php обновлен.\n"
    fi
fi

if [ "$CMS_TYPE" = "wordpress" ]; then
    printf "\n%sШаг 4: Импорт базы данных...%s\n" "${BLUE}" "${NC}"
    ddev import-db --file="$SOURCE_DB_FILE"
    printf "✔ База данных импортирована.\n"
    printf "\n%sШаг 5: Замена URL в базе данных (WP-CLI)...%s\n" "${BLUE}" "${NC}"
    ddev wp search-replace "$OLD_DOMAIN" "$NEW_URL" --all-tables --allow-root
    printf "✔ URL успешно заменены.\n"
else
    printf "\n%sШаг 4: Подготовка и замена URL в SQL-файле...%s\n" "${BLUE}" "${NC}"
    TEMP_SQL_FILE="/tmp/${PROJECT_NAME}_deploy_$(date +%s).sql"
    cp "$SOURCE_DB_FILE" "$TEMP_SQL_FILE"
    sed -i "s|${OLD_DOMAIN}|${NEW_URL}|g" "$TEMP_SQL_FILE"
    printf "✔ Временный SQL-файл с замененными URL подготовлен.\n"
    printf "\n%sШаг 5: Импорт подготовленной базы данных...%s\n" "${BLUE}" "${NC}"
    ddev import-db --file="$TEMP_SQL_FILE"
    rm "$TEMP_SQL_FILE"
    printf "✔ База данных импортирована и обновлена.\n"
fi

printf "\n%s🎉 Проект успешно развернут! 🎉%s\n" "${GREEN}" "${NC}"
printf "Сайт доступен по адресу: %s%s%s\n" "${YELLOW}" "${NEW_URL}" "${NC}"
ddev launch

exit 0

```

```
ddev-deploy
```
