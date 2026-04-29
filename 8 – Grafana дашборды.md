# День 8. Дашборд в Grafana

Сегодня мы собираем простой дашборд для нашей лампочки. Цель не в том, чтобы сделать красивую промышленную панель, а в том, чтобы увидеть весь путь данных: генератор отправляет значения, OpenEgiz обновляет цифровой двойник, InfluxDB хранит временные ряды, Grafana показывает графики.

Открываем Grafana и заходим в `Dashboards`.

![[Pasted image 20260428164128.png]]

Создаем новый дашборд для:

```text
summerschool:lightbulb-01
```

## Панель 1. Яркость

Первый график — яркость лампочки. Тип панели: `Time series`.

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "brightness")
```

По этому графику сразу видно, включена лампочка или нет, и как меняется яркость во времени.

## Панель 2. Потребление

Вторая панель — потребление энергии. Тип панели тоже `Time series`.

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_power_consumption_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "power_consumption")
```

Здесь логика простая: если лампочка горит, потребление есть. Если выключена, потребление падает почти до нуля.

## Панель 3. Температура

Третья панель — температура лампочки.

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_field"] == "value_temperature_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "temperature")
```

Температура нужна, чтобы показать, что цифровой двойник может хранить не только состояние `on/off`, но и физические параметры объекта.

## Панель 4. Текущие значения

Теперь делаем одну `Stat`-панель, где будут последние значения сразу по нескольким параметрам.

Лучше сделать четыре отдельных query в одной панели: `A`, `B`, `C`, `D`. Так мы сможем переименовать длинные поля из InfluxDB в нормальные человеческие названия.

Query A — яркость:

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Яркость (lm)"}))
```

Query B — мощность:

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_power_consumption_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Мощность (Вт)"}))
```

Query C — температура:

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_temperature_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Температура (°C)"}))
```

Query D — напряжение:

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> filter(fn: (r) => r["_field"] == "value_voltage_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Напряжение (В)"}))
```

Ключевая строка здесь:

```flux
|> map(fn: (r) => ({r with _field: "Яркость (lm)"}))
```

Она переименовывает поле. Без этого Grafana покажет длинное техническое имя вроде `value_brightness_properties_value`, и студентам будет сложно читать дашборд.

В настройках `Stat`-панели справа ставим:

```text
Text mode: Value and name
Title size: побольше
Value size: побольше
```

## Панель 5. Состояние ON/OFF

Последняя панель — простой статус лампочки. Берем яркость и превращаем ее в `1` или `0`: если яркость больше `10`, считаем, что лампочка включена.

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) => r["_field"] == "value_brightness_properties_value")
  |> filter(fn: (r) => r["thingId"] == "summerschool:lightbulb-01")
  |> last()
  |> map(fn: (r) => ({r with _value: if r._value > 10.0 then 1.0 else 0.0}))
```

В настройках `Value mapping` делаем:

```text
1 -> ON
0 -> OFF
```

В итоге у нас получается понятный дашборд из пяти элементов: яркость, потребление, температура, текущие значения и статус ON/OFF. Этого достаточно, чтобы показать слушателям, что цифровой двойник не просто создан в интерфейсе, а реально получает живые данные.
