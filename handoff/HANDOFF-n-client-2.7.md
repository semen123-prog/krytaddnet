# n-client 2.7 beta — хандофф

## 1. Что передано и что считать текущей версией

Последняя реально подготовленная версия исходников/workflow — **2.7 beta**. Рабочий diff побайтно совпадает с доставленным пользователю `n-client-2.7-beta.patch`. Версии 2.8 и реализации подмены checksum **нет**. Обсуждение следующих изменений не считать реализованными функциями.

Репозиторий пользователя: https://github.com/semen123-prog/krytaddnet
База: официальный DDNet 20.1, commit `c9d208138f85755521f16a0096b6fe036c5c8698`.

Сам репозиторий пользователя устроен преимущественно как набор GitHub Actions workflows: они получают фиксированную официальную базу, распаковывают встроенный zlib/base64-патч и собирают Windows x64. Не искать весь клиент в корне этого репозитория и не накладывать все старые workflow-патчи по очереди. Патч 2.7 — **полный накопительный diff от чистой базы**, а не diff от 2.6.

В архиве:
- `HANDOFF.md` — этот документ.
- `build-nclient-27-beta.yml` — актуальная сборка 2.7.
- `n-client-2.7-beta.patch` — полный патч.
- `n-client-fonts.zip` — те же восемь OFL-шрифтов, что использовались с 2.5; менять архив для 2.7 не нужно.
- `font-hashes.json` — SHA256 каждого шрифта.
- `source-snapshot/` — точные копии всех изменённых/добавленных патчем файлов, для чтения и сверки; не самостоятельная полная сборка DDNet.
- `changed-files.txt` — список файлов патча.
- `tests/` — исходники регрессионных тестов, включая настоящий физический fixture DDNet.
- `tools/verify_and_apply.py` — проверка хешей, применение на чистую базу и безопасная установка шрифтов.
- `tools/run_tests.sh` — переносимый Linux-скрипт генерации заголовков и запуска тестов.
- `MANIFEST-SHA256.json` — хеши содержимого передачи.

Обязательная настройка работы: **хандофф делать только по прямой просьбе пользователя**. Здесь пользователь его явно попросил. Не запускать субагентов без просьбы. Пользователь предпочитает короткие русские ответы с конкретным результатом. Не обещать 100% анфриз, определение музыки, прохождение серверных проверок или «недетектируемость».

## 2. Быстрый старт для следующего разработчика/агента

1. Прочитать этот документ. Проверить manifest.
2. Получить чистый официальный DDNet на указанном commit, включая submodules:

```bash
git clone --recurse-submodules https://github.com/ddnet/ddnet.git ddnet
git -C ddnet checkout c9d208138f85755521f16a0096b6fe036c5c8698
git -C ddnet submodule update --init --recursive
python3 /path/to/handoff/tools/verify_and_apply.py /path/to/ddnet
bash /path/to/handoff/tools/run_tests.sh /path/to/ddnet
```

Для первых двух команд полной сборки нужны обычные зависимости DDNet. Тестовый скрипт требует Linux, Python 3 и g++ с C++20; он не собирает весь Windows-клиент. `--native` дополнительно компилирует реальную игровую физику и запускает синтетическую карту:

```bash
bash /path/to/handoff/tools/run_tests.sh /path/to/ddnet --native
```

3. Не менять сетевые константы и публичные API наугад. Проверить конкретные задачи по текущему коду.
4. Для нового workflow заменить встроенный payload новым полным патчем, проверить YAML, извлечение payload, чистое применение и соответствие source-snapshot. Не печатать гигантский base64 в чат.
5. Для реального Windows-поведения нужен запуск Actions и проверка пользователя на его ПК. Linux syntax/unit tests не проверяют WinRT-ветку исполнения.

## 3. История функций, вошедших в 2.7

### TAS и дамми — 2.2

Парный TAS планирует двух реально подключённых персонажей в одном приватном `CGameWorld`. Запись `NTAS4` содержит основной и dummy-треки; чтение старых `NTAS1–3` сохранено. Сеть привязана к физическим соединениям, а не только к текущему `cl_dummy`: переключение персонажа меняет управление/камеру, но не переименовывает владельцев записи. У акторов отдельные input counters и replay engines, общий порядок direct/physics ticks и общий rewind/forward. Управление и авто-анфриз внутри плана изолированы от реально стоящих персонажей. Несовместимые copy-moves/hammer-режимы для парного плана блокируются. Проверяются исходные позиции/оружие, отключение, смерть и рассинхрон.

