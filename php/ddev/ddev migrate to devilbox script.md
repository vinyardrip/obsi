```

#!/bin/bash
# ^-- Это "shebang". Он говорит системе, что для выполнения этого файла нужно
# использовать интерпретатор Bash. Обязателен для всех скриптов.

# --- Настройка безопасности скрипта ---
# Эта команда заставляет скрипт немедленно завершиться, если любая из команд
# завершится с ошибкой. Это предотвращает непредсказуемое поведение и
# частичное выполнение миграции.
set -e

# --- ⚙️ БАЗОВАЯ КОНФИГУРАЦИЯ ---
# Здесь мы задаем основные пути. Это единственное место, которое, возможно,
# когда-нибудь потребуется отредактировать, если вы измените структуру своих папок.

# Путь к корневой директории Devilbox
DEVILBOX_PATH_ROOT="$HOME/projects/work/devilbox"
# Путь к директории, где лежат папки с сайтами в Devilbox
DEVILBOX_DATA_BASE_PATH="${DEVILBOX_PATH_ROOT}/data"
# Путь, куда будут создаваться новые проекты DDEV
DDEV_PROJECTS_BASE_PATH="$HOME/projects/work/php"
# Путь для централизованного хранения бэкапов БД
DB_BACKUP_PATH="$HOME/projects/work/db_local"


# --- Утилиты и цвета ---
# Здесь мы объявляем переменные для цветного вывода в терминале.
# Это ANSI escape-коды. Они делают вывод скрипта более читабельным.
GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; RED='\033[0;31m';
NC='\033[0m' # No Color (сброс цвета)


# --- 📝 ИНТЕРАКТИВНАЯ НАСТРОЙКА ---
# Этот блок отвечает за сбор информации от пользователя.
echo -e "${BLUE}--- Настройка миграции проекта ---${NC}"

# Команда 'read' с ключом '-p' выводит приглашение для ввода (prompt).
read -p "1. Имя проекта для DDEV (например, grandmebel): " PROJECT_NAME

# Здесь мы спрашиваем только имя папки, а не полный путь, для удобства.
read -p "2. Имя папки проекта в Devilbox/data [${PROJECT_NAME:-grandmebel}]: " DEVILBOX_PROJECT_FOLDER
# >> Важный нюанс Bash: Конструкция `${VAR:-default}` означает:
# "Использовать значение переменной VAR, но если она пустая, использовать 'default'".
# Здесь, если пользователь просто нажмет Enter, будет использовано значение из PROJECT_NAME.
DEVILBOX_PROJECT_FOLDER=${DEVILBOX_PROJECT_FOLDER:-${PROJECT_NAME:-grandmebel}}

# "Склеиваем" базовый путь и имя папки, чтобы получить полный путь к исходному проекту.
DEVILBOX_PATH_PROJECT="${DEVILBOX_DATA_BASE_PATH}/${DEVILBOX_PROJECT_FOLDER}"

# Проверяем, существует ли указанная директория.
# `if [ ! -d "..." ]` означает "если НЕ существует директория...".
# Это важная проверка, чтобы не запустить скрипт с неверными данными.
if [ ! -d "$DEVILBOX_PATH_PROJECT" ]; then
    echo -e "\n${RED}ОШИБКА: Исходная директория проекта не найдена:${NC} $DEVILBOX_PATH_PROJECT"; exit 1;
fi

# Запрашиваем версию PHP для временного запуска Devilbox.
read -p "3. Какую версию PHP Devilbox запустить для экспорта? (6/7/8): " DEVILBOX_PHP_VERSION
# `if ! [[ "$VAR" =~ ^[6-8]$ ]]` — это проверка с помощью регулярного выражения.
# `=~` — оператор сравнения с "регуляркой".
# `^[6-8]$` — паттерн, означающий: "строка должна начинаться (^), содержать одну
# цифру от 6 до 8, и на этом заканчиваться ($)".
if ! [[ "$DEVILBOX_PHP_VERSION" =~ ^[6-8]$ ]]; then
    echo -e "${RED}ОШИБКА: Неверная версия PHP. Допустимы только 6, 7 или 8.${NC}"; exit 1;
fi

# Запрашиваем остальные настройки для DDEV.
read -p "4. Тип проекта DDEV (wordpress, laravel, php) [wordpress]: " PROJECT_TYPE
PROJECT_TYPE=${PROJECT_TYPE:-wordpress}
read -p "5. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION
DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}

echo -e "${BLUE}--- Учетные данные БД в Devilbox ---${NC}"
read -p "Имя БД [${PROJECT_NAME:-grandmebel}]: " DB_NAME
DB_NAME=${DB_NAME:-${PROJECT_NAME:-grandmebel}}
read -p "Пользователь БД [root]: " DB_USER
DB_USER=${DB_USER:-root}
# Ключ '-s' (silent) у команды 'read' скрывает ввод. Идеально для паролей.
read -s -p "Пароль БД (оставьте пустым, если его нет): " DB_PASS
echo "" # Добавляем перенос строки, так как '-s' его не делает.


# --- Формирование вычисляемых переменных ---
# На основе введенных данных мы конструируем остальные нужные нам переменные.
DDEV_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
OLD_URL="httpshttps://${DEVILBOX_PROJECT_FOLDER}.loc"
NEW_URL="https://${PROJECT_NAME}.lo" # Сразу используем https, так как DDEV настроит его для .lo
# >> Важный нюанс Bash: `$(команда)` — это "command substitution".
# Bash сначала выполнит команду `date +%Y-%m-%d`, а затем подставит
# ее результат (например, "2025-08-13") в строку.
BACKUP_FILE_PATH="${DB_BACKUP_PATH}/${PROJECT_NAME}-$(date +%Y-%m-%d).sql"


# --- ФИНАЛЬНОЕ ПОДТВЕРЖДЕНИЕ ---
# "Точка невозврата". Даем пользователю шанс проверить все данные перед запуском
# потенциально деструктивных команд. Хорошая практика для любых скриптов.
echo -e "\n${YELLOW}--- Пожалуйста, проверьте данные ---${NC}"
echo "Источник (Devilbox):        ${GREEN}${DEVILBOX_PATH_PROJECT}${NC}"
echo "Назначение (DDEV):          ${GREEN}${DDEV_PATH}${NC}"
echo "Новый основной URL:          ${GREEN}${NEW_URL}${NC}"
echo "-------------------------------------"
read -p "Продолжить миграцию? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then echo "Миграция отменена."; exit 1; fi


# --- СКРИПТ АВТОМАТИЗАЦИИ ---

# --- Шаг 1: Управление Devilbox для экспорта ---
echo -e "\n${BLUE}Шаг 1.1: Запуск Devilbox (PHP ${DEVILBOX_PHP_VERSION}) для экспорта...${NC}"
# Переходим в корневую папку Devilbox, так как `docker-compose` ищет .yml файл в текущей директории.
cd "$DEVILBOX_PATH_ROOT"
# Воспроизводим вашу логику переключения .env файлов.
# Сначала прячем основной .env, если он есть, чтобы не затереть его.
if [ -f ".env" ]; then mv .env env; fi
# Переименовываем нужный .env-X в .env, активируя нужную версию PHP.
mv ".env-${DEVILBOX_PHP_VERSION}" .env
# Запускаем контейнеры. Ключ '-d' (detached) запускает их в фоновом режиме.
docker-compose up -d httpd php mysql mailhog
echo -e "${YELLOW}Ожидание стабилизации контейнера БД (10 секунд)...${NC}"
# `sleep 10` — простая пауза. Нужна, чтобы MySQL успел полностью запуститься
# и был готов принимать подключения. Без нее `mysqldump` может провалиться.
sleep 10

# Динамически формируем аргумент для пароля.
# Это решает проблему с пустыми паролями.
PASSWORD_ARG=""
# `if [ -n "$VAR" ]` означает "если переменная VAR НЕ пустая...".
if [ -n "$DB_PASS" ]; then PASSWORD_ARG="-p${DB_PASS}"; fi

echo -e "${BLUE}Шаг 1.2: Экспорт базы данных...${NC}"
# `mkdir -p` создает директорию, только если ее не существует. Ключ '-p'
# также создает все родительские директории.
mkdir -p "$DB_BACKUP_PATH"
# `docker-compose exec -T` выполняет команду внутри работающего контейнера.
# `-T` отключает псевдо-терминал, что рекомендуется для скриптов.
# `$PASSWORD_ARG` используется без кавычек, чтобы если переменная пуста,
# bash просто убрал ее, не подставив пустые кавычки.
docker-compose exec -T mysql mysqldump -u "$DB_USER" $PASSWORD_ARG "$DB_NAME" > "$BACKUP_FILE_PATH"
echo "✔ База данных сохранена в $BACKUP_FILE_PATH"

echo -e "${BLUE}Шаг 1.3: Остановка Devilbox и освобождение портов...${NC}"
# `docker-compose down` — более мощная команда, чем 'stop'. Она не только
# останавливает, но и удаляет контейнеры и сети, полностью очищая ресурсы.
docker-compose down
# Возвращаем .env файлы в исходное состояние.
mv .env ".env-${DEVILBOX_PHP_VERSION}"
if [ -f "env" ]; then mv env .env; fi
echo "✔ Devilbox остановлен, порты свободны."


# --- Шаг 2: Настройка DDEV ---
echo -e "${BLUE}Шаг 2: Копирование файлов проекта...${NC}"
mkdir -p "$DDEV_PROJECTS_BASE_PATH"
# `cp -R` копирует директорию рекурсивно, со всем содержимым.
cp -R "$DEVILBOX_PATH_PROJECT" "$DDEV_PATH"
echo "✔ Файлы скопированы в $DDEV_PATH"

# Переходим в новую папку, чтобы все дальнейшие команды (ddev, git)
# выполнялись в контексте нового проекта.
cd "$DDEV_PATH"

echo -e "${BLUE}Шаг 3: Настройка DDEV и Git...${NC}"
mkdir -p .ddev
# >> Важный нюанс Bash: `cat << EOF > file.txt` — это "Here Document".
# Все, что находится между `EOF` и `EOF`, будет записано в `file.txt`.
# Это удобный способ создавать многострочные файлы прямо в скрипте.
# Переменные внутри этого блока ($PROJECT_NAME и др.) будут заменены на их значения.
cat << EOF > .ddev/config.yaml
name: $PROJECT_NAME
type: $PROJECT_TYPE
docroot: ""
php_version: "$DDEV_PHP_VERSION"
webserver_type: apache-fpm
# Устанавливаем .lo как основной домен верхнего уровня для проекта
project_tld: lo

# Явно указываем имя базы данных 
database: 
	type: mariadb # или mysql 
	version: "10.11" # укажите нужную версию 
	name: $PROJECT_NAME
EOF
# `> /dev/null 2>&1` — конструкция для полного подавления вывода команды.
# `> /dev/null` перенаправляет стандартный вывод (stdout) в "черную дыру".
# `2>&1` перенаправляет вывод ошибок (stderr) туда же, куда и стандартный.
ddev get ddev/ddev-phpmyadmin > /dev/null 2>&1

# "Умная" обработка Git.
if [ -d ".git" ]; then
    # Если репозиторий уже есть, просто добавляем новые файлы DDEV.
    git add .ddev/config.yaml .ddev/phpmyadmin.yaml
    # Делаем коммит, только если действительно есть что коммитить.
    if [[ -n $(git status --porcelain) ]]; then git commit -m "chore: Configure DDEV for local development"; fi
    echo "✔ Конфигурация DDEV добавлена в существующий Git-репозиторий."
else
    # Если репозитория нет, инициализируем новый.
    curl -s -o .gitignore https://raw.githubusercontent.com/github/gitignore/main/WordPress.gitignore
    git init -b main > /dev/null 2>&1; git add . > /dev/null 2>&1; git commit -m "Initial commit: Project setup with DDEV" > /dev/null 2>&1
    echo "✔ Создан новый Git-репозиторий."
fi

echo -e "${BLUE}Шаг 4: Запуск проекта DDEV...${NC}"
echo -e "${YELLOW}Это может занять несколько минут при первом запуске этой версии PHP.${NC}"
# `-y` (yes) автоматически отвечает "да" на все вопросы от DDEV.
ddev start -y

echo -e "${BLUE}Шаг 5: Импорт базы данных...${NC}"
ddev import-db --file="$BACKUP_FILE_PATH"

echo -e "${BLUE}Шаг 6: Обновление URL...${NC}"
# Теперь мы сразу меняем старый URL на новый правильный URL с https.
ddev exec wp search-replace "$OLD_URL" "$NEW_URL" --all-tables > /dev/null 2>&1
echo "✔ URL в базе данных обновлены."

echo -e "${BLUE}Шаг 7: Завершение...${NC}"
read -p "Хотите удалить файл бэкапа (${BACKUP_FILE_PATH})? (y/n) [n]: " DELETE_BACKUP
# `${VAR:-n}` — если пользователь нажмет Enter, будет использовано значение 'n'.
if [[ "${DELETE_BACKUP:-n}" == "y" ]]; then
    rm "$BACKUP_FILE_PATH"
    echo "✔ Временный файл бэкапа удален."
else
    echo "✔ Файл бэкапа сохранен."
fi

echo -e "\n${GREEN}🎉 Миграция успешно завершена! 🎉${NC}"
echo -e "Сайт доступен по адресу: ${YELLOW}${NEW_URL}${NC}"