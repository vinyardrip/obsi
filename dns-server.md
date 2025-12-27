

##  Резервные DoH-серверы (на случай, если xbox-dns.ru ляжет)  `https://xbox-dns.ru/dns-query`

### 1. Быстрая замена  
- `https://dns.google/dns-query` → 8.8.8.8
- `https://cloudflare-dns.com/dns-query` → 1.1.1.1
- `https://dns.quad9.net/dns-query` → 9.9.9.9
- `https://dns.mullvad.net/dns-query` (без подписки, no-logs)
-  `https://dns.malw.link/dns-query`

---

### 2. Настройки по системам

####  Chrome / Chromium / Brave (Linux, Windows, macOS)
1. Открыть `chrome://settings/security`.
2. Включить «**Использовать защищённый DNS**».
3. Выбрать «**Собственный**» и вставить любой URL из списка выше.
4. Добавить их через запятую в расширении «Secure DNS» для быстрого переключения.

####  Firefox
1. `about:config` → `network.trr.uri` = `https://dns.google/dns-query`.
2. `network.trr.mode` = `2` (fallback) или `3` (только DoH).
3. Дополнительно: `network.trr.bootstrapAddress = 8.8.8.8`.

####  Arch Linux (systemd-resolved)
1. Установить пакет `systemd-resolved` (обычно уже есть).
2. Открыть `/etc/systemd/resolved.conf`:
   ```ini
   [Resolve]
   DNS=1.1.1.1 8.8.8.8 9.9.9.9
   DNSOverTLS=yes
   FallbackDNS=1.0.0.1 8.8.4.4
```


3. Перезапустить:
   ```bash
   sudo systemctl restart systemd-resolved

```
   
####  Arch Linux (NetworkManager)

- GUI: Правая кнопка по подключению → Параметры → IPv4/IPv6 → DNS →  
    `1.1.1.1,8.8.8.9.9.9.9` → «Только автоматические (DHCP) адреса» → сохранить.
    
- CLI (nmcli):
    
    bash
    
    Copy
    
    ```bash
    nmcli connection modify <SSID> ipv4.dns "1.1.1.1 8.8.8.8"
    nmcli connection modify <SSID> ipv4.ignore-auto-dns yes
    nmcli connection up <SSID>
    ```
    

####  Router (OpenWrt / AdGuard Home)

- Upstream DoH:
    
    Copy
    
    ```
    https://dns.google/dns-query
    https://cloudflare-dns.com/dns-query
    https://dns.quad9.net/dns-query
    ```
    
- Включить «Bootstrap DNS» = `1.1.1.1` и «Parallel requests».
    

---

### 3. Проверка

bash

Copy

```bash
# Linux
dig @1.1.1.1 google.com +short
# или
resolvectl query google.com
```

В браузере: [https://1.1.1.1/help](https://1.1.1.1/help) → DoH = ✅.

---

### 4. Экстренный fallback

Если DoH совсем не открывается, временно прописать UDP 53:

bash

Copy

```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

или в NetworkManager → «Автоматический (DHCP) адреса только» → ручной DNS.