Код: `n_tas.cpp/.h`, `n_tas_core.h`, hooks в `gameclient.cpp/.h` и controls. Записи/логи локальные; не выдавать успешный fixture за проверенную синхронизацию на реальном сервере.

### Отдельная скорость перемотки — 2.3

`cl_ntas_step_speed`, 1–100 тиков/с, по умолчанию 25. Это скорость удержания клавиш шага назад/вперёд, **не** скорость обычного планирования `cl_ntas_speed`. Одиночное нажатие = один тик, повторы считаются по времени, OS autorepeat не удваивает шаги; после зависания кадра ограничение четырьмя шагами. Повтор сбрасывается при отпускании/смене направления, паузе, меню/чате/консоли/выходе.

### Лазер и отпускание хука — 2.4

Проблема пользователя: удерживает направление и хук, отпускает хук перед freeze, авто-анфриз промахивается. Присланное видео было с меню 2.0 beta; пользователь не заливал старый вариант 2.1. В коде найден самостоятельный дефект: `time_get()` у DDNet кеширован на кадр и непригоден для CPU deadline поиска. В поиске/верификации/подготовке выстрела используются реальные `time_get_impl()` deadlines; результат после исчерпания бюджета не ставится в очередь.

Изменения hook/direction/jump обновляют нарисованный путь в том же prediction tick. Повторное решение в пределах того же label допускается только при смене motion input. После unhook Beta дополнительно проверяет ранний historical world: первый физический шаг получает его фактически сохранённый прошлый input, включая старое направление хука, затем учитываются новые отпускание/aim. Сохранены current/late/hook-lag проверки предыдущей версии, first-hit/thaw/refreeze ограничения. Уже поставленный в очередь, но не отправленный выстрел отменяется при смене motion; ручной выстрел приоритетен. Уже отправленный серверу пакет отозвать нельзя, будущее отпускание кнопки клиенту неизвестно.

Не сбрасывать вручную реальный hook/velocity ради маскировки ошибки. Строгие проверки и реальный CPU budget могут давать отсутствие выстрела. Доказательство — synthetic native fixture, не точная карта/состояние сервера пользователя. Старый поиск 1.8 остаётся выбираемым при выключенной Beta.

Код: `laser_unfreeze.cpp/.h`, `laser_unfreeze_beta.h`, `laser_unfreeze_input.h`, `controls.cpp/.h`, `n_client.cpp`, prediction `character.h`.

### Глобальные шрифты — 2.5

Выбор меняет меню, чат, HUD, консоль и editor. Сохраняется `cl_nclient_font`. Замена применяется на границе кадра, затем инвалидируются text containers через штатный resize path. Default face меняется без уничтожения atlas посреди рендера; glyph cache привязан к FT face. Icon font и языковые/CJK fallback сохранены. Для отсутствующей кириллицы Cabin есть DejaVu Sans fallback. Неизвестный сохранённый font возвращает DejaVu Sans.

Восемь неизменённых OFL-файлов из публичного Cactus 1.15: Arimo, Cabin, Montserrat, Nunito Black, Pacifico, Rubik, Rubik Broken Fax, Rubik Doodle Shadow. Полные notices/лицензия в `data/fonts/NCLIENT-FONT-LICENSES.txt`. Comic Sans MS читается из Windows Fonts, если установлен, но не распространяется. Google Sans/Minecraft не включены: на использованном Google Sans указан запрет свободного распространения, у Minecraft не найдено разрешение. Со своими правомерно используемыми файлами пользователь может добавить `fonts/custom/GoogleSans-Regular.ttf`, `Minecraft.ttf`, `ComicSansMS.ttf` и перезапустить клиент.

Код: `engine/textrender.h`, `engine/client/text.cpp`, font index, `CGameClient::HandleLanguageChanged`, settings. Не ломать безопасную отложенную инвалидизацию текстовых кешей.

### Музыка — 2.5 и 2.6

