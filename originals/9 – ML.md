# 9 день — ML сервис для цифрового двойника

В прошлые дни мы сделали важную вещь: мы научились подключать данные к OpenEgiz.

Сначала данные были симулированные:

```text
наш Python генератор → OpenEgiz → OpenTwins → InfluxDB → Grafana
```

Потом мы подключили внешний источник:

```text
TDengine public MQTT → наш bridge → OpenEgiz → OpenTwins → InfluxDB → Grafana
```

Теперь делаем следующий шаг.

Мы подключаем **внешний ML сервис**.

Важно: ML сервис не обязан быть частью OpenEgiz. Он может жить на другом сервере, быть написан на Python, использовать любую ML библиотеку, брать данные из InfluxDB, из API, из Kafka, откуда угодно.

Логика такая:

```text
OpenEgiz хранит данные цифрового двойника
  ↓
ML сервис читает эти данные
  ↓
ML сервис считает прогноз / риск / аномалию
  ↓
ML сервис отправляет результат обратно в OpenEgiz
  ↓
Grafana показывает не только телеметрию, но и ML-инсайты
```

То есть сегодня у нас будет не просто “график мощности”, а уже аналитика:

```text
станция работает нормально?
есть ли аномалия?
какой риск?
что рекомендует ML?
какая мощность ожидается дальше?
```

---

## Главная идея

OpenEgiz — это платформа цифровых двойников.

ML сервис — это внешний аналитический сервис.

Он не заменяет OpenEgiz. Он добавляет новые данные в цифровой двойник.

```text
ваш физический источник данных → features устройства
ваш ML сервис → ML features устройства
```

Например, у нас есть twin:

```text
summerschool:solar-site-001
```

У него уже есть обычные features:

```text
ac_power_mw
expected_power_mw
performance_ratio
availability_percent
curtailment_percent
energy_today_mwh
poa_irradiance_wm2
active_alarms
```

А сегодня мы добавим ML features:

```text
ml_predicted_power_mw
ml_expected_power_gap_mw
ml_anomaly_score
ml_health_status
ml_risk_level
ml_recommendation
ml_model_version
ml_last_run
```

И это уже будут не сырые данные с датчика, а результат анализа.

---

## Почему это можно назвать ML

В реальном проекте тут могла бы быть большая модель:

- Random Forest
- XGBoost
- LSTM
- Prophet
- нейросеть
- внешний ML API
- отдельный сервер с GPU

Но на летней школе нам важнее понять архитектуру.

Поэтому мы сделаем легкий ML-like сервис:

1. Он читает последние данные солнечной станции из InfluxDB.
2. Считает predicted power.
3. Сравнивает actual power и expected power.
4. Считает anomaly score.
5. Определяет health status.
6. Пишет результат обратно в OpenEgiz.

Это нормальный учебный формат. Мы не пытаемся за 30 минут обучить промышленную модель, мы показываем как ML сервис интегрируется с цифровым двойником.

---

## Архитектура

```text
TDengine MQTT
  ↓
tdengine_solar_bridge.py
  ↓
OpenEgiz Mosquitto
  ↓
Ditto / OpenTwins
  ↓
InfluxDB
  ↓
ml_solar_service.py
  ↓
OpenEgiz Mosquitto
  ↓
Ditto / OpenTwins
  ↓
Grafana dashboard
```

Обратите внимание: ML сервис отправляет данные обратно так же, как любое устройство.

Для OpenEgiz это просто еще один источник данных.

Только этот источник не датчик, а аналитика.

---

## 1. Проверяем, что solar data уже идет

Перед ML днем должен быть запущен bridge из 7-го дня.

На сервере:

```bash
cd ~/test/openegiz/data-generator
source venv/bin/activate
python3 tdengine_solar_bridge.py --site-id all
```

Если он уже запущен в фоне, просто смотрим лог:

```bash
tail -f tdengine_solar_bridge.log
```

Проверяем, что данные есть в Ditto:

```bash
curl -s -u ditto:ditto \
  http://localhost:30525/api/2/things/summerschool:solar-site-001/features \
  | python3 -m json.tool
```

Если видим `ac_power_mw`, `performance_ratio`, `active_alarms`, значит можно подключать ML.

---

## 2. Добавляем ML features в OpenTwins

Заходим:

```text
OpenTwins → Twins → summerschool:solar-site-001
```

Добавляем features:

```text
ml_predicted_power_mw
ml_expected_power_gap_mw
ml_anomaly_score
ml_health_status
ml_risk_level
ml_recommendation
ml_model_version
ml_last_run
```

