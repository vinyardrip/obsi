
# Шпаргалка: Куда вставлять SSH-ключи

> [!TIP]  
> Эта заметка содержит прямые ссылки на страницы добавления SSH-ключей для основных Git-сервисов, чтобы не искать их каждый раз.

---

## Прямые ссылки и пути

## GitHub

- **Прямая ссылка:**  
    [https://github.com/settings/keys](https://www.google.com/url?sa=E&q=https%3A%2F%2Fgithub.com%2Fsettings%2Fkeys)
    
- **Путь в интерфейсе:**  
    Иконка профиля (вверху справа) → Settings → SSH and GPG keys
    

## GitLab

- **Прямая ссылка:**  
    [https://gitlab.com/-/profile/keys](https://www.google.com/url?sa=E&q=https%3A%2F%2Fgitlab.com%2F-%2Fprofile%2Fkeys)
    
- **Путь в интерфейсе:**  
    Иконка профиля (вверху справа) → Preferences → SSH Keys
    

## Bitbucket

- **Прямая ссылка:**  
    [https://bitbucket.org/account/settings/ssh-keys/](https://www.google.com/url?sa=E&q=https%3A%2F%2Fbitbucket.org%2Faccount%2Fsettings%2Fssh-keys%2F)
    
- **Путь в интерфейсе:**  
    Иконка профиля (внизу слева) → Personal settings → SSH keys
    
    > [!INFO]  
    > В Bitbucket поле для названия ключа называется **Label**, а не **Title**, но суть та же.
    

---

## Полный рабочий процесс (чтобы не забыть)

> [!NOTE] Краткая инструкция от А до Я

1. **Создать ключ на компьютере.** Открыть терминал и выполнить команду, подставив свои данные.
    
    Generated bash
    
    ```
    # Формат: ssh-keygen -t ed25519 -f ~/.ssh/имя_файла -C "пользователь@машина сервис"
    ssh-keygen -t ed25519 -f ~/.ssh/github_key -C "erip@erip-rog github"
    ```
        
2. **Скопировать содержимое ПУБЛИЧНОГО ключа.** Вывести ключ в терминал и скопировать всю строку.
    
    Generated bash
    
    ```
    # Важно: копируем файл с расширением .pub!
    cat ~/.ssh/github_key.pub
    ```
        
3. **Перейти по нужной ссылке** из списка выше.
    
4. **Нажать "New SSH key"** (или "Add key").
    
5. **Вставить скопированный ключ** в большое текстовое поле "Key".
    
6. **Убедиться, что поле "Title" / "Label" заполнилось автоматически** из вашего комментария.
    
7. **Нажать "Add SSH key"** для сохранения. Готово