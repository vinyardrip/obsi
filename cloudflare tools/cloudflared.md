# 🛠 Утилита: cloudflared (tunnel)

**Описание:** Создание защищенных туннелей для проброса локальных портов во внешнюю сеть.
**Вспомогательный файл:** `~/projects/tools/cloudflared-php-helper.php`

---

## 📥 1. Установка (Arch / Manjaro)
```bash
sudo pacman -Syu cloudflared
```


🚀 2. Варианты запуска
А) Фронтенд / Чистый проект (Vite, Webpack, HTML)
Прямой проброс порта сборщика или локального сервера.

```
￼
# Замени port-project на нужный (5173, 3000, 8080)
cloudflared tunnel --url http://localhost:port-project
```

Б) Проекты с бэкендом (DDEV / WordPress / Laravel)
Проброс через порт DDEV с передачей Host-заголовка.

# Порт берем из `ddev describe` (колонки 127.0.0.1:XXXXX)

```
cloudflared tunnel --url [http://127.0.0.1](http://127.0.0.1):ПОРТ --http-host-header "PROJECT_NAME.lo"
```
⚙️ 3. Глобальная автоматизация DDEV (PHP)
Настройка среды, чтобы проекты автоматически адаптировались под адрес туннеля.

Шаг 1: Конфиг PHP
Создать файл ~/.ddev/php/global_prepend.ini:

Ini, TOML
￼
auto_prepend_file = "/var/www/html/.ddev_tools/cloudflared-php-helper.php"
Шаг 2: Монтирование папки в Docker
Создать файл ~/.ddev/docker-compose.global_tools.yaml:

YAML
￼
services:
  web:
    volumes:
      - ~/projects/tools:/var/www/html/.ddev_tools:ro
📝 4. Код хелпера (~/projects/tools/cloudflared-php-helper.php)
Этот скрипт правит заголовки «на лету» для корректной генерации ссылок.

PHP
￼
<?php
/**
 * Корректировка окружения под туннель cloudflared
 */
if (isset($_SERVER['HTTP_X_FORWARDED_HOST']) && strpos($_SERVER['HTTP_X_FORWARDED_HOST'], 'trycloudflare.com') !== false) {
    
    $_SERVER['HTTP_HOST'] = $_SERVER['HTTP_X_FORWARDED_HOST'];
    $_SERVER['HTTPS'] = 'on';
    $_SERVER['SERVER_PORT'] = 443;

    // Авто-настройка констант WordPress
    if (!defined('WP_HOME')) define('WP_HOME', 'https://' . $_SERVER['HTTP_HOST']);
    if (!defined('WP_SITEURL')) define('WP_SITEURL', 'https://' . $_SERVER['HTTP_HOST']);
}
🐙 5. Управление инструментами (Git)
Сохранение настроек и хелперов в приватный репозиторий.

Bash
￼
cd ~/projects/tools
git init
git add .
git commit -m "feat: setup cloudflared tunnel tools and helpers"
# Создать приватный репо и запушить
gh repo create ddev-global-tools --private --source=. --remote=origin --push