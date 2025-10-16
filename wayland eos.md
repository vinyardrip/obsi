

# Настройка приложений для Wayland

Эта заметка — моя персональная база знаний для решения типичных проблем с приложениями при работе в сеансе Wayland, как на GNOME, так и на Hyprland.

## Проблема 1: Не печатаются буквы в Electron-приложениях (GNOME)

**Симптомы:**  
В приложениях типа VSCode, Obsidian, Discord и др. не работает ввод с клавиатуры (особенно с не-английскими раскладками).

**Причина:**  
Некорректная работа модуля ввода (IME) в Electron-приложениях под Wayland. Приложение нужно "заставить" использовать правильный модуль ibus.

**Решение:**  
Нужно отредактировать .desktop файл-ярлык приложения, создав его локальную копию и добавив специальные флаги запуска.

### Пошаговая инструкция

1. **Скопировать системный ярлык в домашнюю папку.** Это "правильный" способ, чтобы изменения не затерлись при обновлении.
    
    codeBash
    
    ```
    # Для VSCode
    cp /usr/share/applications/code.desktop ~/.local/share/applications/
    
    # Для Obsidian
    cp /usr/share/applications/obsidian.desktop ~/.local/share/applications/
    ```
    
2. **Отредактировать скопированный файл.**
    
    codeBash
    
    ```
    # Для VSCode
    nano ~/.local/share/applications/code.desktop
    
    # Для Obsidian
    nano ~/.local/share/applications/obsidian.desktop
    ```
    
3. **Изменить строку Exec=.**
    
    - **Для VSCode:**
        
        - **Было:** Exec=/usr/bin/code --new-window %F
            
        - **Стало:** Exec=env GTK_IM_MODULE=ibus /usr/bin/code --enable-features=UseOzonePlatform --ozone-platform=wayland --new-window %F
            
    - **Для Obsidian:**
        
        - **Было:** Exec=/opt/Obsidian/obsidian %U
            
        - **Стало:** Exec=env GTK_IM_MODULE=ibus /opt/Obsidian/obsidian --enable-features=UseOzonePlatform --ozone-platform=wayland %U
            
4. **(Критически важно!) Обновить кэш приложений.** Без этого система не увидит изменения в локальном файле.
    
    codeBash
    
    ```
    update-desktop-database ~/.local/share/applications
    ```