То же самое можно добавить для:

```text
summerschool:solar-site-002
summerschool:solar-site-003
summerschool:solar-site-004
summerschool:solar-site-005
```

Если не добавить руками, Ditto все равно может создать features через MQTT message, но для занятия лучше показать это в интерфейсе, чтобы люди понимали структуру twin.

---

## 3. Устанавливаем зависимости

На сервере:

```bash
cd ~/test/openegiz/data-generator
source venv/bin/activate
pip install influxdb-client paho-mqtt
```

---

## 4. Создаем ML сервис

Создаем файл:

```bash
nano ml_solar_service.py
```

Вставляем код:

```python
#!/usr/bin/env python3
import argparse
import json
import math
import random
import signal
import sys
import time
from datetime import datetime, timezone

import paho.mqtt.client as mqtt
from influxdb_client import InfluxDBClient


DEFAULT_INFLUX_URL = "http://localhost:30716"
DEFAULT_INFLUX_ORG = "opentwins"
DEFAULT_INFLUX_BUCKET = "default"
DEFAULT_INFLUX_TOKEN = "Hjh3ysMQ6evK=qqpFSYqn-s3JGovJLfHxyCDM=eNNZkdM-uuro93dNtJcodejLYYob2geKQ/29z3Kxui=y6FlL?dZeU9EFRxrYn284V/kZG5==jxLVAMJrYOv?LF79ahwIbhvstMN6gmfQ3DH7/IzUB7VlBZK-cd8aN7YqiFrYRLkBUv7H0QkbqPxgf2dMgCMCwZaLMk9RUeMaBfx2lQ=Mq1EEJJw-Jp!BmpCDnhlc!6D22PaE=Y3sgWWNhRv8oP"

DEFAULT_MQTT_HOST = "localhost"
DEFAULT_MQTT_PORT = 30511

SITE_THINGS = [
    "summerschool:solar-site-001",
    "summerschool:solar-site-002",
    "summerschool:solar-site-003",
    "summerschool:solar-site-004",
    "summerschool:solar-site-005",
]


FIELDS = {
    "ac_power": "value_ac_power_mw_properties_value",
    "expected_power": "value_expected_power_mw_properties_value",
    "performance_ratio": "value_performance_ratio_properties_value",
    "availability": "value_availability_percent_properties_value",
    "curtailment": "value_curtailment_percent_properties_value",
    "energy_today": "value_energy_today_mwh_properties_value",
    "irradiance": "value_poa_irradiance_wm2_properties_value",
    "alarms": "value_active_alarms_properties_value",
}


def now_iso():
    return datetime.now(timezone.utc).isoformat()


def build_ditto_message(thing_id, features):
    namespace, name = thing_id.split(":", 1)

    return {
        "topic": f"{namespace}/{name}/things/twin/commands/modify",
        "path": "/features",
        "value": features,
    }


def feature(value, timestamp=None):
    return {
        "properties": {
            "value": value,
            "timestamp": timestamp or now_iso(),
        }
    }


class SolarMLService:
    def __init__(self, args):
        self.args = args
        self.running = True
        self.model_version = "summer-school-ml-v1"

        self.influx = InfluxDBClient(
            url=args.influx_url,
            token=args.influx_token,
            org=args.influx_org,
        )
        self.query_api = self.influx.query_api()

        self.mqtt = mqtt.Client(
            mqtt.CallbackAPIVersion.VERSION2,
            client_id=f"openegiz-ml-service-{random.randint(1000, 9999)}",
            protocol=mqtt.MQTTv5,
        )

    def connect(self):
        print(f"Подключаемся к OpenEgiz MQTT: {self.args.mqtt_host}:{self.args.mqtt_port}")
        self.mqtt.connect(self.args.mqtt_host, self.args.mqtt_port, keepalive=60)
        self.mqtt.loop_start()

    def read_latest_values(self, thing_id):
        field_filters = " or ".join([
            f'r["_field"] == "{field}"'
            for field in FIELDS.values()
        ])

        query = f'''
from(bucket: "{self.args.influx_bucket}")
  |> range(start: -10m)
  |> filter(fn: (r) => r["thingId"] == "{thing_id}")
  |> filter(fn: (r) => {field_filters})
  |> last()
'''

        result = {}

        tables = self.query_api.query(query, org=self.args.influx_org)
        for table in tables:
            for record in table.records:
                field = record.get_field()
                value = record.get_value()

                for short_name, influx_field in FIELDS.items():
                    if field == influx_field:
                        result[short_name] = value

        return result

    def predict(self, values):
        actual = float(values.get("ac_power", 0) or 0)
        expected = float(values.get("expected_power", actual) or actual)
        performance_ratio = float(values.get("performance_ratio", 1) or 1)
        availability = float(values.get("availability", 100) or 100)
        curtailment = float(values.get("curtailment", 0) or 0)
        alarms = float(values.get("alarms", 0) or 0)
        irradiance = float(values.get("irradiance", 0) or 0)

        # Учебная модель.
        # В реальном проекте тут может быть sklearn/xgboost/lstm/api запрос к ML серверу.
        irradiance_factor = min(max(irradiance / 1000.0, 0.0), 1.2)
        operational_factor = max(0.0, availability / 100.0)
        curtailment_factor = max(0.0, 1.0 - curtailment / 100.0)

        predicted_power = expected * operational_factor * curtailment_factor

        if irradiance > 0:
            predicted_power = (predicted_power * 0.75) + (expected * irradiance_factor * 0.25)

        gap = expected - actual
        relative_gap = abs(gap) / max(expected, 1.0)

        anomaly_score = 0.0
        anomaly_score += min(relative_gap * 1.8, 0.55)
        anomaly_score += max(0.0, 0.98 - performance_ratio) * 4.0
        anomaly_score += max(0.0, 98.0 - availability) / 100.0
        anomaly_score += min(curtailment / 100.0, 0.2)
        anomaly_score += min(alarms * 0.25, 0.5)
        anomaly_score = round(min(max(anomaly_score, 0.0), 1.0), 3)

        predicted_power = round(predicted_power, 3)
        gap = round(gap, 3)

        if anomaly_score >= 0.75 or alarms >= 2:
            health_status = "critical"
            risk_level = "high"
            recommendation = "Check inverter/string performance and active alarms"
        elif anomaly_score >= 0.45 or alarms >= 1:
            health_status = "warning"
            risk_level = "medium"
            recommendation = "Monitor power gap and inspect site if trend continues"
        else:
            health_status = "normal"
            risk_level = "low"
            recommendation = "No action required"

        return {
            "ml_predicted_power_mw": predicted_power,
            "ml_expected_power_gap_mw": gap,
            "ml_anomaly_score": anomaly_score,
            "ml_health_status": health_status,
            "ml_risk_level": risk_level,
            "ml_recommendation": recommendation,
            "ml_model_version": self.model_version,
            "ml_last_run": now_iso(),
        }

    def publish_ml_result(self, thing_id, prediction):
        features = {
            name: feature(value)
            for name, value in prediction.items()
        }

        message = build_ditto_message(thing_id, features)
        topic = f"telemetry/{thing_id}"

        result = self.mqtt.publish(topic, json.dumps(message), qos=0)
        result.wait_for_publish(timeout=5)

    def run_once(self, thing_id):
        values = self.read_latest_values(thing_id)

        if not values:
            print(f"{thing_id}: нет данных в InfluxDB за последние 10 минут")
            return

        prediction = self.predict(values)
        self.publish_ml_result(thing_id, prediction)

        print(
            f"{datetime.now().strftime('%H:%M:%S')} {thing_id} | "
            f"pred={prediction['ml_predicted_power_mw']} MW | "
            f"gap={prediction['ml_expected_power_gap_mw']} MW | "
            f"anomaly={prediction['ml_anomaly_score']} | "
            f"health={prediction['ml_health_status']} | "
            f"risk={prediction['ml_risk_level']}"
        )

    def run(self):
        self.connect()

        if self.args.thing_id == "all":
            things = SITE_THINGS
        else:
            things = [self.args.thing_id]

        try:
            while self.running:
                for thing_id in things:
                    self.run_once(thing_id)

                if self.args.once:
                    break

                time.sleep(self.args.interval)
        finally:
            self.stop()

    def stop(self):
        print("Останавливаем ML service...")
        self.mqtt.loop_stop()
        self.mqtt.disconnect()
        self.influx.close()


def main():
    parser = argparse.ArgumentParser(
        description="ML service: InfluxDB -> prediction/anomaly -> OpenEgiz/OpenTwins"
    )
    parser.add_argument("--thing-id", default="summerschool:solar-site-001")
    parser.add_argument("--interval", type=float, default=15)
    parser.add_argument("--once", action="store_true")

    parser.add_argument("--influx-url", default=DEFAULT_INFLUX_URL)
    parser.add_argument("--influx-org", default=DEFAULT_INFLUX_ORG)
    parser.add_argument("--influx-bucket", default=DEFAULT_INFLUX_BUCKET)
    parser.add_argument("--influx-token", default=DEFAULT_INFLUX_TOKEN)

    parser.add_argument("--mqtt-host", default=DEFAULT_MQTT_HOST)
    parser.add_argument("--mqtt-port", type=int, default=DEFAULT_MQTT_PORT)

    args = parser.parse_args()

    service = SolarMLService(args)

    def handle_stop(sig, frame):
        service.running = False

    signal.signal(signal.SIGINT, handle_stop)
    signal.signal(signal.SIGTERM, handle_stop)

    service.run()


if __name__ == "__main__":
    main()
```

