
````
# Шпаргалка: Создание удаленных Git-репозиториев из терминала

Этот документ — краткое руководство по созданию **приватных** и **публичных** репозиториев на популярных хостингах с помощью утилит командной строки.

---

## 1. GitHub

**Требование:** Установленный [**GitHub CLI (gh)**](https://cli.github.com/).
Для входа в аккаунт используется команда: `gh auth login`.

### Создать приватный репозиторий
```bash
# Замените РЕПО_ИМЯ на нужное вам
gh repo create РЕПО_ИМЯ --private
````


### Создать публичный репозиторий

```
# Замените РЕПО_ИМЯ на нужное вам
gh repo create РЕПО_ИМЯ --public
```


### Полезный флаг: создать и сразу отправить существующий код

Если вы уже находитесь в папке с Git-проектом, эта команда создаст репозиторий на GitHub, привяжет его как origin и отправит (push) ваш код одной строкой.


```
# Для приватного репозитория
gh repo create РЕПО_ИМЯ --private --source=. --push

# Для публичного репозитория
gh repo create РЕПО_ИМЯ --public --source=. --push
```


---

## 2. GitLab

**Требование:** Установленный [**GitLab CLI (glab)**](https://www.google.com/url?sa=E&q=https%3A%2F%2Fgitlab.com%2Fgitlab-org%2Fcli).  
Для входа в аккаунт используется команда: glab auth login.

### Создать приватный репозиторий


```
# Замените РЕПО_ИМЯ на нужное вам
glab repo create РЕПО_ИМЯ --private
```


### Создать публичный репозиторий


```
# Замените РЕПО_ИМЯ на нужное вам
glab repo create РЕПО_ИМЯ --public
```


### Полезный флаг: создать и сразу отправить существующий код

Аналогично GitHub, можно создать, привязать и отправить код одной командой.


````
# Для приватного репозитория
glab repo create РЕПО_ИМЯ --private --source=. --push

# Для публичного репозитория
glab repo create РЕПО_ИМЯ --public --source=. --push```

---

## 3. Bitbucket

**Требование:** Установленный `curl`. Официального CLI нет, используется API.

**Подготовка:**
1.  Зайдите в Bitbucket: `Settings` -> `App passwords` -> `Create app password`.
2.  Дайте паролю права на `Repositories: Write`.
3.  **Обязательно скопируйте сгенерированный пароль**. Вы больше не сможете его увидеть.

Для удобства задайте переменные в терминале:
```bash
# Замените значения на ваши
BITBUCKET_USERNAME="ваш_логин_bitbucket"
BITBUCKET_APP_PASSWORD="ваш_пароль_приложения"
REPO_NAME="имя_вашего_репозитория"```

### Создать приватный репозиторий
```bash
curl --request POST \
  --url https://api.bitbucket.org/2.0/repositories/${BITBUCKET_USERNAME}/${REPO_NAME} \
  --user "${BITBUCKET_USERNAME}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Content-Type: application/json' \
  --data '{"is_private": true}'
````


### Создать публичный репозиторий


```
curl --request POST \
  --url https://api.bitbucket.org/2.0/repositories/${BITBUCKET_USERNAME}/${REPO_NAME} \
  --user "${BITBUCKET_USERNAME}:${BITBUCKET_APP_PASSWORD}" \
  --header 'Content-Type: application/json' \
  --data '{"is_private": false}'
```


После создания репозитория через curl, не забудьте вручную привязать его к локальному проекту и отправить код:


```
git remote add origin https://${BITBUCKET_USERNAME}@bitbucket.org/${BITBUCKET_USERNAME}/${REPO_NAME}.git
git push -u origin master # или main
```