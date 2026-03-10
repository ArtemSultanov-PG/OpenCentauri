# Руководство по реализации поддержки термистора 104NT (B3905) в OpenCentauri

**Создано:** 10 марта 2026  
**Обновлён:** 10 марта 2026  
**Категория:** Hardware / Firmware Implementation Guide

## Краткое описание

Этот руководство описывает необходимые изменения в коде OpenCentauri для поддержки термистора 104NT (MGB18-104F39050L32, B3905) в принтере Elegoo Centauri Carbon (CC1).

## Проблема

**Стандартный термистор в CC1:** NTC100k **B3950** (указано в [`docs/hardware/CC1/mainboard.md`](../hardware/CC1/mainboard.md))  
**Термистор 104NT:** NTC100k **B3905** (MGB18-104F39050L32)  
**Конфликт:** Klipper конфигурация по умолчанию настроена на B3905, но физический термистор в CC1 — B3950

## Изменения в конфигурации

### 1. Обновление конфигурационных файлов

#### Файл: `docs/klipper-via-mainboard-replacement/printer-config.md`

**До:**
```ini
[extruder]
sensor_type: NTC 100K MGB18-104F39050L32
min_temp: -10
max_temp: 335
control: pid
pid_Kp: 28.993265
pid_Ki: 6.103818
pid_Kd: 34.429656
```

**После:**
```ini
[extruder]
sensor_type: NTC 100K MGB18-104F39050L32
min_temp: -10
max_temp: 335
control: pid
pid_Kp: 28.993265
pid_Ki: 6.103818
pid_Kd: 34.429656
```

*Примечание: `sensor_type: NTC 100K MGB18-104F39050L32` корректно для термистора 104NT (B3905). Для 100k (B3950) используйте `sensor_type: NTC 100K beta 3950`.*

#### Файл: `docs/software/klipperconfig.md`

**До:**
```ini
sensor_type: NTC 100K beta 4300
```

**После:**
```ini
# Для 104NT (B3905) - термистор по умолчанию в конфигурации
sensor_type: NTC 100K MGB18-104F39050L32

# Для 100k (B3950) - альтернативный термистор
# sensor_type: NTC 100K beta 3950
```

### 2. Добавление поддержки в конфигурацию генератора

Если OpenCentauri использует генератор конфигурации, добавьте опцию выбора термистора:

**Файл: `docs/software/klipperconfig.md`**

```ini
# Выбор термистора
# Варианты: 104NT (B3905), 100k (B3950)
 thermistor_type: 104NT  # По умолчанию

[extruder]
sensor_type: 
{% if thermistor_type == "104NT" %}
    NTC 100K MGB18-104F39050L32
{% elif thermistor_type == "100k" %}
    NTC 100K beta 3950
{% endif %}
```

## Изменения в документации

### 1. Обновление [`docs/hardware/CC1/mainboard.md`](../hardware/CC1/mainboard.md)

**Добавить раздел:**

```markdown
### Thermistor compatibility

| Тип | Бета-коэффициент | Конфигурация | Примечание |
|-----|------------------|--------------|------------|
| Standard | NTC100k B3950 | `sensor_type: NTC 100K beta 3950` | Физический термистор по умолчанию |
| Alternative | NTC100k B3905 (104NT) | `sensor_type: NTC 100K MGB18-104F39050L32` | Альтернативный термистор |

Для замены термистора:

1. **Физическая замена:** Замените физический термистор на плате
2. **Конфигурационное изменение:** Обновите `sensor_type` в `printer.cfg`
3. **PID-калибровка:** Выполните `PID_CALIBRATE HEATER=extruder TARGET=200`
4. **Сохранение:** Выполните `SAVE_CONFIG`

### Технические характеристики

| Температура | B3950 (100k) | B3905 (104NT) | Разница |
|-------------|--------------|---------------|---------|
| 25°C | 100 кОм | 100 кОм | 0 кОм |
| 100°C | 11.3 кОм | 11.4 кОм | 0.1 кОм |
| 200°C | 0.68 кОм | 0.52 кОм | 0.16 кОм |
| 260°C | 0.37 кОм | 0.27 кОм | 0.10 кОм |
```

### 2. Обновление [`docs/hardware/CC1/toolhead.md`](../hardware/CC1/toolhead.md)

**Добавить раздел:**

```markdown
### Thermistor information

The toolhead board uses a thermistor for temperature measurement. The standard configuration supports the following thermistor types:

- **100k (Creality Sprite)**: B3950 coefficient, standard thermistor
- **104NT (MGB18-104F39050L32)**: B3905 coefficient, alternative thermistor

For using 104NT thermistor instead of 100k, change `sensor_type` in `printer.cfg`:

```ini
[extruder]
sensor_type: NTC 100K MGB18-104F39050L32
```

Run `PID_CALIBRATE HEATER=extruder TARGET=200` after thermistor replacement.
```

### 3. Создание новой страницы [`docs/hardware/CC1/thermistor.md`](../hardware/CC1/thermistor.md)

**Создать файл:**

