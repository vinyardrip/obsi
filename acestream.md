

---
### MOC: [[Linux]]

### Tags: #linux #arch #endeavouros #acestream #aur #systemd #mpv #how-to

# Установка Ace Stream на Arch / EndeavourOS (полная инструкция)

Это полная инструкция по установке и настройке Ace Stream на Arch Linux и его производных (EndeavourOS, Manjaro) через AUR. Метод исправляет распространенную проблему, когда **движок (acestream-engine) устанавливается отдельно от лаунчера (acestream-launcher)**, что приводит к неработоспособности ссылок.

Этот способ использует системные библиотеки, занимает меньше места и лучше интегрируется в систему, чем Snap-версия.

## 1. Установка: Движок, Лаунчер и Плеер

Ключевой момент — установить **три** пакета:

1. acestream-engine — основной движок, который работает как фоновая служба.
    
2. acestream-launcher — скрипт-помощник, который принимает ссылки acestream:// и передает их движку. **Без него интеграция с браузером не работает!**
    
3. mpv — легковесный и рекомендуемый видеоплеер.
    

codeBash

```
# Устанавливаем все три компонента одной командой из AUR и официальных репозиториев
yay -S acestream-engine acestream-launcher mpv
```

> **Важно:** Раньше acestream-launcher был частью пакета acestream-engine, но позже был вынесен в отдельный пакет. Это основная причина, почему старые инструкции могут не работать.

## 2. Настройка системной службы (Systemd)

Движок должен работать в фоне постоянно. Для этого нужно запустить его службу и включить ее для автозагрузки.

codeBash

```
# Запустить службу немедленно
sudo systemctl start acestream-engine.service

# Включить автозапуск службы при загрузке системы
sudo systemctl enable acestream-engine.service

# Проверить статус (убедиться, что горит зеленым active (running))
systemctl status acestream-engine.service
```

## 3. Интеграция с браузером (запуск по клику)

Чтобы ссылки acestream:// в браузере (Chrome, Firefox) открывались автоматически, создадим и зарегистрируем специальный .desktop файл.

#### A. Создание файла-обработчика

codeBash

```
# Можно использовать любой текстовый редактор (nano, micro, vim)
micro ~/.local/share/applications/acestream-launcher.desktop
```

#### B. Содержимое файла

Скопируйте и вставьте в редактор следующий текст. Он указывает системе, что для acestream-ссылок нужно вызывать acestream-launcher с плеером mpv.

codeIni

```
[Desktop Entry]
Type=Application
Name=Ace Stream Launcher
Comment=Launch Ace Stream links in MPV Player
Exec=acestream-launcher -p mpv %u
Terminal=false
MimeType=x-scheme-handler/acestream;
Categories=AudioVideo;Player;
```

Сохраните файл (Ctrl+S) и выйдите (Ctrl+Q).

#### C. Регистрация обработчика в системе

Выполните эти две команды, чтобы система узнала о новом обработчике.

codeBash

```
# Установить наш файл как обработчик по умолчанию
xdg-mime default acestream-launcher.desktop x-scheme-handler/acestream

# Обновить базу данных .desktop файлов
update-desktop-database ~/.local/share/applications/
```

После этого **полностью перезапустите браузер**, чтобы он подхватил новые настройки.

---

## Шпаргалка и Диагностика

Если что-то пошло не так, вот команды для проверки.

### Полезные команды

codeBash

```
# Проверить статус службы движка
systemctl status acestream-engine.service

# Остановить/запустить/перезапустить службу
sudo systemctl stop acestream-engine.service
sudo systemctl start acestream-engine.service
sudo systemctl restart acestream-engine.service

# Протестировать запуск ссылки из терминала (минуя браузер)
acestream-launcher -p mpv "acestream://айди_трансляции"
```

### Порядок диагностики неисправностей

1. **Проверить, что движок запущен:** systemctl status acestream-engine.service. Если не active (running), запустить вручную.
    
2. **Проверить, что лаунчер установлен:** Просто введите в терминале acestream-launcher. Если ошибка command not found, значит, вы пропустили его установку. Вернитесь к шагу 1.
    
3. **Проверить, что обработчик xdg настроен:**
    
    codeBash
    
    ```
    xdg-mime query default x-scheme-handler/acestream
    ```
    
    Правильный ответ: acestream-launcher.desktop
    
4. **Если все работает в терминале, но не в браузере:** Вероятно, браузер "запомнил" неправильное действие. Для Chrome/Brave нужно отредактировать файл ~/.config/google-chrome/Local State, найти секцию protocol_handler и удалить оттуда запись для "acestream".