Сохраняем:

```text
Ctrl+O → Enter → Ctrl+X
```

Делаем исполняемым:

```bash
chmod +x ml_solar_service.py
```

---

## 5. Запускаем ML один раз

Сначала запускаем только для одной станции:

```bash
python3 ml_solar_service.py --thing-id summerschool:solar-site-001 --once
```

Если все нормально, увидим:

```text
12:55:10 summerschool:solar-site-001 | pred=15.84 MW | gap=0.72 MW | anomaly=0.118 | health=normal | risk=low
```

Проверяем, что ML features появились в Ditto:

```bash
curl -s -u ditto:ditto \
  http://localhost:30525/api/2/things/summerschool:solar-site-001/features \
  | python3 -m json.tool
```

Должны появиться:

```text
ml_predicted_power_mw
ml_expected_power_gap_mw
ml_anomaly_score
ml_health_status
ml_risk_level
ml_recommendation
ml_model_version
ml_last_run
```

---

## 6. Запускаем ML постоянно

Для одной станции:

```bash
python3 ml_solar_service.py --thing-id summerschool:solar-site-001 --interval 15
```

Для всех 5 станций:

```bash
python3 ml_solar_service.py --thing-id all --interval 15
```

Запуск в фоне:

```bash
nohup python3 ml_solar_service.py --thing-id all --interval 15 > ml_solar_service.log 2>&1 &
```