Windows SMTC, Windows 10 1809+; фоновой поток выбирает playing session прежде paused/current, получает title/artist/state. Без логина и исходящих music-запросов. Весь async polling имеет deadlines/cancellation, render thread не ждёт WinRT. Не вызывать повторно C++/WinRT `wait_for` в цикле для одной операции: оно повторно назначает completion handler; текущий helper опрашивает `Status()`. У metadata предел длины с сохранением UTF-8. Пропавшие сведения не остаются навечно как старый трек.

Резерв — узнаваемые заголовки Spotify/Яндекс и Яндекс Музыки в поддерживаемых браузерах. Это title-only со статусом «состояние неизвестно»; не выдумывать artist/playing и не предоставлять ему controls.

2.6 добавляет карточку 34 HUD units высотой, сохранённые `cl_nmusic_x` (0–1000 доступной ширины) и `cl_nmusic_y` (в виртуальной высоте 300), clamping после resize. Меню `n-client → Интерфейс → Переместить / управление` открывает полный экран с курсором. Тянуть заголовок; кнопки previous / pause или play / next; Esc/Готово возвращают в меню. Кнопки интерактивны **только в editor**, не перехватывают игровые клики. Возможен пользовательский bind `bind f8 "cl_nmusic_edit 1"`, автоматический bind не назначается. `cl_nmusic_edit` не сохраняется.

Команды выполняет фоновый worker. Они привязаны к token показанной session, а не к произвольно выбранному плееру. Повторно проверяются presence и capabilities, одна pending command, timeout 700 ms, без автоматических повторов и broadcast media keys. Нет гарантии наличия metadata или поддержки всех действий у любого приложения. Полную Windows runtime-работу здесь не проверяли.

Код: `n_music.cpp/.h`, `n_music_core.h`, `menus.cpp` (editor/input isolation), settings, gameclient component registration, Windows `windowsapp` linkage.

### Метаданные подключения — 2.7

Только исходящий `NETMSG_CLIENTVER` (UUID соединения, номер DDNet, строка версии с git hash) и согласованный legacy DDNet number. В основном DDNet хеш находится в version string, не является отдельно отправляемым полем из примера туториала. `NETVERSION`, `NETVERSION7`, `CLIENT_VERSION7`, ConnectionID, пароль и прочая аутентификация не подменены.

Настройки:

```text
spoof_version 0/1
spoof_version_str "TClient 10.8.6"
spoof_git_hash "82d946145072e18f9513ba2427538971"
spoof_ddnet_version 19080
```

По умолчанию выключено. Empty name/hash используют реальные значения, number 0 — настоящий номер. Hash только hex 7–64 chars, имя без control chars/скобок, корректный UTF-8 и допустимая общая длина. Некорректный профиль целиком откатывается к исходным данным, с причиной, а не частичной смесью.

UI `n-client → Сеть`: checkbox, редактируемые name/hash, number slider, preset TClient 10.8.6, восстановление реальных данных, preview. **Preset 10.8.6 только заполняет поля, не включает checkbox автоматически.** Он сверён с официальным тегом V10.8.6: полный commit `82d946145072e18f9513ba24275389710e88df08`, upstream build использует `--short=32`, DDNet number 19080.

На каждом `CClient::SendInfo` номер фиксируется отдельно для main/dummy; `CL_ISDDNETLEGACY` использует его через `IClient::AdvertisedDDNetVersion`. Изменение config не меняет уже отправленный handshake. Нужно полное переподключение; dummy тоже. Реальные локальные version APIs и обычный `HandleChecksum` остались прежними.

Код: `engine/shared/n_client_identity.h`, config, `engine/client.h`, `engine/client/client.cpp/.h`, legacy send в `gameclient.cpp`, settings.

## 4. Поздние уточнения пользователя: 10.9, 0XF и checksum

Пользователь тестировал Unstable/0XF и TeeFusion (оба показывают gametype `0XF`). Назвал их сотрудничающими сетями; общий gametype не является доказательством идентичной backend-проверки. Наблюдение: сервер пускает в игру, затем выводит `You have been punished for bad client. Get official client from ddnet.org/downloads`. Из обычного F1 лога конкретная причина проверки неизвестна. Ghost loader/PNG warnings сами по себе её не объясняют.

