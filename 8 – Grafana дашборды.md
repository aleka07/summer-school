
Заходим в нашу Графана. в Dashboards
![[Pasted image 20260428164128.png]]

## 2. Дашборд: Лампочка (Lightbulb Monitoring)

### 2.1 Яркость (Brightness) — Time Series

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "brightness")
```

### 2.2 Потребление энергии (Power Consumption) — Time Series

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_power_consumption_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "power_consumption")
```

### 2.3 Напряжение (Voltage) — Time Series

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_voltage_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "voltage")
```

### 2.4 Температура (Temperature) — Time Series

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_temperature_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "temperature")
```

### 2.5 Все параметры лампочки — Stat панели (текущее значение)

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) =>
       r["_field"] == "value_brightness_properties_value" or
       r["_field"] == "value_power_consumption_properties_value" or
       r["_field"] == "value_voltage_properties_value" or
       r["_field"] == "value_temperature_properties_value"
  )
  |> last()
```

Проблема в том, что названия полей из InfluxDB длинные (`value_brightness_properties_value` и т.д.) — они отображаются как мелкие подписи. Нужно переименовать их в Flux-запросе. Вот исправленный вариант:

Лучше всего сделать **4 отдельных query** (A, B, C, D) в одной Stat-панели — по одному на каждый параметр:

**Query A — Яркость:**
```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Яркость (lm)"}))
```

**Query B — Мощность:**
```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_power_consumption_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Мощность (Вт)"}))
```

**Query C — Температура:**
```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_temperature_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Температура (°C)"}))
```

**Query D — Напряжение:**
```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_voltage_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Напряжение (В)"}))
```

Ключевое — строка `|> map(fn: (r) => ({r with _field: "..."}))` переименовывает поле, и Grafana покажет читаемое название вместо `value_brightness_properties_value`.

Также в настройках Stat-панели справа:
- **Text size** → увеличьте Title и Value
- **Text mode** → выберите `Value and name` чтобы имя отображалось крупно

### 2.6 Определение состояния (ON/OFF) — Stat

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> last()
  |> map(fn: (r) => ({r with _value: if r._value > 10.0 then 1.0 else 0.0}))
```

> [!TIP]
> В настройках Stat панели используйте Value Mapping: `1 → ON 🟢`, `0 → OFF 🔴`

---
