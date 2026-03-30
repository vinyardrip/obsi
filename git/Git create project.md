
# 📑 Шпаргалка: Создание удаленных репозиториев (CLI)

Этот документ описывает, как правильно создавать репозитории и, главное, **связывать** их с локальной папкой, чтобы избежать ошибок «оторванного» облака.

## 1. GitHub (утилита `gh`)

_Авторизация:_ `gh auth login`

### Вариант А: Создание «с нуля» (Рекомендуемый)

Если у тебя есть папка с файлами, и ты хочешь превратить её в репо на GitHub одной командой:

Bash

```
# 1. Зайти в папку
# 2. Инициализировать гит и сделать ПЕРВЫЙ коммит (обязательно, иначе push не сработает)
git init
git branch -m main   
git add .
git commit -m "initial commit"

# 3. Создать репо в облаке и сразу связать/отправить
gh repo create ИМЯ_РЕПО --private --source=. --remote=origin --push
```

### Вариант Б: Если репо на GitHub уже создан («Ручной мост»)

Если ты сначала создал репо командой `gh repo create ИМЯ_РЕПО --private` БЕЗ флагов, выполни это:

Bash

```
git init
git remote add origin https://github.com/ПОЛЬЗОВАТЕЛЬ/ИМЯ_РЕПО.git
git add .
git commit -m "initial commit"
git push -u origin main
```

---

## 2. GitLab (утилита `glab`)

_Авторизация:_ `glab auth login`

Bash

```
# Создать и привязать текущую папку
glab repo create ИМЯ_РЕПО --private --source=. --push
```

---

## 3. Bitbucket (через API `curl`)

У Bitbucket нет официального CLI, поэтому связь всегда настраивается вручную.

**Шаг 1: Создать «коробку» в облаке:**

Bash

```
curl --request POST \
  --url https://api.bitbucket.org/2.0/repositories/ЛОГИН/ИМЯ_РЕПО \
  --user "ЛОГИН:ПАРОЛЬ_ПРИЛОЖЕНИЯ" \
  --header 'Content-Type: application/json' \
  --data '{"is_private": true}'
```

**Шаг 2: Привязать локальную папку:**

Bash

```
git init
git remote add origin https://ЛОГИН@bitbucket.org/ЛОГИН/ИМЯ_РЕПО.git
git add .
git commit -m "initial commit"
git push -u origin main
```