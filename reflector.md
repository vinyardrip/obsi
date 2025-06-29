
```
alias up_reflector='sudo reflector --verbose --country 'Belarus,Russia,Poland,Germany,Ukraine' --protocol https --age 12 --sort rate --latest 20 --save /etc/pacman.d/mirrorlist && echo "--- mirrors update! ---"'
```

```
sudo pacman -Syyu
```

```
sudo micro /etc/systemd/system/reflector.service
```

```
[Unit]
Description=Pacman mirrorlist update
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/reflector --verbose --country 'Belarus,Russia,Poland,Germany,Ukraine' --protocol https --age 12 --sort rate --latest 20 --save /etc/pacman.d/mirrorlist

[Install]
WantedBy=multi-user.target
```

```
sudo micro /etc/systemd/system/reflector.timer
```

```
[Unit]
Description=Run reflector weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

```
sudo systemctl enable reflector.timer
sudo systemctl start reflector.timer
```

