```bash
#!/usr/bin/env bash
# =================================================================
# Интерактивный скрипт для миграции PHP-проекта с Devilbox на DDEV
# ВЕРСИЯ 21.4 («Надежная санитизация»)
#
# Что нового:
# - УЛУЧШЕНИЕ: Добавлена полноценная санитизация (очистка)
#   вводимого имени БД. Скрипт теперь удаляет спецсимволы
#   и заменяет пробелы, создавая безопасное имя для БД.
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"
DEVILBOX_DATA_BASE_PATH="${DEVILBOX_PATH_ROOT}/data"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
DB_BACKUP_PATH="$HOME/projects/work/db_local"

# --- Утилиты и цвета ---
if command -v tput >/dev/null 2>&1; then
    GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3); BLUE=$(tput setaf 4); RED=$(tput setaf 1); NC=$(tput sgr0)
else
    GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'
fi

# --- 📝 ИНТЕРАКТИВНАЯ НАСТРОЙКА ---
printf "%s--- Настройка миграции проекта ---%s\n" "${BLUE}" "${NC}"

read -p "1. Имя проекта для DDEV (например, grandmebel): " PROJECT_NAME
if [ -z "$PROJECT_NAME" ]; then printf "%sОШИБКА: Имя проекта не может быть пустым.%s\n" "${RED}" "${NC}"; exit 1; fi

DDEV_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
if [ -d "$DDEV_PATH" ]; then
  printf "\n%sВНИМАНИЕ: Директория проекта '%s' уже существует.%s\n" "${YELLOW}" "${DDEV_PATH}" "${NC}"
  read -p "Удалить её и ВСЕ данные DDEV для этого проекта, чтобы начать заново? (y/n): " CONFIRM_DELETE
  if [ "$CONFIRM_DELETE" = "y" ]; then
    printf "Полное удаление проекта '%s' из DDEV...\n" "$PROJECT_NAME"
    (cd "$DDEV_PATH" && ddev delete -y) 2>/dev/null || ddev delete "$PROJECT_NAME" -y 2>/dev/null || true
    rm -rf "$DDEV_PATH"
    printf "✔ Проект и его данные полностью удалены.\n\n"
  else
    printf "%sМиграция отменена.%s\n" "${RED}" "${NC}"; exit 1;
  fi
fi

read -p "2. Имя папки проекта в Devilbox/data [${PROJECT_NAME}]: " DEVILBOX_PROJECT_FOLDER
DEVILBOX_PROJECT_FOLDER=${DEVILBOX_PROJECT_FOLDER:-${PROJECT_NAME}}
DEVILBOX_PATH_PROJECT="${DEVILBOX_DATA_BASE_PATH}/${DEVILBOX_PROJECT_FOLDER}"
if [ ! -d "$DEVILBOX_PATH_PROJECT" ]; then
  printf "\n%sОШИБКА: Исходная директория не найдена:%s %s\n" "${RED}" "${NC}" "$DEVILBOX_PATH_PROJECT"; exit 1;
fi

read -p "3. Какую версию PHP Devilbox запустить для экспорта? (6/7/8): " DEVILBOX_PHP_VERSION
if [[ ! "$DEVILBOX_PHP_VERSION" =~ ^[6-8]$ ]]; then
  printf "%sОШИБКА: Допустимы только 6, 7 или 8.%s\n" "${RED}" "${NC}"; exit 1;
fi

read -p "4. Тип проекта DDEV [wordpress]: " PROJECT_TYPE; PROJECT_TYPE=${PROJECT_TYPE:-wordpress}

TABLE_PREFIX=""
if [ "$PROJECT_TYPE" = "wordpress" ]; then
    read -p "4a. Префикс таблиц WordPress в БД [wp_]: " TABLE_PREFIX; TABLE_PREFIX=${TABLE_PREFIX:-wp_}
fi

read -p "5. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION; DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}

printf "%s--- Конфигурация Базы Данных (как в Devilbox) ---%s\n" "${BLUE}" "${NC}"
read -p "6. Тип БД (mysql, mariadb) [mysql]: " DB_TYPE_DDEV; DB_TYPE_DDEV=${DB_TYPE_DDEV:-mysql}
read -p "7. Версия БД (например, 5.7) [5.7]: " DB_VERSION_DDEV; DB_VERSION_DDEV=${DB_VERSION_DDEV:-5.7}

# --- АВТОМАТИЧЕСКИЕ УЧЕТНЫЕ ДАННЫЕ С САНИТИЗАЦИЕЙ ---
printf "%s--- Учетные данные БД в Devilbox ---%s\n" "${BLUE}" "${NC}"
read -p "Имя БД в Devilbox [${PROJECT_NAME}]: " RAW_DB_NAME

# Если ввод пустой, используем имя проекта как значение по умолчанию
TEMP_DB_NAME=${RAW_DB_NAME:-${PROJECT_NAME}}

# Обрабатываем ввод:
# 1. Заменяем пробелы и дефисы на знак подчеркивания '_'.
# 2. Удаляем ВСЕ символы, кроме букв (a-z, A-Z), цифр (0-9) и знака подчеркивания.
DB_NAME_FOR_EXPORT=$(echo "$TEMP_DB_NAME" | sed -e 's/[[:space:]]\+/-/g' -e 's/[^a-zA-Z0-9_-]//g' -e 's/-/ /g' | awk '{print $1}' | sed 's/_*$//')

# Если мы что-то изменили, вежливо сообщаем пользователю
if [ "$TEMP_DB_NAME" != "$DB_NAME_FOR_EXPORT" ]; then
    printf "✔ Введено: '%s'. После очистки будет использовано имя: %s%s%s\n" "$TEMP_DB_NAME" "${GREEN}" "$DB_NAME_FOR_EXPORT" "${NC}"
fi

DB_USER_FOR_EXPORT="root"
DB_PASS_FOR_EXPORT=""
printf "✔ Используются стандартные учетные данные: %suser: %s, password: (пустой)%s\n" "${GREEN}" "${DB_USER_FOR_EXPORT}" "${NC}"

BACKUP_FILE_TO_CREATE="${DB_BACKUP_PATH}/db_${PROJECT_NAME}.sql"
NEW_URL="https://${PROJECT_NAME}.lo"
OLD_URL_TO_REPLACE="https://${DEVILBOX_PROJECT_FOLDER}.loc"

read -p "Продолжить миграцию? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then printf "Миграция отменена.\n"; exit 1; fi

# =================================================================
#  Шаги автоматизации
# =================================================================

cd "$DEVILBOX_PATH_ROOT"; ddev poweroff
[ -f ".env" ] && mv .env env.bak; mv ".env-${DEVILBOX_PHP_VERSION}" .env
docker-compose up -d httpd php mysql mailhog; sleep 15
mkdir -p "$DB_BACKUP_PATH"
docker-compose exec -T -e MYSQL_PWD="$DB_PASS_FOR_EXPORT" mysql mysqldump -u "$DB_USER_FOR_EXPORT" "$DB_NAME_FOR_EXPORT" > "$BACKUP_FILE_TO_CREATE"
docker-compose down
mv .env ".env-${DEVILBOX_PHP_VERSION}"
[ -f "env.bak" ] && mv env.bak .env

# --- НОВЫЙ БЛОК: Проверка и освобождение сетевых портов ---
printf "\n%s--- Проверка и освобождение сетевых портов ---%s\n" "${BLUE}" "${NC}"
# Проверяем, активен ли Apache2, и если да, останавливаем его
if systemctl is-active --quiet apache2; then
  printf "%sОбнаружен активный сервис Apache2. Требуются права администратора для его остановки.%s\n" "${YELLOW}" "${NC}"
  sudo systemctl stop apache2
  printf "✔ Сервис Apache2 успешно остановлен.\n"
fi
# Проверяем, активен ли Nginx, и если да, останавливаем его
if systemctl is-active --quiet nginx; then
  printf "%sОбнаружен активный сервис Nginx. Требуются права администратора для его остановки.%s\n" "${YELLOW}" "${NC}"
  sudo systemctl stop nginx
  printf "✔ Сервис Nginx успешно остановлен.\n"
fi
printf "✔ Проверка портов завершена. Порты 80/443 должны быть свободны.\n\n"
# --- КОНЕЦ НОВОГО БЛОКА ---

mkdir -p "$DDEV_PROJECTS_BASE_PATH"
cp -R "$DEVILBOX_PATH_PROJECT" "$DDEV_PATH"
cd "$DDEV_PATH"

ddev config --project-name "$PROJECT_NAME" --project-type "$PROJECT_TYPE" \
  --php-version "$DDEV_PHP_VERSION" --database "$DB_TYPE_DDEV:$DB_VERSION_DDEV" \
  --docroot "" --webserver-type apache-fpm --project-tld "lo"

if [ "$PROJECT_TYPE" = "wordpress" ]; then
    mv wp-config.php wp-config-original.php
    cp wp-config-ddev.php wp-config.php
fi

if [ -d ".git" ]; then
  git add .ddev; if [ "$PROJECT_TYPE" = "wordpress" ]; then git add wp-config.php; fi
  if ! git diff --cached --quiet; then git commit -m "chore: Configure DDEV"; fi
else
  curl -fsSL --retry 3 -o .gitignore https://raw.githubusercontent.com/github/gitignore/main/WordPress.gitignore
  git init -b main; git add .; git commit -m "Initial commit with DDEV";
fi

printf "%sШаг 4: Запуск DDEV...%s\n" "${BLUE}" "${NC}"
ddev start -y

printf "%sШаг 5: Импорт бэкапа...%s\n" "${BLUE}" "${NC}"
ddev import-db --file="$BACKUP_FILE_TO_CREATE"

if [ "$PROJECT_TYPE" = "wordpress" ]; then
    printf "%sШаг 6: Автоматическая замена URL...%s\n" "${BLUE}" "${NC}"
    printf "Старый URL (авто): %s%s%s\n" "${YELLOW}" "$OLD_URL_TO_REPLACE" "${NC}"
    printf "Новый URL:         %s%s%s\n" "${GREEN}" "$NEW_URL" "${NC}"
    ddev wp search-replace "$OLD_URL_TO_REPLACE" "$NEW_URL" --all-tables --allow-root
fi

printf "\n%s--- Финальный шаг ---%s\n" "${BLUE}" "${NC}"
read -p "Удалить файл бэкапа ($BACKUP_FILE_TO_CREATE)? (y/n) [n]: " DELETE_BACKUP
if [ "${DELETE_BACKUP:-n}" = "y" ]; then rm "$BACKUP_FILE_TO_CREATE"; printf "✔ Файл удалён\n"; else printf "✔ Файл сохранён\n"; fi

printf "\n%s🎉 Миграция успешно завершена! 🎉%s\n" "${GREEN}" "${NC}"
printf "Сайт доступен: %s%s%s\n" "${YELLOW}" "${NEW_URL}" "${NC}"
ddev launch