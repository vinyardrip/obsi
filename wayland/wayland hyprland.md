
# Hyprland: Лечение морганий и тормозов в Electron и Chrome

**Симптомы:**  
Приложения, особенно на движке Electron (VSCode, Obsidian, Element) и Chrome, моргают при изменении размера, переключении рабочих столов или скроллинге. Ощущаются графические артефакты и общая "заторможенность" интерфейса.

**Причина:**  
По умолчанию эти приложения запускаются в Hyprland через слой совместимости **XWayland**. Hyprland — это очень быстрый Wayland-композитор, и слой совместимости XWayland часто становится "бутылочным горлышком", вызывая проблемы с синхронизацией и рендерингом.

**Решение:**  
Принудительно запустить эти приложения в **нативном (родном) режиме Wayland**. Это заставит их напрямую взаимодействовать с Hyprland, что устранит моргание и улучшит производительность.

---

### Для VSCode, Obsidian, Element (Electron-приложения)

#### Шаг 1: Находим и копируем .desktop файл

Сначала найдем оригинальный ярлык приложения и создадим его локальную копию, чтобы наши изменения не затерлись при обновлении.

codeBash

```
# Ищем оригинальный файл (замените 'obsidian' на 'code', 'element-desktop' и т.д.)
# Сначала смотрим в /usr/share/applications/
ls /usr/share/applications/obsidian.desktop

# Копируем найденный файл в домашнюю папку
cp /usr/share/applications/obsidian.desktop ~/.local/share/applications/
```

#### Шаг 2: Открываем и редактируем скопированный файл

codeBash

```
# Открываем локальную копию в редакторе
nano ~/.local/share/applications/obsidian.desktop
```

#### Шаг 3: Заменяем строку Exec=

Это самый важный шаг. Нам нужно добавить флаги --enable-features=UseOzonePlatform --ozone-platform=wayland.

- **Для Obsidian:**
    
    - **Было:** Exec=/opt/Obsidian/obsidian %U
        
    - **Стало:** Exec=/opt/Obsidian/obsidian --enable-features=UseOzonePlatform --ozone-platform=wayland %U
        
- **Для VSCode:**
    
    - **Было:** Exec=/usr/bin/code %F
        
    - **Стало:** Exec=/usr/bin/code --enable-features=UseOzonePlatform --ozone-platform=wayland %F
        

Сохраните файл и закройте редактор (Ctrl+X, Y, Enter).

#### Шаг 4: Обновляем кэш приложений

Эта команда заставит систему немедленно увидеть наши изменения.

codeBash

```
update-desktop-database ~/.local/share/applications
```

---

### Для Google Chrome / Chromium

Процесс абсолютно идентичен.

#### Шаг 1: Находим и копируем .desktop файл

codeBash

```
# Ищем ярлык для Chrome
ls /usr/share/applications/google-chrome.desktop

# Копируем его
cp /usr/share/applications/google-chrome.desktop ~/.local/share/applications/
```

#### Шаг 2: Открываем и редактируем скопированный файл

codeBash

```
nano ~/.local/share/applications/google-chrome.desktop
```

#### Шаг 3: Заменяем строку Exec=

Флаги те же самые.

- **Было:** Exec=/usr/bin/google-chrome-stable %U
    
- **Стало:** Exec=/usr/bin/google-chrome-stable --enable-features=UseOzonePlatform --ozone-platform=wayland %U
    

Сохраните и закройте редактор.

#### Шаг 4: Обновляем кэш приложений

codeBash

```
update-desktop-database ~/.local/share/applications
```

#### Как проверить результат?

После запуска Chrome откройте в браузере адрес chrome://gpu. Найдите поле **Ozone platform**. Там должно быть написано **Wayland**. Если написано X11, значит флаги не применились.