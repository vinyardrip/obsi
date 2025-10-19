
Установка Ace Stream на Arch / EndeavourOS (без Snap)
#linux #arch #endeavouros #acestream #aur #systemd #mpv
Инструкция по установке и настройке Ace Stream через AUR. Этот метод использует системные библиотеки, занимает значительно меньше места и лучше интегрируется в систему, чем Snap-версия.
1. Установка движка и плеера
Устанавливаем движок acestream-engine из AUR и рекомендуемый плеер mpv из официальных репозиториев.
code
Bash
# Установка движка из AUR
yay -S acestream-engine

# Установка плеера MPV (если его еще нет)
sudo pacman -S mpv
2. Настройка системной службы (Systemd)
Движок работает как фоновая системная служба. Её нужно запустить и включить для автозагрузки.
Важно: Это системная служба, поэтому все команды выполняются с sudo и без флага --user.
code
Bash
# Запустить службу немедленно
sudo systemctl start acestream-engine.service

# Включить автозапуск службы при загрузке системы
sudo systemctl enable acestream-engine.service

# Проверить статус службы (убедиться, что она active (running))
systemctl status acestream-engine.service
3. Интеграция с браузером (запуск по клику на ссылку)
Чтобы ссылки acestream:// открывались автоматически в плеере при клике в браузере (Chrome, Firefox и т.д.), нужно создать .desktop файл и зарегистрировать его в системе.
A. Создание .desktop файла
Используем текстовый редактор micro для создания файла-обработчика.
code
Bash
micro ~/.local/share/applications/acestream-launcher.desktop
B. Содержимое файла
Вставьте в редактор micro следующий текст. Этот файл указывает системе запускать acestream-launcher с плеером mpv.
code
Ini
[Desktop Entry]
Type=Application
Name=Ace Stream Launcher
Comment=Launch Ace Stream links in MPV Player
Exec=acestream-launcher -p mpv %u
Terminal=false
MimeType=x-scheme-handler/acestream;
Categories=AudioVideo;Player;
Сохраните файл (Ctrl+S) и выйдите из micro (Ctrl+Q).
C. Регистрация обработчика
Выполните эти две команды, чтобы система узнала о новом обработчике.
code
Bash
# Установить наш файл как обработчик по умолчанию для acestream
xdg-mime default acestream-launcher.desktop x-scheme-handler/acestream

# Обновить базу данных приложений
update-desktop-database ~/.local/share/applications/
После этого полностью перезапустите браузер, чтобы он подхватил новые настройки.
4. Шпаргалка: полезные команды
code
Bash
# Запустить трансляцию из терминала
acestream-launcher -p mpv acestream://...ссылка...

# Проверить статус службы
systemctl status acestream-engine.service

# Остановить службу
sudo systemctl stop acestream-engine.service

# Запустить службу вручную (если была остановлена)
sudo systemctl start acestream-engine.service