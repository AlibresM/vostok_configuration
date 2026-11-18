# AGENTS.md — VOSTOK Configuration

Конфигурация Klipper для 3D-принтера VOSTOK (IDEX, две печатающие головы).
Репозиторий содержит базовую конфигурацию, набор модулей электроники и
слайсер-постпроцессоры для быстрой смены инструмента.

## Стек и зависимости

- **Klipper**: обязательно ветка [`generic-cartesian`](https://github.com/dmbutyugin/klipper/tree/generic-cartesian) от dmbutyugin. Mainline не подходит — нужен `generic_cartesian` kinematics и `dual_carriage` с `SET_DUAL_CARRIAGE MODE=DIRECT` (см. `[printer]` и `[dual_carriage wc]` в `printer_base.cfg`).
- **gcode_shell_command** (устанавливается через KIAUH, раздел Extensions): нужен для `IDEX_SHAPER_CALIBRATE`. Секции `[gcode_shell_command ...]` подключены в `printer_base.cfg` всегда, поэтому без расширения Klipper не запустится.
- **Python 3**: для `postprocessing/*.py`. Только stdlib (argparse, re, math, heapq, datetime, optparse, sys, pathlib). Никаких pip-зависимостей.
- **Слайсер**: post-processing scripts цепляются в PrusaSlicer / SuperSlicer / OrcaSlicer.

## Структура репозитория

```
printer.cfg                  # Пользовательский файл. Редактируется под каждый принтер.
printer_base.cfg             # !!! НЕ РЕДАКТИРОВАТЬ. База: carriages, steppers, extruders, homing, IDEX-макросы.
electronics_*.cfg            # Официально поддерживаемые модули распиновки под конкретную плату + коммутационные платы.
chamber_heater.cfg           # Опциональный модуль нагревателя активной термокамеры (на основе Smart Chamber Heater, MIT).
iheater_2x.cfg               # Опциональный модуль: два нагревателя камеры iHeater, каждый со своим USB-MCU (на основе 2XiHeater.cfg, GPLv3).
                             # Подключается вместо chamber_heater.cfg: оба определяют M141, M191 и TURN_OFF_HEATERS.
postprocessing/
  fast_tool_swaps.py         # Слайсер-постпроцессор быстрой смены инструмента (dmbutyugin, GPLv3).
  fix_toolchange.py          # Чистит G-code: убирает G1 E* при T-переключениях, склеивает дубли, добавляет deretract.
pics/                        # Картинки для README.
README.md                    # Пользовательская документация (на русском).
README.en.md                 # её английский перевод.
LICENCE.md                   # Лицензия CC-BY 4.0 для *.cfg.
user_configs/                # Пользовательские конфигурации электроники.
  README.md                  # Описание папки (на русском) + README.en.md.
```

## Архитектура конфигурации

Конфиг разделён на три слоя по принципу "приоритет последнего include" (Klipper применяет последнее значение одноимённого параметра):

1. **`printer_base.cfg`** — параметры и макросы, общие для всех принтеров. Содержит:
   - секции кареток `[carriage xc]`, `[dual_carriage wc]`, `[carriage ylc]`, `[extra_carriage yrc]`, `[carriage zc]`;
   - stepper-секции `[stepper motor_x]`, `[stepper motor_w]`, `[stepper motor_yl]`, `[stepper motor_yr]`, `[stepper motor_z]`;
   - `[extruder]` и `[extruder1]`;
   - `[homing_override]` с G28-логикой, использующей `_IDEX_SETTINGS` и `_HOME_Z`;
   - макросы IDEX: `IDEX_RESET`, `IDEX_MODE_FULL_CONTROL`, `IDEX_MODE_COPY`, `IDEX_MODE_MIRROR`, `T0`, `T1`, `PARK_X`, `PARK_W`, `PARK_XW`, `SET_W_OFFSETS`, `CALC_OFFSET`;
   - макросы печати: `PRINT_END`, `CANCEL_PRINT`, `PAUSE`, `RESUME`, `M600`, `M601`;
   - калибровка: `IDEX_SHAPER_CALIBRATE`, `ENABLE_EQUAL_PA`, `DISABLE_EQUAL_PA`, обёртка `SET_PRESSURE_ADVANCE`;
   - передача данных из слайсера (сейчас не используются ни конфигом, ни стартовым G-кодом): `SET_PA_VALUES`, `APPLY_PA_VALUES`, `SET_NEXT_PRINT_MODE`, `RESET_VARIABLES`.
2. **`electronics_*.cfg`** — только распиновка (`step_pin`, `dir_pin`, `enable_pin`, `endstop_pin`, `heater_pin`, `sensor_pin`, `pin` вентиляторов), настройки драйверов (`run_current`, `[tmcXXXX ...]`), акселерометры и `[resonance_tester]`.
3. **`printer.cfg`** — калибруемые пользователем параметры: `position_min/max/endstop`, `max_velocity`, `rotation_distance`, `pid_*`, `z_offset`, `bed_mesh`, `screws_tilt_adjust`, `_IDEX_SETTINGS`, `[mcu ...]` с CAN-uuid, `[include ...]` нужного `electronics_*.cfg`.

**`printer.cfg` подключается строго в порядке** (см. раздел «Базовая конфигурация» в начале `printer.cfg`):

```
[include printer_base.cfg]
[include electronics_*.cfg]    # ← единственный include, который выбирает пользователь
... остальное — параметры и переопределения
```

Не добавлять `[include]` между `printer_base.cfg` и `electronics_*.cfg` — `printer_base.cfg` должен идти первым, иначе IDEX-макросы перебьют параметры из последующих include.

**Начало печати** выполняется стартовым G-кодом слайсера (нагрев, `G28`, карта высот, режим IDEX, линия очистки, парковка). Макросов `PRINT_START*` в конфигурации нет — они удалены в v9.5. Конец печати — `PRINT_END`.

## Правила именования и каретки

- Оси: `X` управляется кареткой `xc`, `W` (правая IDEX-каретка) — `wc`, `Y` — двумя каретками `ylc` (ведущая) + `yrc` (ведомая через `[extra_carriage]`), `Z` — `zc`.
- Extruders: `extruder` (левый, на T0CB) и `extruder1` (правый, на T1CB).
- Моторы: `motor_x`, `motor_w`, `motor_yl`, `motor_yr`, `motor_z` — эти имена должны сохраняться везде, на них ссылаются секции `[carriage ...]` и `homing_override`.
- Шаблоны имён MCU: `T0CB` (левая коммутационная плата) и `T1CB` (правая). CAN-uuid задаётся в `printer.cfg` (`[mcu T0CB]`, `[mcu T1CB]`); без этих секций пины вида `T0CB:PB9` не разрешатся.
- Акселерометры: `[adxl345 t0_accelerometer]` и `[adxl345 t1_accelerometer]`. Эти имена `IDEX_SHAPER_CALIBRATE` использует по умолчанию (переопределяются через `t0_accel_chip` / `t1_accel_chip` в `_IDEX_SETTINGS`).

## Именование `electronics_*.cfg`

Шаблон: `electronics_<материнская>_<ревизия>_<MCU>_<двигатели-материнской>x<модель-драйвера>_<коммутационные>x<модель>.cfg`.

Пример: `electronics_octopus_pro_v1.1_h723_5x2240_2xH36v1.3.cfg` = BTT Octopus Pro v1.1 (STM32H723) + 5 драйверов TMC2240 + 2 коммутатора Fysetc H36 v1.3.

В шапке такого файла обязательно:
- перечень плат и драйверов;
- таблица соответствия "оборудование → разъём" (см. шапку `electronics_octopus_pro_v1.1_h723_5x2240_2xH36v1.3.cfg`);
- блоки `[stepper ...]`, `[extruder]`, `[extruder1]`, `[heater_bed]`, `[fan]`, `[heater_fan ...]`, `[controller_fan ...]`, `[probe]` и `endstop_pin` кареток с пинами.

## Команды печати (IDEX)

Макросы для пользователя (все в `printer_base.cfg`):

| Макрос | Назначение |
|---|---|
| `IDEX_RESET` | Сброс в классический режим, активная голова = T0. |
| `IDEX_MODE_FULL_CONTROL` | Полное раздельное управление головами: левая — ось X, правая — ось W. |
| `IDEX_MODE_COPY [PRINTHEAD_DISTANCE=...] [MOVE=1]` | Дублирующий режим. |
| `IDEX_MODE_MIRROR [MOVE=1]` | Зеркальный режим. |
| `T0`, `T1` `[X=] [Y=] [Z=]` | Смена инструмента. С координатами: откат, одновременное перемещение следующей головы в точку и предыдущей на парковку, возврат отката (используется скриптом быстрой смены инструмента). |
| `PARK_X`, `PARK_W` `[RETRACT=...]`, `PARK_XW` | Парковка левой, правой или обеих голов. Режим IDEX, активный экструдер и активная голова после парковки не меняются. |
| `PAUSE` / `RESUME`, `M600`, `M601` | Пауза с подъёмом стола и парковкой голов, продолжение печати с возвратом в точку паузы. `M600` и `M601` вызывают `PAUSE`. |
| `PRINT_END`, `CANCEL_PRINT` | Конец/отмена печати: выключение нагревателей, опускание стола, парковка голов. |
| `SET_W_OFFSETS X=... Y=...` | Смещения правой головы (только классический режим). |
| `CALC_OFFSET XL= XR= YF= YB=` | Расчёт смещений правой головы по калибровочной модели (только выводит значения). |
| `IDEX_SHAPER_CALIBRATE [TEST_W=1] [TEST_Y=0]` | Калибровка input shaper для X или W и для Y, акселерометром тестируемой головы. |
| `ENABLE_EQUAL_PA` / `DISABLE_EQUAL_PA` | Одинаковый Pressure Advance для обеих голов (для калибровки PA в зеркальном/дублирующем режиме). |

Ключевые переменные в `_IDEX_SETTINGS` (`printer.cfg`, раздел «Настройки принтера»):
- `default_printhead_distance` — расстояние между головами в режиме копирования;
- `right_printhead_x/y_offset` — оффсеты правой головы (только классический режим);
- `reset_on_g28` — сбрасывать ли режим IDEX при `G28`;
- `safety_offset` — отступ от крайних позиций при парковках;
- `toolchange_retract_length/speed/travel_speed` — параметры T0/T1;
- `t0_accel_chip` / `t1_accel_chip` — акселерометры голов для `IDEX_SHAPER_CALIBRATE`.

## Особенности макросов

- Klipper вычисляет весь Jinja-шаблон макроса **до** выполнения первой его команды. Все значения `printer.…` внутри макроса — состояние на момент его вызова. Если нужно состояние после команд внутри макроса, эту часть выносят во вспомогательный макрос: он вычисляется в момент своего вызова (так сделаны `_PARK_PREPARE` и `_PARK_RESTORE_EXTRUDERS`).
- `RESTORE_GCODE_STATE` и `RESTORE_DUAL_CARRIAGE_STATE` не восстанавливают активный экструдер и синхронизацию экструдеров (`SYNC_EXTRUDER_MOTION`) — это нужно делать отдельно.
- В этой ветке Klipper `ACTIVATE_EXTRUDER` завершается ошибкой, если экструдер синхронизирован с другим: сначала `ACTIVATE_EXTRUDER`, потом `SYNC_EXTRUDER_MOTION`.

## Post-processing scripts

`postprocessing/` запускаются из слайсера:

- `fast_tool_swaps.py` — основной. Парсит G-code, моделирует состояние принтера (класс `GCodeState` в самом скрипте), заранее нагревает и прочищает следующую голову (через `SYNC_EXTRUDER_MOTION`), вставляет wipe-движения по оси W перед T-переключениями. Обрабатывает только G-code с маркером `;fast_tool_swaps_start` (ставится в стартовом G-коде слайсера). Помечает выход `; Processed by fast tool swaps script`.
- `fix_toolchange.py` — вспомогательный. Удаляет `G1 E*` в строках вида `T... X... Y...`, удаляет подряд идущие дубли, добавляет deretract-команду (`--long_deretract_length`).

Замечание: эти два скрипта **не импортируются конфигом Klipper**. Они обрабатывают G-code до печати, но рассчитывают на макросы `T0`/`T1` с параметрами `X`/`Y`.

## Стиль и конвенции

- **Кодировка и язык**: комментарии — на русском, UTF-8.
- **Язык документации**: каждый пользовательский README существует в двух версиях — `README.md` (русский) и `README.en.md` (английский). Правки в одной переносятся на вторую в том же коммите, иначе версии разъедутся.
- **Лицензия**: CC-BY 4.0 для `*.cfg`, шапка указана в каждом cfg. Исключения — `chamber_heater.cfg` (MIT) и `iheater_2x.cfg` (GPLv3), см. их шапки. Скрипты в `postprocessing/` — GPLv3.
- **Macro-именование**: пользовательские макросы — `UPPER_SNAKE_CASE` (`IDEX_RESET`, `T0`, `IDEX_MODE_COPY`). Служебные (с подчёркиванием) — `_IDEX_SETTINGS`, `_IDEX_VARIABLES`, `_HOME_Z`, `_PARK_PREPARE`, `_PARK_RESTORE_EXTRUDERS`.
- **Параметры Klipper**: `snake_case`.
- **Порядок секций в `printer_base.cfg`**: printer → carriages → steppers → extruders → bed/probe → homing → fans → macros. При добавлении секции держитесь этой структуры.
- **Header в `printer_base.cfg`** начинается с `# !!! НЕ ВНОСИТЕ ИЗМЕНЕНИЯ В ЭТОТ ФАЙЛ !!!`. Любые пользовательские правки делаются в `printer.cfg` путём копирования секции и переопределения значения.

## Типичные задачи

### Добавить конфиг для новой платы
1. Создать `electronics_<плата>.cfg` в `user_configs/` (в корне лежат только официально поддерживаемые конфигурации).
2. В шапке — таблицу соответствия разъёмов.
3. Скопировать структуру существующего `electronics_*.cfg`.
4. Создать ветку `add-<платы>`, оформить PR (см. README.md, раздел "Как добавить свою конфигурацию"; английская версия — README.en.md).

### Изменить параметр из базы
Скопировать нужную секцию/параметр в `printer.cfg` в раздел "Пользовательские настройки" (в конце файла) и поменять значение. Не редактировать `printer_base.cfg`.

### Калибровка
- `rotation_distance` экструдера — секции `[extruder]` и `[extruder1]` в `printer.cfg`.
- `position_min/endstop/max` кареток — секции `[carriage xc]`, `[dual_carriage wc]`, `[carriage ylc]`, `[carriage zc]` в `printer.cfg`.
- `safe_distance` второй каретки — `[dual_carriage wc]` в `printer.cfg` (занизить на ~0.1мм для обхода бага расчёта safe-distance).
- PID — закомментированы в `[extruder]`, `[extruder1]` и `[heater_bed]` в `printer.cfg`; раскомментировать и вписать после `PID_CALIBRATE`.
- Input shaper — `IDEX_SHAPER_CALIBRATE` (и с `TEST_W=1` для правой головы), результат вписать в `[delayed_gcode init_shaper]` в `printer.cfg`.

### Изменение параметров T0/T1
Через `_IDEX_SETTINGS` в `printer.cfg`. Не трогать сами макросы `T0`/`T1` в `printer_base.cfg`.

## Язык ответов

Пользовательская документация в репозитории — на русском. Комментарии в `*.cfg`, обсуждения PR и issues — также на русском. Отвечайте на русском, если не указано иное.