Смотреть логи:

```bash
tail -f ml_solar_service.log
```

Остановить:

```bash
ps aux | grep ml_solar_service.py
kill PID
```

---

## 7. Что мы только что сделали

Мы сделали отдельный ML сервис.

Он:

1. Читает последние значения из InfluxDB.
2. Считает прогноз мощности.
3. Считает разницу между expected и actual.
4. Считает anomaly score.
5. Определяет статус: `normal`, `warning`, `critical`.
6. Отправляет результат обратно в OpenEgiz.

Для цифрового двойника это выглядит так, будто появился новый источник данных:

```text
источник 1: TDengine MQTT → физические параметры
источник 2: ML service → аналитические параметры
```

Это очень важная идея.

Цифровой двойник может собирать данные из разных источников:

- датчики
- SCADA
- MQTT
- ERP
- ML сервер
- API
- ручной ввод

Главное — все это приходит в один twin и становится частью общей картины.

---

## 8. Grafana dashboard для ML

Теперь строим дашборд, где есть уже не только телеметрия, но и ML.

Не надо делать много панелей. Для занятия лучше сделать маленький понятный dashboard.

Логика такая:

```text
что происходит с мощностью?
есть ли аномалия?
какой статус у объекта?
что говорит ML по всем станциям?
```

Итого делаем всего 4 панели:

```text
1. Time series: Actual vs Expected vs ML Prediction
2. Gauge: Anomaly Score
3. Stat: Health Status + Risk Level
4. Table: ML summary по всем станциям
```

---

## 8.1 Actual vs Expected vs ML Prediction

Тип панели:

```text
Time series
```

Query A — Actual:

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_ac_power_mw_properties_value")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({r with _field: "Actual Power"}))
```

Query B — Expected:

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_expected_power_mw_properties_value")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({r with _field: "Expected Power"}))
```

Query C — ML Prediction:

```flux
from(bucket: "default")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_ml_predicted_power_mw_properties_value")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({r with _field: "ML Prediction"}))
```

Unit:

```text
MW
```

Этот график показывает сразу три линии:

- что станция реально выдала
- что ожидалось
- что сказал ML сервис

---

## 8.2 Anomaly Score

Тип панели:

```text
Gauge или Stat
```

```flux
from(bucket: "default")
  |> range(start: -2m)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_ml_anomaly_score_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Anomaly Score"}))
```

Thresholds:

