# Arch Linux: Руководство по настройке (Беларусь)

Этот документ содержит шаги по оптимизации системы Arch Linux, настройке автоматического обновления зеркал и решению распространенных проблем с доступом к репозиториям.

## 1. Оптимизация зеркал Pacman с помощью Reflector

reflector — это скрипт для выбора самых быстрых и актуальных зеркал (серверов с пакетами), что значительно ускоряет установку и обновление программ. Мы настроим его на автоматический еженедельный запуск.

### Шаг 1: Установка

Если reflector и micro еще не установлены:

Generated bash

```
sudo pacman -S reflector micro
```



### Шаг 2: Создание резервной копии

Всегда создавайте резервную копию перед изменениями.

Generated bash

```
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```


### Шаг 3: Настройка службы systemd для автоматизации

**1. Создание файла службы:**

Откройте файл с помощью micro.

Generated bash

```
sudo micro /etc/systemd/system/reflector.service
```


Вставьте в него следующий текст:

Generated ini

```
[Unit]
Description=Pacman mirrorlist update
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/reflector --verbose --country 'Belarus,Russia,Poland,Germany,Ukraine' --protocol https --age 12 --sort rate --latest 20 --save /etc/pacman.d/mirrorlist

[Install]
WantedBy=multi-user.target
```


> **Примечание:** Эта команда ищет 20 самых быстрых зеркал в Беларуси и соседних странах, которые обновлялись в последние 12 часов. Это надежный и сбалансированный подход.

Сохраните файл и закройте micro (обычно это Ctrl+S, затем Ctrl+Q).

**2. Создание файла таймера:**

Generated bash

```
sudo micro /etc/systemd/system/reflector.timer
```


Вставьте в него:

Generated ini

```
[Unit]
Description=Run reflector weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```


**3. Включение и запуск таймера:**

Generated bash

```
sudo systemctl enable reflector.timer
sudo systemctl start reflector.timer
```


Теперь ваш список зеркал будет обновляться автоматически раз в неделю.

## 2. Решение проблем с доступом

### Проблема 1: Ошибки при обновлении с зеркал Pacman

**Симптомы:**

Generated code

```
error: failed retrieving file '...' from ... : OpenSSL SSL_read: ... unexpected eof while reading
warning: too many errors from ..., skipping for the remainder of this transaction
```


**Причина:** Выбранное зеркало нестабильно или блокируется на маршруте.

**Решение:** Запустите reflector вручную с проверенными параметрами, чтобы немедленно получить новый список зеркал, а затем обновите систему.

Generated bash

```
sudo reflector --verbose --country 'Belarus,Russia,Poland,Germany,Ukraine' --protocol https --age 12 --sort rate --latest 20 --save /etc/pacman.d/mirrorlist
sudo pacman -Syyu
```


### Проблема 2: Ошибки доступа к AUR (Arch User Repository)

**Симптомы:**

Generated code

```
fatal: unable to access 'https://aur.archlinux.org/...' : Failed to connect to aur.archlinux.org port 443
```


**Причина:** Это почти всегда означает, что доступ к aur.archlinux.org **заблокирован на уровне интернет-провайдера**.

**Решение (основное и самое надежное): Использование VPN.**

1. Установите и включите любой VPN-сервис.
    
2. После активации VPN повторите команду установки/обновления пакета из AUR (например, yay -Syu).
    
3. Трафик пойдет через зашифрованный туннель, обходя блокировку.

включить warp - алиасы

```
w_start
w_stop
```

### Проблема 3: Сообщение "Flagged Out Of Date" от AUR-хелпера

**Симптомы:**

Generated code

```
Flagged Out Of Date AUR Packages: package-name
there is nothing to do
```


**Что это значит:** Это **не ошибка**. Это информационное сообщение. Сообщество пометило пакет как "устаревший", но его сопровождающий еще не обновил "рецепт" для сборки в AUR. Ваш AUR-хелпер (yay, paru) видит это, но поскольку сам рецепт не изменился, ему "нечего делать".

**Решение:** Просто проигнорируйте и подождите. Обычно через 1-2 дня пакет обновляют.

## 3. Полезные команды

- **Полное обновление системы (официальные репозитории + AUR):**
    
    Generated bash
    
    ```
    yay -Syu
    ```
    
    
- **Принудительное обновление баз данных pacman (нужно после ручной смены зеркал):**
    
    Generated bash
    
    ```
    sudo pacman -Syyu
    ```
    
    
- **Проверить статус автоматического обновления зеркал:**
    
    Generated bash
    
    ```
    systemctl status reflector.timer
    ```
    
    
- **Посмотреть текущий список зеркал:**
    
    Generated bash
    
    ```
    micro /etc/pacman.d/mirrorlist
    ```
    