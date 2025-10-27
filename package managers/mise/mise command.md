
# Шпаргалка: Сравнение команд Mise и ASDF (v0.16+)

Этот документ содержит справочную информацию и сравнение команд `mise` и актуальной версии `asdf` (написанной на Go).

---

## Mise: Ключевые команды

`mise` отличается лаконичным синтаксисом и автоматизацией многих действий, таких как установка плагинов.

| Задача | Команда `mise` | Псевдонимы | Примечание |
| :--- | :--- | :--- | :--- |
| Показать инструменты | `mise current` / `mise ls` | `mi ls` | `ls` показывает статус всех инструментов. |
| Установить всё из `.tool-versions` | `mise install` | `mi i` | Можно настроить на авто-установку. |
| Установить конкретную версию | `mise install node@20.10.0` | `mi i node@latest`| Используется синтаксис `name@version`. |
| Установить глобальную версию | `mise global node@20` | `mi g node@20` | Изменяет `~/.config/mise/config.toml`. |
| Установить локальную версию | `mise use node@20` | `mi use node@20` | Создает/изменяет `./.tool-versions`. |
| Удалить версию | `mise uninstall node@20` | `mi un node@20` | |
| Показать путь к исполняемому файлу | `mise which node` | - | Работает как системная команда `which`. |
| Список установленных плагинов | `mise plugins ls` | `mi p ls` | |
| Обновить плагины | `mise plugins update` | `mi p up` | |


---

## Сравнительная таблица (Mise vs ASDF v0.16+)

| Задача | `mise` (Предпочтительный) | `asdf` (Новый синтаксис) |
| :--- | :--- | :--- |
| Установить локальную версию | `mise use node@20` | `asdf set nodejs 20` |
| Установить глобальную версию | `mise global node@20` | `asdf set --home nodejs 20` |
| Установка из `.tool-versions` | `mise install` | `asdf install` |
| Добавление плагина | **(Автоматически)** | `asdf plugin add nodejs` |
| Показать активные версии | `mise current` | `asdf current` |
| Показать путь к шиму | `mise which node` | `asdf which node` |
| Обновить все плагины | `mise plugins update` | `asdf plugin update --all` |