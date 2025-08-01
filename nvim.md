
# Шпаргалка по Neovim (Vim)

Эта заметка — моя база знаний по основным сочетаниям клавиш в Neovim. Все команды указаны для **Нормального режима** (Normal Mode), если не указано иное. Чтобы войти в него из любого другого режима, нажмите `Esc`.

---

## I. Основы: Навигация

| Сочетание | Действие | Примечание |
| :--- | :--- | :--- |
| `h` / `j` / `k` / `l` | Перемещение влево / вниз / вверх / вправо | Основа основ |
| `w` | Переместиться на начало следующего слова | `w`ord |
| `b` | Переместиться на начало предыдущего слова | `b`ackward |
| `e` | Переместиться в конец текущего слова | `e`nd |
| `0` | Перейти в самое начало строки | |
| `^` | Перейти к первому непробельному символу в строке | |
| `$` | Перейти в конец строки | |
| `gg` | Перейти в начало файла | |
| `G` | Перейти в конец файла | |
| `[N]G` | Перейти на строку с номером `N` (например, `15G`) | |
| `Ctrl + f` | Прокрутка на страницу (экран) вперед | `f`orward |
| `Ctrl + b` | Прокрутка на страницу (экран) назад | `b`ackward |

## II. Редактирование текста

### Вход в режим вставки (Insert Mode)

| Сочетание | Действие |
| :--- | :--- |
| `i` | Войти в режим вставки **до** курсора (`i`nsert) |
| `a` | Войти в режим вставки **после** курсора (`a`ppend) |
| `I` | Войти в режим вставки в начале строки |
| `A` | Войти в режим вставки в конце строки |
| `o` | Вставить новую строку **ниже** и войти в режим вставки |
| `O` | Вставить новую строку **выше** и войти в режим вставки |

### Изменение, копирование и удаление

Эти команды часто используются с командами навигации (например, `dw` - удалить слово).

| Сочетание  | Действие                                              | Примечание                                               |
| :--------- | :---------------------------------------------------- | :------------------------------------------------------- |
| `x`        | Удалить символ под курсором                           | (backspace)                                              |
| `d`        | Удалить (вырезать) (`d`elete).                        | Например, `dd` - удалить строку, `dw` - удалить слово.   |
| `c`        | Изменить (удалить и войти в режим вставки) (`c`hange) | Например, `cc` - изменить строку, `cw` - изменить слово. |
| `y`        | Скопировать (`y`ank)                                  | Например, `yy` - скопировать строку.                     |
| `p`        | Вставить **после** курсора (`p`aste)                  | (р)                                                      |
| `P`        | Вставить **до** курсора                               | (shift+p)                                                |
| `u`        | Отменить последнее действие (`u`ndo)                  |                                                          |
| `Ctrl + r` | Вернуть отмененное действие (`r`edo)                  |                                                          |
| `.`        | **Повторить последнее изменение**                     | Один из самых мощных операторов!                         |
| `r`        | Заменить один символ под курсором (`r`eplace)         |                                                          |

## III. Визуальный режим (Выделение)

Сначала войдите в режим, затем используйте навигацию для выделения. После выделения можно применить оператор (`d`, `y`, `c`).

| Сочетание | Действие |
| :--- | :--- |
| `v` | Войти в визуальный режим (посимвольное выделение) |
| `V` | Войти в визуальный режим (построчное выделение) |
| `Ctrl + v` | Войти в блочный визуальный режим (выделение прямоугольником) |

*Пример: Нажмите `v`, переместите курсор с помощью `j` и `l`, чтобы выделить текст, затем нажмите `d`, чтобы удалить выделенное.*

## IV. Поиск и замена

| Сочетание | Действие |
| :--- | :--- |
| `/` | Начать поиск вперед по тексту |
| `?` | Начать поиск назад по тексту |
| `n` | Перейти к следующему совпадению в том же направлении |
| `N` | Перейти к следующему совпадению в обратном направлении |
| `:%s/найти/заменить/g` | Заменить все вхождения в файле |
| `:%s/найти/заменить/gc` | Заменить все вхождения с подтверждением каждого случая |

## V. Работа с окнами и файлами

| Сочетание | Действие |
| :--- | :--- |
| `:w` | Сохранить файл (`w`rite) |
| `:q` | Закрыть окно/Neovim (`q`uit) |
| `:wq` | Сохранить и закрыть |
| `:q!` | Закрыть без сохранения |
| `:e [путь]` | Открыть файл для редактирования (`e`dit) |
| `:ls` | Показать список открытых файлов (буферов) |
| `:b[N]` | Перейти к буферу с номером `N` (например, `:b2`) |
| `Ctrl+w s` | Разделить окно горизонтально (`s`plit) |
| `Ctrl+w v` | Разделить окно вертикально (`v`ertical split) |
| `Ctrl+w` + `h/j/k/l` | Перемещение между разделенными окнами |
| `:tabe [путь]` | Открыть файл в новой вкладке (`tab` `e`dit) |
| `gt` | Перейти на следующую вкладку |
| `gT` | Перейти на предыдущую вкладку |

---

### Pro Tip

Если вы только начинаете, запустите в терминале команду `vimtutor`. Это лучший интерактивный курс по основам, который займет у вас 30-60 минут и даст прочную базу.