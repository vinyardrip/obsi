
#!/bin/bash

# =================================================================
# Интерактивный скрипт для миграции PHP-проекта с Devilbox на DDEV
# ВЕРСИЯ 11.0 ("Полный Автопилот")
# =================================================================

set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"
DEVILBOX_DATA_BASE_PATH="${DEVILBOX_PATH_ROOT}/data"
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
DB_BACKUP_PATH="$HOME/projects/work/db_local"

# --- Утилиты и цвета ---
GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m'; NC='\033[0m'

# --- 📝 ИНТЕРАКТИВНАЯ НАСТРОЙКА ---
echo -e "${BLUE}--- Настройка миграции проекта ---${NC}"

read -p "1. Имя проекта для DDEV (например, grandmebel): " PROJECT_NAME
read -p "2. Имя папки проекта в Devilbox/data [${PROJECT_NAME:-grandmebel}]: " DEVILBOX_PROJECT_FOLDER
DEVILBOX_PROJECT_FOLDER=${DEVILBOX_PROJECT_FOLDER:-${PROJECT_NAME:-grandmebel}}

DEVILBOX_PATH_PROJECT="${DEVILBOX_DATA_BASE_PATH}/${DEVILBOX_PROJECT_FOLDER}"

if [ ! -d "$DEVILBOX_PATH_PROJECT" ]; then
    echo -e "\n${RED}ОШИБКА: Исходная директория проекта не найдена:${NC} $DEVILBOX_PATH_PROJECT"; exit 1;
fi

# --- Новый промпт для управления Devilbox ---
read -p "3. Какую версию PHP Devilbox запустить для экспорта? (6/7/8): " DEVILBOX_PHP_VERSION
if ! [[ "$DEVILBOX_PHP_VERSION" =~ ^[6-8]$ ]]; then
    echo -e "${RED}ОШИБКА: Неверная версия PHP. Допустимы только 6, 7 или 8.${NC}"; exit 1;
fi

read -p "4. Тип проекта DDEV (wordpress, laravel, php) [wordpress]: " PROJECT_TYPE
PROJECT_TYPE=${PROJECT_TYPE:-wordpress}
read -p "5. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION
DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}

echo -e "${BLUE}--- Учетные данные БД в Devilbox ---${NC}"
read -p "Имя БД [${PROJECT_NAME:-grandmebel}]: " DB_NAME
DB_NAME=${DB_NAME:-${PROJECT_NAME:-grandmebel}}
read -p "Пользователь БД [root]: " DB_USER
DB_USER=${DB_USER:-root}
read -s -p "Пароль БД (оставьте пустым, если его нет): " DB_PASS
echo ""

# --- Формирование вычисляемых переменных ---
DDEV_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
OLD_URL="httpshttps://${DEVILBOX_PROJECT_FOLDER}.loc"
NEW_URL="http://${PROJECT_NAME}.lo"
BACKUP_FILE_PATH="${DB_BACKUP_PATH}/${PROJECT_NAME}-$(date +%Y-%m-%d).sql"

# --- ФИНАЛЬНОЕ ПОДТВЕРЖДЕНИЕ ---
echo -e "\n${YELLOW}--- Пожалуйста, проверьте данные ---${NC}"
echo "Источник (Devilbox):        ${GREEN}${DEVILBOX_PATH_PROJECT}${NC}"
echo "Назначение (DDEV):          ${GREEN}${DDEV_PATH}${NC}"
echo "Будет временно запущен:    ${RED}Devilbox с PHP ${DEVILBOX_PHP_VERSION}${NC}"
echo "-------------------------------------"
read -p "Продолжить миграцию? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then echo "Миграция отменена."; exit 1; fi

# --- СКРИПТ АВТОМАТИЗАЦИИ ---

# --- Шаг 1: Управление Devilbox для экспорта ---
echo -e "\n${BLUE}Шаг 1.1: Запуск Devilbox (PHP ${DEVILBOX_PHP_VERSION}) для экспорта...${NC}"
cd "$DEVILBOX_PATH_ROOT"
if [ -f ".env" ]; then
    mv .env env
