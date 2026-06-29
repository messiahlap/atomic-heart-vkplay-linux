# Atomic Heart (VK Play / GameCenter) на Linux — Lutris + UMU + Proton

[English](README.md) | **Русский**

Как запустить **версию VK Play (VKGC / GameCenter)** игры **Atomic Heart** на Linux через
**Lutris + umu-run + Proton**: чиним мгновенный краш на старте, сломанное видео и рендер в
720p/мыло, получаем нативный 4K.

> **Кратко:** Игра падает при запуске, потому что бэкенд Media Foundation в Proton
> (**winedmo**, а также **winegstreamer**) не может проиграть стартовый ролик
> `Launch_FHD_60FPS_PC_Steam.mp4`. **Переименуйте/уберите файлы `Launch_*.mp4`** и форсируйте
> GStreamer-бэкенд через `PROTON_MEDIA_USE_GST=1`. Затем поправьте разрешение в
> `GameUserSettings.ini`. HDR в этой сборке игры **не реализован**. Подробности ниже.

---

## Окружение, на котором проверено

| | |
|---|---|
| ОС | Bazzite (Fedora atomic), ядро 7.0.x |
| DE / сессия | KDE Plasma 6.6.5, kwin 6.6.5, **Wayland** |
| GPU / драйвер | NVIDIA RTX 4090, проприетарный драйвер 610.43.02 |
| CPU | Ryzen 7 7800X3D |
| Лаунчер | Lutris 0.5.22, runner `umu-run` (umu-launcher 1.4.0) |
| Proton | GE-Proton10-34 |
| Игра | Atomic Heart, **версия VK Play** (через GameCenter.exe), UE4 4.27.2 |

Запуск идёт по цепочке: **Lutris → umu-run → GameCenter.exe (VKGC) → AtomicHeart-Win64-Shipping.exe**.

---

## Симптом 1 — мгновенный краш на старте

Игра падает сразу при запуске (`SecondsSinceStart=0`), крэш-дамп вида:

```
C:\users\steamuser\AppData\Local\AtomicHeart\Saved\Crashes\UE4CC-Windows-XXXX_0000
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000
```

В логе Proton (`PROTON_LOG`) видно:

```
winedmo_demuxer_create url L"../../../AtomicHeart/Content/Movies/Launch_FHD_60FPS_PC_Steam.mp4"
demuxer_create Failed to open input, error -22 (Invalid argument).
demuxer_create Failed to open input, error 1 (Error number 1 occurred).
winedmo_demuxer_create demuxer_create failed, status 0xc0000001
```

### Причина

Стартовый ролик `Launch_FHD_60FPS_PC_Steam.mp4` (H264 + AAC) проигрывается через **Media Foundation**,
которую в Proton реализует **winedmo** (FFmpeg-бэкенд). У winedmo дефектная реализация чтения/`Seek()`
байт-стрима — он не может прочитать поток (известный баг, ломает старт-видео и в других играх; на
Steam-версии Atomic Heart видео тоже сломаны). MF не получает media source → `CreatePresentationDescriptor`
возвращает NULL → разыменование NULL → краш.

Переключение на второй бэкенд (`winegstreamer`) убирает краш winedmo, **но крашится сам winegstreamer**
в glue-слое `IMFByteStream → wg_parser`. Сам файл при этом **исправен** (декодируется хостовым
gstreamer без ошибок). То есть оба MF-бэкенда Proton не тянут этот ролик.

### Фикс — пропустить стартовый ролик

Проверенный обходной путь (как и на Steam-версии): убрать стартовые ролики, игра грузится сразу в меню.

```bash
# путь к Movies внутри префикса VK Play (поправьте под свой)
MOV="$HOME/Games/atomic-heart-vk/pfx/drive_c/VK Play/Atomic Heart/AtomicHeart/Content/Movies"

# переименовываем ВСЕ Launch_* (PC_Steam + PS4/PS5/Xbox — чтобы игра не подхватила другой вариант)
for f in "$MOV"/Launch_*.mp4; do mv -v "$f" "$f.bak"; done
```

Откат (вернуть ролики):

```bash
for f in "$MOV"/Launch_*.bak; do mv "$f" "${f%.bak}"; done
```

> ⚠️ **VK Play GameCenter** при проверке/обновлении сверяет GUP-манифест (size + MD5) и пометит
> переименованные файлы как «отсутствующие» → докачает. На обычном запуске обычно не трогает.
> Если докачивает на каждом старте — обходить проверку GameCenter / запускать
> `AtomicHeart-Win64-Shipping.exe` напрямую.

Внутриигровые ролики (мультики на выборе сложности и т.п.) на Proton будут просто **чёрными**,
но **не крашат** игру.

### Дополнительно — форс GStreamer-бэкенда

В переменные окружения добавляем:

