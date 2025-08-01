
# Настройка Box в Linux с помощью Rclone

## Проблема

Облачный сервис **Box** не предоставляет официального клиента для Linux. Это делает невозможной прямую синхронизацию файлов. Попытки работать через служебные папки клиента для Windows (AppData) ненадежны и рискованны.

## Решение

Использовать утилиту командной строки **Rclone**. Она выступит в роли моста между облаком Box и файловой системой Linux, монтируя облачное хранилище как локальную папку. Это надежный, стабильный и рекомендованный способ.

---

### Шаг 1: Установка Rclone

Устанавливаем Rclone из официальных репозиториев. Для Arch-based дистрибутивов (EndeavourOS, Manjaro) команда следующая:

Generated bash

```
sudo pacman -S rclone
```

---

### Шаг 2: Первичная настройка (rclone config)

Это разовый интерактивный процесс для "знакомства" Rclone с вашим аккаунтом Box.

1. Запустите конфигуратор в терминале:
    
    Generated bash
    
    ````
    rclone config
    ```2.  Rclone покажет список текущих настроек (сначала он будет пуст). Выберите `n` (создать новую):
    `n) New remote`
    ````
        
2. Придумайте имя для вашего подключения. Оно будет использоваться в командах. Давайте назовем его Box:  
    name> Box
    
3. Rclone покажет огромный список поддерживаемых сервисов. Найдите в нем Box (обычно номер 8 или около того) и введите соответствующую цифру.  
    Storage> 8 (цифра может отличаться, смотрите по списку)
    
4. Box App Client Id: Просто нажмите **Enter**, чтобы оставить поле пустым.  
    client_id> (пусто)
    
5. Box App Client Secret: Снова просто **Enter**.  
    client_secret> (пусто)
    
6. Box config file: И еще раз **Enter**.  
    box_config_file> (пусто)
    
7. Edit advanced config?: Выберите n (нет).  
    y/n> n
    
8. Use auto config?: Это ключевой шаг.
    
    - **Если вы работаете за компьютером с графическим интерфейсом**, введите y (да). Rclone автоматически откроет браузер для входа в Box.
        
    - **Если вы настраиваете через SSH** или браузер не открылся, введите n (нет). Rclone даст вам ссылку, которую нужно скопировать и открыть в браузере на любом устройстве.  
        y/n> y
        
9. В открывшемся окне браузера войдите в свой аккаунт Box и предоставьте Rclone доступ, нажав кнопку "Grant access to Box".
    
10. После успешного входа вернитесь в терминал. Вы увидите сообщение Success!.
    
11. Configure this as a team drive?: Выберите n (нет), если это ваш личный аккаунт.  
    y/n> n
    
12. Rclone покажет итоговую конфигурацию. Если все верно, подтвердите:  
    y/e/d> y
    
13. Вы вернетесь в главное меню. Нажмите q, чтобы выйти из конфигуратора.  
    q) Quit config
    

**Поздравляю, настройка завершена!**

---

### Шаг 3: Монтирование Облака

Теперь "прикрепим" ваше облако к локальной папке.

1. Создайте пустую папку в вашей домашней директории. Она и будет точкой монтирования.
    
    Generated bash
    
    ```
    mkdir ~/Box
    ```
        
2. Выполните команду монтирования:
    
    Generated bash
    
    ```
    rclone mount Box: ~/Box --daemon
    ```
      
    - Box: — имя подключения, которое вы создали на Шаге 2 (двоеточие обязательно!).
        
    - ~/Box — папка, которую вы только что создали.
        
    - --daemon — флаг, который запускает процесс в фоновом режиме, освобождая ваш терминал.
        

Откройте файловый менеджер Thunar. Зайдите в папку Box в вашей домашней директории. Вы увидите все ваши облачные файлы и папки!

---

### Шаг 4: Автоматическое монтирование при входе (Рекомендуется)

Чтобы не вводить команду rclone mount каждый раз после перезагрузки, создадим службу systemd.

1. Создайте папку для пользовательских служб, если ее еще нет:
    
    Generated bash
    
    ```
    mkdir -p ~/.config/systemd/user/
    ```
      
2. Создайте файл службы с помощью текстового редактора (например, nano):
    
    Generated bash
    
    ```
    nano ~/.config/systemd/user/rclone-box.service
    ```
        
3. Вставьте в этот файл следующий текст. **Важно:** если вы назвали папку для монтирования не Box, измените пути в файле.
    
    Generated systemd
    
    ```
    [Unit]
    Description=Box Mount Service
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Type=forking
    ExecStart=/usr/bin/rclone mount Box: %h/Box --daemon \
        --vfs-cache-mode writes \
        --allow-other
    ExecStop=/usr/bin/fusermount -u %h/Box
    Restart=always
    RestartSec=10
    
    [Install]
    WantedBy=default.target
    ```
        
    - --vfs-cache-mode writes — важная опция для лучшей совместимости и производительности.
        
    - %h — это системный псевдоним для вашей домашней директории.
        
4. Сохраните файл (Ctrl+O, Enter) и выйдите из редактора (Ctrl+X).
    
5. **Включите и запустите службу.** Это нужно сделать один раз.
    
    Generated bash
    
    ```
    systemctl --user daemon-reload
    systemctl --user enable --now rclone-box.service
    ```
        

Теперь папка Box будет монтироваться автоматически при каждом входе в систему.

---

### Полезные команды

- **Проверить статус службы:**
    
    Generated bash
    
    ```
    systemctl --user status rclone-box.service
    ```
    
- **Отключить (размонтировать) папку вручную:**
    
    Generated bash
    
    ```
    fusermount -u ~/Box
    ```
    
- **Остановить автоматическую службу:**
    
    Generated bash
    
    ```
    systemctl --user stop rclone-box.service
    ```
