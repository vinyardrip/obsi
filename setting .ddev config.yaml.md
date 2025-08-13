
# .ddev/config.yaml

name: my-hoster-by-project
type: wordpress # Укажите тип вашего проекта (wordpress, drupal, laravel, php)
docroot: "public_html" # На шаред-хостингах часто используется эта папка

# --- САМАЯ ВАЖНАЯ ЧАСТЬ ---

# 1. Указываем версию PHP, которую вы нашли в cPanel
php_version: "7.4" # ЗАМЕНИТЕ НА ВАШУ ВЕРСИЮ С ХОСТИНГА

# 2. Указываем, что PHP должен обрабатывать Apache (чтобы работали .htaccess)
#    Это эмулирует вашу связку Apache + Nginx
webserver_type: apache-fpm

# 3. Указываем версию базы данных, которую вы нашли в phpMyAdmin
database:
  type: mariadb
  version: "10.6" # ЗАМЕНИТЕ НА ВАШУ ВЕРСИЮ С ХОСТИНГА

# --- КОНЕЦ ВАЖНОЙ ЧАСТИ ---

# Включаем phpMyAdmin
phpmyadmin: true
# Для более свежих версий DDEV может понадобиться `ddev add-on get ddev/ddev-phpmyadmin`
# Если вы уже установили дополнение, эта строка не нужна.

# Остальные настройки DDEV по умолчанию...