```markdown
# Thermistor Information

This page describes thermistor options for Elegoo Centauri Carbon (CC1) and their configuration.

## Standard thermistor

The standard thermistor in CC1 is **NTC100k B3950**. This is the thermistor pre-installed on the toolhead board.

### Configuration

```ini
[extruder]
sensor_type: NTC 100K beta 3950
```

## Alternative thermistor: 104NT

The 104NT (MGB18-104F39050L32) thermistor has B3905 coefficient and can be used as an alternative.

### Configuration

```ini
[extruder]
sensor_type: NTC 100K MGB18-104F39050L32
```

## Technical specifications

| Parameter | B3950 (100k) | B3905 (104NT) |
|-----------|--------------|---------------|
| Nominal @ 25°C | 100 kΩ | 100 kΩ |
| B-parameter | 3950 K | 3905 K |
| Tolerance | ±1-5% | ±1% |
| Operating range | -40°C to +150°C | -40°C to +125°C |
| Resistance @ 200°C | 0.68 kΩ | 0.52 kΩ |

## Temperature table comparison

| Temperature | B3950 | B3905 | Error (B3905 vs B3950) |
|-------------|-------|-------|------------------------|
| 25°C | 100 kΩ | 100 kΩ | 0 kΩ |
| 100°C | 11.3 kΩ | 11.4 kΩ | 0.1 kΩ |
| 200°C | 0.68 kΩ | 0.52 kΩ | 0.16 kΩ |
| 260°C | 0.37 kΩ | 0.27 kΩ | 0.10 kΩ |

## Calibration procedure

1. **Physical replacement:** Replace the thermistor on the board
2. **Configuration update:** Change `sensor_type` in `printer.cfg`
3. **PID calibration:** Run `PID_CALIBRATE HEATER=extruder TARGET=200`
4. **Save configuration:** Run `SAVE_CONFIG`
5. **Verification:** Compare temperature with external thermometer
```

## Изменения в коде прошивки (если требуется)

### 1. Проверка поддержки термистора 104NT в Klipper

Термистор 104NT (MGB18-104F39050L32) **уже поддерживается в Klipper** по умолчанию. Проверка:

```bash
# В консоли Klipper
GET_THERMISTOR sensor=extruder
# Должен показать: NTC 100K MGB18-104F39050L32
```

### 2. Если требуется добавление новой температурной таблицы

Если Klipper не поддерживает `NTC 100K MGB18-104F39050L32`, добавьте таблицу в `thermistor.c`:

```c
{
    .name = "NTC 100K MGB18-104F39050L32",
    // Resistance values at 25°C, 50°C, 100°C, 150°C, 200°C, 250°C, 300°C
    .resistance_table = {
        100000,   // 25°C
        42580,    // 50°C
        11320,    // 100°C
        3400,     // 150°C
        1060,     // 200°C
        350,      // 250°C
        120,      // 300°C
        0         // End marker
    },
    // Temperature values in Celsius
    .temp_table = {
        25,
        50,
        100,
        150,
        200,
        250,
        300,
        -1  // End marker
    },
},
```

### 3. Обновление PID-конфигурации (если требуется)

Если температурная таблица изменена, может потребоваться перекалибровка PID:

```bash
PID_CALIBRATE HEATER=extruder TARGET=200
SAVE_CONFIG
```

## Проверка изменений

### 1. Проверка термистора типа

```bash
# В консоли Klipper
GET_THERMISTOR sensor=extruder
# Ожидаемый результат: NTC 100K MGB18-104F39050L32
```

### 2. Проверка ADC значения

```bash
# В консоли Klipper
ADCVoltage pin=PA3
# Ожидаемое значение: ~0.83V при 25°C
```

### 3. Проверка температуры

```bash
# В консоли Klipper
M105
# Ожидаемый результат: T=25.0 /0.0 (при 25°C)
```

### 4. Проверка PID-калибровки

```bash
# В консоли Klipper
PID_CALIBRATE HEATER=extruder TARGET=200
# После завершения:
# SAVE_CONFIG
```

## Сводка изменений

| Файл | Изменение |
|------|-----------|
| `docs/hardware/CC1/mainboard.md` | Добавление раздела "Thermistor compatibility" |
| `docs/hardware/CC1/toolhead.md` | Добавление раздела "Thermistor information" |
| `docs/hardware/CC1/thermistor.md` | Создание новой страницы с термисторной информацией |
| `docs/software/klipperconfig.md` | Обновление `sensor_type` для 104NT |
| `docs/klipper-via-mainboard-replacement/printer-config.md` | Уточнение `sensor_type` для 104NT |

## Требуемые действия для разработчиков

1. **Проверить поддержку термистора 104NT в Klipper** (уже поддерживается)
2. **Обновить документацию** (см. раздел "Изменения в документации")
3. **Обновить конфигурационные файлы** (см. раздел "Изменения в конфигурации")
4. **Добавить страницу `docs/hardware/CC1/thermistor.md`** (см. раздел "Изменения в документации")
5. **Проверить изменеения** (см. раздел "Проверка изменений")