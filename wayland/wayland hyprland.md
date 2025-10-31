## Hyprland: Лечение морганий и тормозов в Electron и Chrome

**Симптомы:**  
Приложения, особенно на движке Electron (VSCode, Obsidian, Element) и Chrome, моргают при изменении размера, переключении рабочих столов или скроллинге. Ощущаются графические артефакты и общая «заторможенность» интерфейса.

**Причина:**  
По умолчанию эти приложения запускаются в Hyprland через слой совместимости **XWayland**. Hyprland — это очень быстрый Wayland-композитор, и слой совместимости XWayland часто становится «бутылочным горлышком», вызывая проблемы с синхронизацией и рендерингом.

**Решение:**  
Принудительно запустить эти приложения в **нативном (родном) режиме Wayland**. Это заставит их напрямую взаимодействовать с Hyprland, что устранит моргание и улучшит производительность.

---

### Для VSCode, Obsidian, Element, Figma (Electron-приложения)

#### Шаг 1: Находим и копируем .desktop-файл

bash

Copy

```bash
# Ищем оригинальный файл (замените имя при необходимости)
ls /usr/share/applications/obsidian.desktop
ls /usr/share/applications/io.element.Element.desktop
ls /usr/share/applications/code-insiders.desktop
ls /usr/share/applications/figma-linux.desktop   # <-- для Figma

# Копируем найденный файл в домашнюю папку
cp /usr/share/applications/obsidian.desktop ~/.local/share/applications/
cp /usr/share/applications/io.element.Element.desktop ~/.local/share/applications/
cp /usr/share/applications/code-insiders.desktop ~/.local/share/applications/
cp /usr/share/applications/figma-linux.desktop ~/.local/share/applications/   # <-- для Figma
```

#### Шаг 2: Открываем и редактируем скопированный файл

bash

Copy

```bash
# Открываем локальную копию в редакторе
micro ~/.local/share/applications/obsidian.desktop
micro ~/.local/share/applications/figma-linux.desktop   # пример для Figma
```

#### Шаг 3: Заменяем строку Exec=

Добавляем флаги `--enable-features=UseOzonePlatform --ozone-platform=wayland` **перед** последним параметром (`%U`, `%F`, `%f`, `%u`).

- **Для Obsidian:**  
    Было:  
    `Exec=/usr/bin/obsidian %U`  
    Стало:  
    `Exec=/usr/bin/obsidian --enable-features=UseOzonePlatform --ozone-platform=wayland %U`
    
- **Для VSCode / Code-Insiders:**  
    Было:  
    `Exec=/usr/bin/code-insiders %F`  
    Стало:  
    `Exec=/usr/bin/code-insiders --enable-features=UseOzonePlatform --ozone-platform=wayland %F`
    
- **Для Figma (figma-linux, AUR/deb/rpm):**  
    Было:  
    `Exec=/opt/figma-linux/figma-linux --no-sandbox ...другие-флаги... %U`  
    Стало:  
    `Exec=/opt/figma-linux/figma-linux --no-sandbox ...другие-флаги... --enable-features=UseOzonePlatform --ozone-platform=wayland %U`
    

Сохраните файл и закройте редактор.

#### Шаг 4: Обновляем кэш приложений

bash

Copy

```bash
update-desktop-database ~/.local/share/applications
```

---

### Для Google Chrome / Chromium

Процесс абсолютно идентичен.

#### Шаг 1: Находим и копируем .desktop-файл

bash

Copy

```bash
# Ищем ярлык для Chrome
ls /usr/share/applications/google-chrome.desktop

# Копируем его
cp /usr/share/applications/google-chrome.desktop ~/.local/share/applications/
```

#### Шаг 2: Открываем и редактируем скопированный файл

bash

Copy

```bash
micro ~/.local/share/applications/google-chrome.desktop
```

#### Шаг 3: Заменяем строку Exec=

Флаги те же самые.

- Было:  
    `Exec=/usr/bin/google-chrome-stable %U`
    
- Стало:  
    `Exec=/usr/bin/google-chrome-stable --enable-features=UseOzonePlatform --ozone-platform=wayland %U`
    

Сохраните и закройте редактор.

#### Шаг 4: Обновляем кэш приложений

bash

Copy

```bash
update-desktop-database ~/.local/share/applications
```

#### Как проверить результат?

После запуска Chrome откройте в браузере адрес `chrome://gpu`. Найдите поле **Ozone platform**. Там должно быть написано **Wayland**. Если написано **X11**, значит флаги не применились.

---

Готово — все перечисленные приложения теперь запускаются в нативном Wayland-режиме и больше не моргают в Hyprland.