fi
mv ".env-${DEVILBOX_PHP_VERSION}" .env
docker-compose up -d httpd php mysql mailhog
echo -e "${YELLOW}Ожидание стабилизации контейнера БД (10 секунд)...${NC}"
sleep 10 # Даем MySQL время полностью запуститься

PASSWORD_ARG=""
if [ -n "$DB_PASS" ]; then PASSWORD_ARG="-p${DB_PASS}"; fi
echo -e "${BLUE}Шаг 1.2: Экспорт базы данных...${NC}"
mkdir -p "$DB_BACKUP_PATH"
docker-compose exec -T mysql mysqldump -u "$DB_USER" $PASSWORD_ARG "$DB_NAME" > "$BACKUP_FILE_PATH"
echo "✔ База данных сохранена в $BACKUP_FILE_PATH"

echo -e "${BLUE}Шаг 1.3: Остановка Devilbox и освобождение портов...${NC}"
docker-compose down # Используем 'down' для полной очистки
mv .env ".env-${DEVILBOX_PHP_VERSION}"
if [ -f "env" ]; then
    mv env .env
fi
echo "✔ Devilbox остановлен, порты свободны."


# --- Шаг 2: Настройка DDEV ---
echo -e "${BLUE}Шаг 2: Копирование файлов проекта...${NC}"
mkdir -p "$DDEV_PROJECTS_BASE_PATH"
cp -R "$DEVILBOX_PATH_PROJECT" "$DDEV_PATH"
echo "✔ Файлы скопированы в $DDEV_PATH"

cd "$DDEV_PATH"

echo -e "${BLUE}Шаг 3: Настройка DDEV и Git...${NC}"
mkdir -p .ddev
cat << EOF > .ddev/config.yaml
name: $PROJECT_NAME
type: $PROJECT_TYPE
docroot: ""
php_version: "$DDEV_PHP_VERSION"
webserver_type: apache-fpm
additional_hostnames:
  - "${PROJECT_NAME}.lo"
EOF
ddev get ddev/ddev-phpmyadmin > /dev/null 2>&1
# ... (блок с Git без изменений)
if [ -d ".git" ]; then
    git add .ddev/config.yaml .ddev/phpmyadmin.yaml
    if [[ -n $(git status --porcelain) ]]; then git commit -m "chore: Configure DDEV for local development"; fi
    echo "✔ Конфигурация DDEV добавлена в существующий Git-репозиторий."
else
    curl -s -o .gitignore https://raw.githubusercontent.com/github/gitignore/main/WordPress.gitignore
    git init -b main > /dev/null 2>&1; git add . > /dev/null 2>&1; git commit -m "Initial commit: Project setup with DDEV" > /dev/null 2>&1
    echo "✔ Создан новый Git-репозиторий."
fi

echo -e "${BLUE}Шаг 4: Запуск проекта DDEV...${NC}"
echo -e "${YELLOW}Это может занять несколько минут при первом запуске этой версии PHP.${NC}"
ddev start -y

echo -e "${BLUE}Шаг 5: Импорт базы данных...${NC}"
ddev import-db --file="$BACKUP_FILE_PATH"

echo -e "${BLUE}Шаг 6: Обновление URL...${NC}"
ddev exec wp search-replace "$OLD_URL" "$NEW_URL" --all-tables > /dev/null 2>&1

echo -e "${BLUE}Шаг 7: Завершение...${NC}"
read -p "Хотите удалить файл бэкапа (${BACKUP_FILE_PATH})? (y/n) [n]: " DELETE_BACKUP
if [[ "${DELETE_BACKUP:-n}" == "y" ]]; then
    rm "$BACKUP_FILE_PATH"
    echo "✔ Временный файл бэкапа удален."
else
    echo "✔ Файл бэкапа сохранен."
fi

echo -e "\n${GREEN}🎉 Миграция успешно завершена! 🎉${NC}"
echo -e "Сайт доступен по адресу: ${YELLOW}${NEW_URL}${NC}"