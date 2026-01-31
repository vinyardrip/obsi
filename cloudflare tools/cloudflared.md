### 🛠 Утилита: cloudflared (tunnel)

**Описание:** Создание защищенных туннелей для проброса локальных портов во внешнюю сеть. **Вспомогательный файл:** `~/projects/tools/cloudflared-php-helper.php`

#### 📥 1. Установка (Arch / Manjaro)

Bash

```
sudo pacman -Syu cloudflared
```

#### ⚙️ 2. Глобальная настройка среды (Один раз на новой системе)

Чтобы DDEV подхватывал глобальные конфиги из домашней папки:

Bash

```
ddev config global --use-docker-compose-from-home
```

#### 🚀 3. Глобальная автоматизация DDEV (PHP)

**Шаг 1: Конфиг PHP** Создать файл `~/.ddev/php/global_prepend.ini`:

Ini, TOML

```
auto_prepend_file = "/var/www/html/.ddev_tools/cloudflared-php-helper.php"
```

**Шаг 2: Монтирование папки в Docker** Создать файл `~/.ddev/docker-compose.global_tools.yaml`:

YAML

```
version: '3.6'
services:
  web:
    volumes:
      # Использование переменной ${HOME} делает конфиг переносимым между машинами
      - ${HOME}/projects/tools:/var/www/html/.ddev_tools:ro
```

#### 🔍 4. Проверка (Тест-драйв)

После настройки выполни в любом проекте:

Bash

```
ddev restart
ddev ssh -s web "ls -la /var/www/html/.ddev_tools"
```

_Если файл виден — монтирование успешно._

#### 📝 5. Код хелпера (`~/projects/tools/cloudflared-php-helper.php`)

PHP

```
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
```

#### 🚀 6. Запуск туннеля

1. Получи порт: `ddev describe` (раздел `127.0.0.1:XXXXX`).
    
2. Запусти:
    

Bash

```
cloudflared tunnel --url http://127.0.0.1:ПОРТ --http-host-header "PROJECT_NAME.lo"
```

---