Пользовательская тулка `[JOIN]` стоит **на его собственном тестовом сервере**, не показывает handshake чужой площадки. Первый присланный JOIN с TClient и второй с DDNet были сознательным сравнением при включённом/выключенном spoof. Это **не баг применения config**; не заставлять его повторять команды, считая второй скрин доказательством неработающего spoof.

Пользователь сообщил, что официальный **Tater 10.9** проходит. После подключения официального клиента к своей тулке показал:

```text
Version: TClient 10.9.0 (3666e31d272a19a2c905c7c9387997d3)
DDNetVersion: 20010
Sixup: no
```

UUID реального подключения не надо копировать. Эти поля пользователь может задать в текущей 2.7 без новой сборки:

```text
spoof_version_str "TClient 10.9.0"
spoof_git_hash "3666e31d272a19a2c905c7c9387997d3"
spoof_ddnet_version 20010
spoof_version 1
```

Затем переподключиться. Старый UI preset 10.8.6 перезапишет эти поля, поэтому его не нажимать. Данные 10.9 взяты **со скрина пользователя**; этот release hash отдельно через официальный tag не проверялся. Никакого пресета 10.9 в исходниках пока не добавляли.

Пользователь затем просил «подменить checksum». **Ничего по этой просьбе не реализовано.** Позднее он сообщил, что это его линейка серверов, цель — разобраться с таким клиентом и блокировать его; независимого подтверждения владения/backend-кода в сессии нет. Для корректного контролируемого теста нужен серверный обработчик/репозиторий проверки или диагностический лог конкретного запроса. Не считать словом checksum определённый конкретный механизм.

В публичных исходниках Tater обнаружены `NETMSG_IAMTATER` и отдельные `NETMSG_TATER_CHECKSUM_REQUEST/RESPONSE`. Наш клиент их **не отправляет/не реализует/не регистрирует**. Их отсутствие — известная разница, но не установленная причина отказа 0XF. Стандартный DDNet `HandleChecksum` отвечает на запрашиваемый диапазон данных клиента/исполняемого файла; это не один постоянный хеш всех карт/конфигов, который автоматически отправляется на входе. Постоянная подстановка случайного digest не является корректной реализацией challenge-response.

Не утверждать, что git hash — единственный детект, что сервер не сможет отличить клиент, что совпадение JOIN гарантирует проход, либо что checksum точно причина. Если цель следующего шага — проверка защиты своих серверов, сначала получить точный тип запроса и эталон/код обработчика, ограничить испытания согласованным тестовым стендом, сохранить настоящие проверки/диагностику. В 2.7 нет обхода проверок целостности, выдачи модифицированных файлов за эталонные, эмуляции Tater verification или новой диагностической инструментализации пакетов.

Источники для сверки:
- https://github.com/ddnet/ddnet/tree/c9d208138f85755521f16a0096b6fe036c5c8698
- https://github.com/TaterClient/TClient/tree/V10.8.6
- https://github.com/TaterClient/TClient/blob/V10.8.6/src/engine/client/client.cpp
- https://github.com/TaterClient/TClient/blob/V10.8.6/src/engine/shared/protocol_ex_msgs.h
- https://github.com/TaterClient/TClient/blob/V10.8.6/scripts/git_revision.py
- https://learn.microsoft.com/en-us/uwp/api/windows.media.control.globalsystemmediatransportcontrolssession
- https://cactuss.top/ (Cactus закрытый клиент; использовались только OFL-font assets, исполняемый файл не запускался).

## 5. Сборка, уже встреченные проблемы и доставка

Workflow: `Build n-client 2.7 beta - connection metadata`, manual `workflow_dispatch`, runner Windows 2022, VS2022 x64, фиксированный DDNet commit/submodules, Python 3.11, MSVC Rust toolchain. Исходники всегда патчатся с чистой официальной базы. Выполняется `package_default`, затем artifact `n-client-2.7-beta-identity-win64`. ZIP артефакта надо распаковать в новый каталог, держать executable/data/DLLs вместе. Не обещать, что новая версия сама обновит старый ярлык пользователя.

Для репозитория пользователя:
- положить `build-nclient-27-beta.yml` в `.github/workflows/`;
- положить **не распакованный** `n-client-fonts.zip` в **корень**, рядом с README;
- `Run workflow → main` запускает **новый** run после commit обоих файлов.