```
PROTON_MEDIA_USE_GST=1
```

Отключает winedmo и переводит видео на winegstreamer. Полезно для внутриигровых видео и в целом
безопаснее на этой связке. (Это та же переменная, которую GE-Proton проставляет в protonfixes
для игр с битым видео.)

---

## Симптом 2 — мыло / рендер в 720p, курсор зажат в углу

После лого появляется чёрное окно в **1280×720**, картинка мыльная, а курсор в меню двигается
только в пределах верхнего-левого угла (зоны 720p поверх 4K-экрана).

### Причина

В `GameUserSettings.ini` рассинхрон: `ResolutionSizeX/Y` стоит 4K, но фактический размер окна
берётся из `DesiredScreenWidth/Height=1280×720`.

### Фикс

Файл:
```
<prefix>/drive_c/users/steamuser/AppData/Local/AtomicHeart/Saved/Config/WindowsNoEditor/GameUserSettings.ini
```

Привести все поля разрешения к нативному (пример для 4K):

```ini
ResolutionSizeX=3840
ResolutionSizeY=2160
LastUserConfirmedResolutionSizeX=3840
LastUserConfirmedResolutionSizeY=2160
DesiredScreenWidth=3840
DesiredScreenHeight=2160
bUseDesiredScreenHeight=True
LastUserConfirmedDesiredScreenWidth=3840
LastUserConfirmedDesiredScreenHeight=2160
```

Ключевым было именно `DesiredScreenWidth/Height`. После этого — нативное 4K, курсор по всему экрану.

---

## HDR

В **этой сборке игры нет настройки HDR**, поэтому UE4 не активирует HDR-свопчейн и HDR на выходе
не появляется, какие бы переменные ни ставить.

Если игра HDR поддерживает, то на KDE Plasma 6.6 + Wayland + NVIDIA (драйвер новее 595.58.03)
HDR без gamescope включается так (для GE-Proton):

```
PROTON_ENABLE_WAYLAND=1   # Wine Wayland-драйвер
PROTON_ENABLE_HDR=1       # = DXVK_HDR=1 в GE-Proton
DXVK_HDR=1
```
+ включить HDR в самой игре (`bUseHDRDisplayOutput=True`). На драйверах старше 595.58.03
дополнительно нужен `vk-hdr-layer` + `ENABLE_HDR_WSI=1`.

> **gamescope** как путь к HDR здесь не годится: он крашит сам GameCenter (VKGC), и к тому же
> gamescope-HDR на NVIDIA после Plasma 6.5 сам по себе сломан.

---

## Итоговая конфигурация Lutris (env-секция)

```yaml
system:
  disable_runtime: true
  env:
    GAMEID: '0'
    LC_ALL: ''
    PROTONPATH: /home/<user>/.local/share/Steam/compatibilitytools.d/GE-Proton10-34
    PROTON_MEDIA_USE_GST: '1'        # винегстример вместо битого winedmo
    DXVK_HDR: '1'                    # HDR-готовность (в игре эффекта нет — нет настройки)
    PROTON_ENABLE_HDR: '1'
    PROTON_ENABLE_WAYLAND: '1'
    WINEDLLOVERRIDES: sl.interposer=
    STEAM_COMPAT_CLIENT_INSTALL_PATH: /home/<user>/.local/share/Steam
    STEAM_COMPAT_DATA_PATH: /home/<user>/Games/atomic-heart-vk
  gamescope: false
wine:
  runner_executable: /usr/bin/umu-run
```

Для отладки видео/MF можно временно добавить:
```
PROTON_LOG: -all,+loaddll,+module,+warn,+err,+mfplat,+quartz,+dmo
PROTON_LOG_DIR: /home/<user>
GST_DEBUG: '2'
```

---

## Чек-лист «с нуля»

1. Установить игру VK Play через GameCenter (в Lutris через umu-run / GE-Proton).
2. `PROTON_MEDIA_USE_GST=1` в окружение.
3. Переименовать `Content/Movies/Launch_*.mp4` → `.bak`.
4. Запустить → дойти до меню → выставить разрешение; при мыле/720p поправить
   `GameUserSettings.ini` (`DesiredScreenWidth/Height`).
5. Играть.

---

## Источники / благодарности

- ProtonDB — Atomic Heart (app 668580), отчёты про сломанные видео и пропуск стартовых роликов.
- CachyOS proton-cachyos CHANGELOG — описание бага winedmo `Seek()`.
- GE-Proton (GloriousEggroll) — переменная `PROTON_MEDIA_USE_GST` / protonfixes.
- ValveSoftware/Proton issue #6554 (Atomic Heart, видео не рендерятся).

Диагностика проведена реверс-инжинирингом `winedmo.so` и разбором Proton-логов.

---

*Если помогло — звезда приветствуется. PR с подтверждениями на другом железе/драйверах welcome.*
