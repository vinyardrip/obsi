# 🎧 Настройка звука в Linux (EasyEffects)

Для работы всех модулей EasyEffects в Arch Linux необходимо установить наборы плагинов.

| Эффект в EasyEffects | Пакет в Arch (pacman/yay) | Описание |
| :--- | :--- | :--- |
| **Эквалайзер / Компрессор** | `lsp-plugins-lv2` | Основной движок для EQ и динамики |
| **Лимитер / Максимайзер** | `zam-plugins-lv2` | Защита от перегрузок (клиппинга) |
| **Шумоподавление (Mic)** | `rnnoise` | AI-подавление фонового шума |
| **Multiband Compressor** | `calf` | Раздельная компрессия частот |
| **Exciter / Bass Enhancer** | `calf` | Улучшение четкости и баса |
| **Пресеты сообщества** | `easyeffects-presets-git` | Готовые конфиги из AUR |

## 🛠 Быстрая установка
```bash
sudo pacman -S lsp-plugins-lv2 calf zam-plugins-lv2 mda.lv2 rnnoise libebur128
```



