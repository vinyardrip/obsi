```bash
#!/bin/bash
# ^-- Это "shebang". Он говорит системе, что для выполнения этого файла нужно
# использовать интерпретатор Bash. Обязателен для всех скриптов.

# --- Настройка безопасности скрипта ---
# Эта команда заставляет скрипт немедленно завершиться, если любая из команд
# завершится с ошибкой. Это предотвращает непредсказуемое поведение.
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
# Это делает вывод скрипта более читабельным.
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
if [ ! -d "$DEVILBOX_PATH_PROJECT" ]; then
    echo -e "\n${RED}ОШИБКА: Исходная директория проекта не найдена:${NC} $DEVILBOX_PATH_PROJECT"; exit 1;
fi

# Запрашиваем версию PHP для временного запуска Devilbox.
read -p "3. Какую версию PHP Devilbox запустить для экспорта? (6/7/8): " DEVILBOX_PHP_VERSION
# `if ! [[ "$VAR" =~ ^[6-8]$ ]]` — это проверка с помощью регулярного выражения.
if ! [[ "$DEVILBOX_PHP_VERSION" =~ ^[6-8]$ ]]; then
    echo -e "${RED}ОШИБКА: Неверная версия PHP. Допустимы только 6, 7 или 8.${NC}"; exit 1;
fi

# Запрашиваем остальные настройки для DDEV.
read -p "4. Тип проекта DDEV (wordpress, laravel, php) [wordpress]: " PROJECT_TYPE
PROJECT_TYPE=${PROJECT_TYPE:-wordpress}
read -p "5. Версия PHP для DDEV [7.4]: " DDEV_PHP_VERSION
DDEV_PHP_VERSION=${DDEV_PHP_VERSION:-7.4}

echo -e "${BLUE}--- Учетные данные БД в Devilbox (для экспорта) ---${NC}"
# Мы используем уникальные имена переменных с суффиксом _FOR_EXPORT, чтобы избежать
# конфликтов с системными переменными, которые может создавать DDEV.
read -p "6. Имя БД в Devilbox [${PROJECT_NAME:-grandmebel}]: " DB_NAME_FOR_EXPORT
DB_NAME_FOR_EXPORT=${DB_NAME_FOR_EXPORT:-${PROJECT_NAME:-grandmebel}}
read -p "7. Пользователь БД в Devilbox [root]: " DB_USER_FOR_EXPORT
DB_USER_FOR_EXPORT=${DB_USER_FOR_EXPORT:-root}
# Ключ '-s' (silent) у команды 'read' скрывает ввод. Идеально для паролей.
read -s -p "8. Пароль БД в Devilbox (оставьте пустым, если его нет): " DB_PASS_FOR_EXPORT
echo "" # Добавляем перенос строки, так как '-s' его не делает.


# --- Формирование вычисляемых переменных ---
# На основе введенных данных мы конструируем остальные нужные нам переменные.
DDEV_PATH="${DDEV_PROJECTS_BASE_PATH}/${PROJECT_NAME}"
OLD_URL="httpshttps://${DEVILBOX_PROJECT_FOLDER}.loc"
NEW_URL="https://${PROJECT_NAME}.lo" # Сразу используем https, так как DDEV настроит его для .lo
# >> Важный нюанс Bash: `$(команда)` — это "command substitution".
# Bash сначала выполнит команду `date +%Y-%m-%d`, а затем подставит
# ее результат (например, "2025-08-13") в строку.
BACKUP_FILE_TO_CREATE="${DB_BACKUP_PATH}/${PROJECT_NAME}-$(date +%Y-%m-%d).sql"


# --- ФИНАЛЬНОЕ ПОДТВЕРЖДЕНИЕ ---
# "Точка невозврата". Даем пользователю шанс проверить все данные перед запуском.
echo -e "\n${YELLOW}--- Пожалуйста, проверьте данные ---${NC}"
echo "Источник (Devilbox):        ${GREEN}${DEVILBOX_PATH_PROJECT}${NC}"
echo "Назначение (DDEV):          ${GREEN}${DDEV_PATH}${NC}"
echo "Новый основной URL:          ${GREEN}${NEW_URL}${NC}"
echo "Новый бэкап будет создан как: ${GREEN}${BACKUP_FILE_TO_CREATE}${NC}"
echo "-------------------------------------"
read -p "Продолжить миграцию? (y/n): " CONFIRM
if [ "$CONFIRM" != "y" ]; then echo "Миграция отменена."; exit 1; fi


# --- СКРИПТ АВТОМАТИЗАЦИИ ---

# --- Шаг 1: Управление Devilbox для экспорта ---
echo -e "\n${BLUE}Шаг 1.1: Запуск Devilbox (PHP ${DEVILBOX_PHP_VERSION}) для экспорта...${NC}"
# Переходим в корневую папку Devilbox, так как `docker-compose` ищет .yml файл в текущей директории.
cd "$DEVILBOX_PATH_ROOT"
# Воспроизводим вашу логику переключения .env файлов.
if [ -f ".env" ]; then mv .env env; fi
mv ".env-${DEVILBOX_PHP_VERSION}" .env
# Запускаем контейнеры. Ключ '-d' (detached) запускает их в фоновом режиме.
docker-compose up -d httpd php mysql mailhog
echo -e "${YELLOW}Ожидание стабилизации контейнера БД (10 секунд)...${NC}"
# `sleep 10` — простая пауза. Нужна, чтобы MySQL успел полностью запуститься.
sleep 10

# Динамически формируем аргумент для пароля.
PASSWORD_ARG=""
# `if [ -n "$VAR" ]` означает "если переменная VAR НЕ пустая...".
if [ -n "$DB_PASS_FOR_EXPORT" ]; then PASSWORD_ARG="-p${DB_PASS_FOR_EXPORT}"; fi

echo -e "${BLUE}Шаг 1.2: Экспорт базы данных...${NC}"
# `mkdir -p` создает директорию, только если ее не существует.
mkdir -p "$DB_BACKUP_PATH"
# `docker-compose exec -T` выполняет команду внутри работающего контейнера.
# `$PASSWORD_ARG` используется без кавычек, чтобы если переменная пуста,
# bash просто убрал ее, не подставив пустые кавычки.
docker-compose exec -T mysql mysqldump -u "$DB_USER_FOR_EXPORT" $PASSWORD_ARG "$DB_NAME_FOR_EXPORT" > "$BACKUP_FILE_TO_CREATE"
echo "✔ База данных сохранена в $BACKUP_FILE_TO_CREATE"

echo -e "${BLUE}Шаг 1.3: Остановка Devilbox и освобождение портов...${NC}"
# `docker-compose down` — останавливает и удаляет контейнеры, полностью очищая ресурсы.
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
echo "✔ Файлы скопированы."

# Переходим в новую папку, чтобы все дальнейшие команды выполнялись в контексте нового проекта.
cd "$DDEV_PATH"

echo -e "${BLUE}Шаг 3: Настройка DDEV и Git...${NC}"
mkdir -p .ddev
# >> `cat << EOF > file.txt` — "Here Document".
# Удобный способ создавать многострочные файлы прямо в скрипте.
cat << EOF > .ddev/config.yaml
name: $PROJECT_NAME
type: $PROJECT_TYPE
docroot: ""
php_version: "$DDEV_PHP_VERSION"
webserver_type: apache-fpm
# Устанавливаем .lo как основной домен верхнего уровня для проекта
project_tld: lo

# Примечание: мы НЕ указываем здесь имя БД, так как выяснили,
# что данная версия DDEV для WordPress его игнорирует и всегда
# использует 'db'. Это позволяет избежать путаницы.
EOF
# `> /dev/null 2>&1` — конструкция для полного подавления вывода.
ddev get ddev/ddev-phpmyadmin > /dev/null 2>&1

# "Умная" обработка Git.
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
# `-y` (yes) автоматически отвечает "да" на все вопросы от DDEV.
ddev start -y

# *** КЛЮЧЕВОЕ ИЗМЕНЕНИЕ: Надежный поиск и импорт самого свежего бэкапа ***
echo -e "${BLUE}Шаг 5: Поиск и импорт самого свежего бэкапа...${NC}"
# Ищем самый свежий бэкап для нашего проекта. Работает в bash и zsh.
# `ls -t ...` - сортирует файлы по времени изменения (самые новые - вверху)
# `head -n1` - берет только первую строку из вывода (т.е. самый свежий файл)
BACKUP_TO_IMPORT=$(ls -t "${DB_BACKUP_PATH}/${PROJECT_NAME}"*.sql 2>/dev/null | head -n1)
# Проверяем, а нашелся ли вообще файл.
if [ -z "$BACKUP_TO_IMPORT" ]; then
    echo -e "${RED}ОШИБКА: Не удалось найти файл бэкапа для импорта!${NC}"
    exit 1
fi
echo "Импортируем файл: $BACKUP_TO_IMPORT"
ddev import-db --file="$BACKUP_TO_IMPORT"

echo -e "${BLUE}Шаг 6: Обновление URL...${NC}"
# Теперь мы сразу меняем старый URL на новый правильный URL с https.
ddev exec wp search-replace "$OLD_URL" "$NEW_URL" --all-tables > /dev/null 2>&1
echo "✔ URL в базе данных обновлены."

echo -e "${BLUE}Шаг 7: Завершение...${NC}"
read -p "Хотите удалить импортированный файл бэкапа (${BACKUP_TO_IMPORT})? (y/n) [n]: " DELETE_BACKUP
# `${VAR:-n}` — если пользователь нажмет Enter, будет использовано значение 'n'.
if [[ "${DELETE_BACKUP:-n}" == "y" ]]; then
    rm "$BACKUP_TO_IMPORT"
    echo "✔ Файл бэкапа удален."
else
    echo "✔ Файл бэкапа сохранен."
fi

echo -e "\n${GREEN}🎉 Миграция успешно завершена! 🎉${NC}"
echo -e "Сайт доступен по адресу: ${YELLOW}${NEW_URL}${NC}"
```