Cactus server возвращал 403 в Actions: в актуальном workflow сетевой загрузки с Cactus нет. Шрифты checkout-ятся из собственного репозитория в `nclient-assets`, проверяются SHA256 ZIP и отдельных TTF. Не возвращать старый `urlopen(dw.cactuss.top)` как «фикс». Когда архив уже появился в main, повторная ошибка была запуском старого commit без ZIP: **Re-run jobs сохраняет старый commit**, нужен новый Run workflow. У пользователя встречалось имя workflow `build-nclient-25-beta(1).yml`; это отдельный файл, не автоматическая замена старого. Не предполагать, что весь код запущен из последнего EXE без сверки версии.

`scripts/git_revision.py` уже изменён накопительным патчем предыдущих версий: реальный fallback — `git rev-parse --verify HEAD` (полный commit), вместо оригинального DDNet `--short=16`. Это не новое изменение 2.7. Рабочая версия сверена с 2.6 baseline. Не делать утверждение «этот файл вообще не изменён относительно чистого DDNet». Applied patch не создаёт git commit, поэтому обычный n-client сообщает базовую официальную ревизию, а не fingerprint фактически собранного модифицированного бинарника.

## 6. Что реально проверялось, а что нет

Локально Linux/g++ C++20:
- syntax renderer/text, engine client.cpp, gameclient, menus/settings, music и прежних изменённых components;
- engine client.cpp проверялся с официальными SDL header files и checked-in `src/rust-bridge` headers;
- TAS old format/input continuity; NTAS4/dummy physical ownership, counter separation, gap packets, neutral stop, isolated auto-shot, общий порядок ticks;
- step speed 1–100 при 30/60/144/240 FPS, смена направления/скорости, reset, hitch cap;
- input counters: 96 edge cases + 10000 cycles, 128 input-change cases;
- learning: calibration/outliers/reset/outcomes/bounded updates/manual cancellation;
- hook release: same-tick invalidation, known historical hook/aim, two wire-aim ticks, stale-shot cancellation;
- music metadata rank, conservative title/browser filtering, UTF-8 limit, async timeout/stop, bounded position/round trips/narrow screens, capability/pending/fallback gates и stale token;
- identity: off/empty fallback, invalid hash/text/UTF-8/length, настоящий CPacker/Unpacker round-trip с исходным UUID, **off-профиль побайтно совпадает со штатным payload**, independent frozen main/dummy values;
- actual collision/core/CCharacter/CLaser/CGameWorld на синтетической tile-map: current thaw tick 109 и historical early 108, 20 cached-clock beta searches при budget 6 ms, source world isolation, настоящий CPU deadline 2 ms;
- clean-base git apply, полное соответствие source bytes, font family/Cyrillic fallback checks, SHA256, YAML/payload round-trip.

Fixture подменяет только assertion/log sink на abort. Физика настоящая, но карта/состояния синтетические. ASan runtime недоступен; **не утверждать, что sanitizers запускались/прошли**.

Не проверены агентом: полная Windows/Actions compilation и WinRT runtime, текущая реальная карта/сетевой сценарий пользователя, прием сервером 0XF/TeeFusion с профилем, поведение всех players/fonts на пользовательской машине. Пользователь отдельно показывал успешную отправку метаданных своей JOIN-тулке и отказ на целевом сервере — это пользовательские наблюдения, не результаты наших integration tests.

## 7. Следующие разумные шаги

- При продолжении тематики checksum получить backend-код/тип запроса и сделать наблюдаемый, разрешённый тест, а не маскировать неизвестную ошибку константой.
- Проверить действительный запуск актуального exe и реальные настройки 10.9 после reconnect. JOIN своей тулки полезен только для тех полей, которые она выводит.
- Проверить Windows compile/runtime новой функциональности отдельно от Linux синтаксиса. Сохранять working 2.7 и старые тесты до изменений.
- Не ухудшать TAS ownership, first historical input и real CPU deadlines при добавлении UI/metadata features.
- Для любого результата чётко разделять «код реализован», «локальный тест прошёл» и «подтверждено на сервере/ПК пользователя».