```text
0.00 - 0.45 green
0.45 - 0.75 yellow
0.75 - 1.00 red
```

Это самая понятная ML панель.

Если score низкий — все нормально.

Если score высокий — ML сервис считает, что поведение станции странное.

---

## 8.3 Health Status и Risk Level

Тип панели:

```text
Stat
```

Делаем две маленькие Stat панели рядом.

Health Status:

```flux
from(bucket: "default")
  |> range(start: -2m)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_ml_health_status_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Health Status"}))
```

Value mappings:

```text
normal   → NORMAL, green
warning  → WARNING, yellow
critical → CRITICAL, red
```

Risk Level:

```flux
from(bucket: "default")
  |> range(start: -2m)
  |> filter(fn: (r) => r["thingId"] == "summerschool:solar-site-001")
  |> filter(fn: (r) => r["_field"] == "value_ml_risk_level_properties_value")
  |> last()
  |> map(fn: (r) => ({r with _field: "Risk Level"}))
```

Value mappings:

```text
low    → LOW, green
medium → MEDIUM, yellow
high   → HIGH, red
```

---

## 8.4 Таблица ML по всем станциям

Если ML сервис запущен для всех:

```bash
python3 ml_solar_service.py --thing-id all --interval 15
```

Делаем table:

```flux
from(bucket: "default")
  |> range(start: -2m)
  |> filter(fn: (r) =>
       r["_field"] == "value_ml_predicted_power_mw_properties_value" or
       r["_field"] == "value_ml_expected_power_gap_mw_properties_value" or
       r["_field"] == "value_ml_anomaly_score_properties_value" or
       r["_field"] == "value_ml_health_status_properties_value" or
       r["_field"] == "value_ml_risk_level_properties_value" or
       r["_field"] == "value_ml_recommendation_properties_value"
  )
  |> last()
  |> pivot(rowKey:["thingId"], columnKey: ["_field"], valueColumn: "_value")
  |> keep(columns: [
       "thingId",
       "value_ml_predicted_power_mw_properties_value",
       "value_ml_expected_power_gap_mw_properties_value",
       "value_ml_anomaly_score_properties_value",
       "value_ml_health_status_properties_value",
       "value_ml_risk_level_properties_value",
       "value_ml_recommendation_properties_value"
  ])
```

Переименовываем колонки:

```text
value_ml_predicted_power_mw_properties_value   → Predicted MW
value_ml_expected_power_gap_mw_properties_value → Gap MW
value_ml_anomaly_score_properties_value        → Anomaly
value_ml_health_status_properties_value        → Health
value_ml_risk_level_properties_value           → Risk
value_ml_recommendation_properties_value       → Recommendation
```

Эта таблица заменяет отдельную панель `ML Recommendation`. Так проще: все рекомендации сразу видны в одном месте.

---

## 8.5 Как должен выглядеть dashboard

Я бы сделал так:

```text
Row 1:
[Anomaly Score] [Health Status] [Risk Level]

Row 2:
[Actual vs Expected vs ML Prediction]

Row 3:
[ML summary table по всем solar sites]
```

Этого достаточно, чтобы показать ML идею.

Не надо делать 10 графиков, потому что люди потеряются. Нам нужно показать главное:

```text
данные пришли → ML их обработал → результат вернулся в twin → Grafana это показала
```

---

## 9. Как объяснить слушателям

Можно сказать так:

> До этого цифровой двойник показывал нам, что происходит с объектом прямо сейчас.
>
> Теперь мы добавили ML сервис, который пытается понять, нормально ли это поведение.
>
> То есть цифровой двойник становится не просто витриной данных, а точкой, куда собираются телеметрия, аналитика, прогнозы и рекомендации.

Еще важная мысль:

> В реальном проекте ML сервис может быть полностью отдельной системой. OpenEgiz не заставляет вас писать ML внутри платформы. Вы можете подключить любой внешний ML backend, если он умеет читать данные и отправлять результат обратно.

---

## Итог

Сегодня мы сделали:

```text
цифровой двойник + внешний источник данных + внешний ML сервис + Grafana dashboard
```

Это уже похоже на реальный промышленный сценарий:

```text
объект генерирует данные
  ↓
цифровой двойник хранит текущее состояние
  ↓
ML сервис анализирует состояние
  ↓
Grafana показывает телеметрию, прогнозы, риски и рекомендации
```

Главная формула:

```text
ваш источник данных → ваш цифровой двойник
ваш ML сервис → умные features вашего цифрового двойника
```
