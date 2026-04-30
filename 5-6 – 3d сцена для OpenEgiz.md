# День 5-6. 3D-сцена для OpenEgiz

Сегодня мы делаем 3D-представление нашего цифрового двойника. У нас уже есть двойник лампочки, генератор данных и Grafana. Теперь добавляем визуальную часть: Unity-сцену, которую потом встроим в OpenEgiz через Grafana.

Главная идея такая: цифровой двойник — это не только таблица параметров. Мы можем показать объект в 3D и рядом выводить его актуальные значения.

## Установка Unity

Сначала скачиваем Unity Hub под вашу операционную систему.

![[Pasted image 20260427162628.png]]

Устанавливаем Unity Hub и авторизуемся.

![[Pasted image 20260427162747.png]]

После этого ждем, пока Unity загрузится и установится.

![[Pasted image 20260427162904.png]]

Пока Unity устанавливается, подготовим 3D-модель нашей лампочки.

## Создание 3D-модели

Для учебного проекта нам не нужна идеальная промышленная 3D-модель. Нам достаточно модели, которая визуально похожа на лампочку и которую можно загрузить в Unity.

Я использую Telegram-бот:

```text
@img_2_3d_bot
```

В него отправляем картинку лампочки и получаем 3D-модель.

После скачивания модель лучше проверить в онлайн-viewer:

```text
OBJ: https://3dviewer.net/
GLB: https://gltf-viewer.donmccurdy.com/
```

Если модель открывается в viewer, значит дальше можно идти в Unity.

## Проверяем Web Build Support

Нам нужен WebGL build, потому что потом Unity-сцену мы будем открывать внутри Grafana/OpenEgiz.

В Unity Hub слева открываем `Installs`, нажимаем `Manage`, потом `Add modules` и проверяем, установлен ли `Web Build Support`.

![[Pasted image 20260427164014.png]]

![[Pasted image 20260427164110.png]]

Если Web Build Support не установлен, ставим его.

## Создаем Unity-проект

Слева открываем `Projects`, нажимаем `New Project`.

![[Pasted image 20260429105831.png]]

Выбираем 3D Universal шаблон.

![[Pasted image 20260429105847.png]]

После создания проекта открываем настройки билда.

![[Pasted image 20260427172033.png]]

Переключаем платформу на Web.

![[Pasted image 20260427172106.png]]

Дальше идем в:

```text
Player -> Publishing Settings -> Compression Format
```

Ставим:

```text
Disabled
```

![[Pasted image 20260427172220.png]]

Это важно, чтобы потом WebGL build нормально отдавался с нашего сервера.

## Импорт модели

Внизу в `Assets` создаем папку:

```text
models
```

![[Pasted image 20260427164641.png]]

Перетаскиваем туда наш `.obj` файл лампочки.

![[Pasted image 20260427165235.png]]

Теперь добавляем модель на сцену.

Самый критичный момент: объект в Unity нужно назвать так же, как называется цифровой двойник.

В нашем случае:

```text
summerschool:lightbulb-01
```

![[Pasted image 20260427165628.png]]

Если в названии будет лишний пробел или другая буква, связка может не сработать. Поэтому `thingId` копируем внимательно.

## Скрипт приема данных

Теперь создаем C# script. Назовем его:

```text
UniversalReceiver
```

![[Pasted image 20260427165718.png]]

Открываем созданный файл и вставляем код:

```C#
using UnityEngine;
using System.Collections.Generic;
using TMPro;
using System.Globalization;

public class UniversalReceiver : MonoBehaviour
{
    public TMP_Text statusLabel;

void Start()
    {
        // Автопоиск текста, если забыли привязать в инспекторе
        if (statusLabel == null) statusLabel = GetComponentInChildren<TMP_Text>();
    }

    public void SetValues(string jsonString)
    {
        string cleanJson = ExtractInnerJSON(jsonString);
        if (string.IsNullOrEmpty(cleanJson)) return;

        Dictionary<string, string> dataDict = ParseSimpleJSON(cleanJson);
        BuildTable(dataDict);
    }

    // --- ЛОГИКА ОТОБРАЖЕНИЯ ---
    void BuildTable(Dictionary<string, string> data)
    {
        if (statusLabel == null) return;
        string finalText = "";

        foreach (var entry in data)
        {
            string rawKey = entry.Key;
            string rawValue = entry.Value;

            // Пропускаем технические поля
            if (rawKey.StartsWith("_") || rawKey == "result" || rawKey == "table" || rawKey == "topic" || rawKey == "host")
                continue;

            string unit = "";
            string prettyKey = BeautifyKey(rawKey, out unit);

            string formattedValue = rawValue;
            string valueColor = "#00FF66"; // Зеленый по умолчанию

            // Если это число, округляем его до 2 знаков и проверяем на пороги
            if (float.TryParse(rawValue, NumberStyles.Float, CultureInfo.InvariantCulture, out float numericValue))
            {
                formattedValue = numericValue.ToString("0.00");
                valueColor = GetValueColor(rawKey, numericValue); // Запрашиваем цвет (светофор)
            }

            // Собираем строку:
            // 1. Белый ключ (#FFFFFF)
            // 2. Жесткое выравнивание цифр на 65% ширины поля (<pos=65%>)
            // 3. Цветное жирное значение
            // 4. Серая единица измерения, уменьшенная до 80% размера
            finalText += $"<color=#FFFFFF>{prettyKey}</color> <pos=65%><b><color={valueColor}>{formattedValue}</color></b> <color=#AAAAAA><size=80%>{unit}</size></color>\n";
        }

        statusLabel.text = finalText;
    }

    // --- ЛОГИКА ПОРОГОВ (РЕАЛЬНЫЕ ПРОМЫШЛЕННЫЕ ЗНАЧЕНИЯ) ---
    string GetValueColor(string key, float value)
    {
        string k = key.ToLower();

        string colorGreen = "#00FF66";  // Норма (ОК)
        string colorYellow = "#FFD700"; // Внимание (Предупреждение)
        string colorRed = "#FF3333";    // Авария (Критично)

        // 1. НАПРЯЖЕНИЕ ФАЗНОЕ (Voltage)
        // Номинал 230В. Норма +-5%, предел +-10% (IEC 60038)
        if (k.Contains("voltage"))
        {
            if (value > 253f || value < 207f) return colorRed;
            if (value > 241.5f || value < 218.5f) return colorYellow;
            return colorGreen;
        }

        // 2. ЧАСТОТА (Frequency)
        // Для сети 50 Гц: жесткие нормы стабильности
        else if (k.Contains("frequency"))
        {
            if (value > 50.4f || value < 49.6f) return colorRed;
            if (value > 50.2f || value < 49.8f) return colorYellow;
            return colorGreen;
        }

        // 3. КОЭФФИЦИЕНТ МОЩНОСТИ (Power Factor Total)
        else if (k.Contains("power_factor"))
        {
            if (value < 0.6f) return colorRed; // Слишком низкий (холостой ход или проблема)
            if (value < 0.85f) return colorYellow; // Ниже нормы, штрафы за реактивку
            return colorGreen;
        }

        // 4. ТОК (Current)
        // Настроено на основе ваших логов (пик ~26А). 
        else if (k.Contains("current"))
        {
            if (value > 40f) return colorRed;    // Риск срабатывания автомата
            if (value > 30f) return colorYellow; // Повышенная нагрузка
            return colorGreen;
        }

        // 5. АКТИВНАЯ МОЩНОСТЬ (Active Power) в Ваттах
        // Лимиты под мотор мощностью около 10 кВт
        else if (k.Contains("active_power") || k.Contains("active power"))
        {
            if (value > 14000f) return colorRed; // Выше 14 кВт - жесткий перегруз
            if (value > 11000f) return colorYellow;
            return colorGreen;
        }

        // Остальные параметры (накопленная энергия kWh, varh) всегда зеленого цвета
        return colorGreen;
    }

    // --- ФУНКЦИЯ ДЛЯ КРАСОТЫ ТЕКСТА ---
    string BeautifyKey(string key, out string unit)
    {
        unit = "";
        string k = key.ToLower();

        // Убираем мусорные приставки
        k = k.Replace("value_", "").Replace("_properties_value", "");

        // Сокращаем длинные слова, чтобы они не наезжали на цифры
        k = k.Replace("_positive", " Pos");
        k = k.Replace("_negative", " Neg");

        // Вытаскиваем известные единицы измерения с конца строки
        if (k.EndsWith("_kwh")) { unit = "kWh"; k = k.Substring(0, k.Length - 4); }
        else if (k.EndsWith("_varh")) { unit = "varh"; k = k.Substring(0, k.Length - 5); }
        else if (k.EndsWith("_hz")) { unit = "Hz"; k = k.Substring(0, k.Length - 3); }
        else if (k.EndsWith("_w")) { unit = "W"; k = k.Substring(0, k.Length - 2); }
        else if (k.EndsWith("_v")) { unit = "V"; k = k.Substring(0, k.Length - 2); }
        else if (k.EndsWith("_a")) { unit = "A"; k = k.Substring(0, k.Length - 2); }

        // Заменяем подчеркивания на пробелы
        k = k.Replace("_", " ");

        // Делаем каждое слово с большой буквы (Title Case)
        string[] words = k.Split(' ');
        for (int i = 0; i < words.Length; i++)
        {
            if (words[i].Length > 0)
                words[i] = char.ToUpper(words[i][0]) + words[i].Substring(1);
        }

        return string.Join(" ", words);
    }

    // --- ПАРСЕР JSON ---
    string ExtractInnerJSON(string json)
    {
        int start = json.LastIndexOf("{");
        int end = json.IndexOf("}", start);
        if (start != -1 && end != -1)
        {
            return json.Substring(start + 1, end - start - 1);
        }
        return json.Replace("{", "").Replace("}", "").Replace("[", "").Replace("]", "");
    }

    Dictionary<string, string> ParseSimpleJSON(string jsonBody)
    {
        var result = new Dictionary<string, string>();
        string[] pairs = jsonBody.Split(',');

        foreach (string pair in pairs)
        {
            string[] parts = pair.Split(':');
            if (parts.Length >= 2)
            {
                string key = parts[0].Trim().Trim('\"');
                string val = parts[1].Trim().Trim('\"');

                if (!string.IsNullOrEmpty(key))
                {
                    if (result.ContainsKey(key)) result[key] = val;
                    else result.Add(key, val);
                }
            }
        }
        return result;
    }
}
```

Теперь перетаскиваем этот скрипт из `Assets` на объект лампочки в сцене.

## Текст рядом с объектом

Создаем `TextMeshPro`, чтобы показывать значения рядом с моделью.

Правая кнопка мыши по 3D-объекту слева в `Hierarchy`, дальше выбираем текстовый объект.

![[Pasted image 20260427170655.png]]

Если Unity попросит импортировать TMP Essentials, импортируем верхний вариант.

![[Pasted image 20260427170313.png]]

Выбираем `Canvas` слева в `Hierarchy`. Справа ставим:

```text
Render Mode: World Space
Event Camera: Main Camera
```

![[Pasted image 20260429160026.png]]

После этого у `Canvas` ставим `Scale`:

```text
X: 0.01
Y: 0.01
Z: 0.01
```

И размещаем текст за нашей лампочкой или рядом с ней.

![[Pasted image 20260429160237.png]]

Теперь выбираем лампу, на которую уже добавили `UniversalReceiver`, и в поле `Status Label` перетаскиваем созданный `Text TMP`.

![[Pasted image 20260427171507.png]]

## Камера и zoom

Теперь сделаем простую навигацию камеры: кликаем по лампочке, камера приближается; нажимаем правую кнопку мыши или `Esc`, камера возвращается назад.

Создаем два скрипта.

`ClickToZoom.cs`:

```cs
using UnityEngine;

public class ClickToZoom : MonoBehaviour
{
    public Transform myViewPoint; // Сюда привяжем View_Oven1 или View_Oven2
    private CameraNavigator camNav;

    void Start()
    {
        // Находим камеру автоматически
        camNav = Camera.main.GetComponent<CameraNavigator>();
    }

    // Эта функция срабатывает сама, когда вы кликаете мышкой по объекту (нужен Collider!)
    void OnMouseDown()
    {
        if (myViewPoint != null && camNav != null)
        {
            camNav.FlyTo(myViewPoint);
        }
    }
}
```

`CameraNavigator.cs`:

```cs
using UnityEngine;

public class CameraNavigator : MonoBehaviour
{
    public Transform overviewPoint; // Сюда привяжем View_Overview
    public float flySpeed = 5.0f;   // Скорость полета

    private Transform currentTarget; // Куда мы летим прямо сейчас

    void Start()
    {
        // При старте целью является общий обзор
        currentTarget = overviewPoint;
    }

    void Update()
    {
        if (currentTarget == null) return;

        // --- ПЛАВНЫЙ ПОЛЕТ (Lerp) ---
        // Двигаем позицию
        transform.position = Vector3.Lerp(transform.position, currentTarget.position, Time.deltaTime * flySpeed);
        
        // Двигаем поворот (Rotation)
        transform.rotation = Quaternion.Lerp(transform.rotation, currentTarget.rotation, Time.deltaTime * flySpeed);

        // --- ВОЗВРАТ НАЗАД ---
        // Если нажали Правую Кнопку Мыши (или Esc), возвращаемся на общий вид
        if (Input.GetMouseButtonDown(1) || Input.GetKeyDown(KeyCode.Escape))
        {
            GoToOverview();
        }
    }

    // Метод, который мы будем вызывать, чтобы полететь к печке
    public void FlyTo(Transform newTarget)
    {
        currentTarget = newTarget;
    }

    public void GoToOverview()
    {
        currentTarget = overviewPoint;
    }
}
```

Добавляем оба скрипта в `Assets`.

![[Снимок экрана — 2026-04-29 в 16.06.27.png]]

## Collider на лампочке

Чтобы клик по объекту работал, у модели должен быть collider.

Выбираем лампу в `Hierarchy`, справа нажимаем:

```text
Add Component -> Box Collider
```

![[Снимок экрана — 2026-04-29 в 16.07.59.png]]

![[Pasted image 20260429160858.png]]

После этого настраиваем размер collider так, чтобы он покрывал лампочку.

![[Снимок экрана — 2026-04-29 в 16.09.19.png]]

Дальше открываем:

```text
Edit -> Project Settings
```

![[Pasted image 20260429161410.png]]

В `Player -> Other Settings -> Active Input Handling` ставим:

```text
Both
```

![[Снимок экрана — 2026-04-29 в 16.15.02.png]]

![[Снимок экрана — 2026-04-29 в 16.16.11.png]]

## Точка приближения

Теперь создаем `Empty` внутри объекта лампочки:

```text
Hierarchy -> summerschool:lightbulb-01 -> Create Empty
```

![[Pasted image 20260429161822.png]]

Обязательно выбираем этот пустой объект слева в `Hierarchy`.

![[Pasted image 20260429163351.png]]

В сцене зажимаем правую кнопку мыши и подлетаем ближе к объекту. Для движения используем `WASD`. Когда нашли удобный ракурс, нажимаем:

```text
Windows: Ctrl + Shift + F
Mac: Cmd + Shift + F
```

После этого справа должны измениться координаты выбранного empty object. Переименовываем его:

```text
zoomed_view
```

Теперь выбираем лампу, добавляем на нее скрипт `ClickToZoom`, и в поле `My View Point` перетаскиваем `zoomed_view`.

![[Снимок экрана — 2026-04-29 в 16.11.25.png]]

## Общий вид камеры

Теперь в сцене отлетаем назад, чтобы был общий вид. В `Hierarchy` создаем еще один `Empty Object`, но уже глобально, не внутри лампы.

Называем его:

```text
global_view
```

Снова выбираем удобный общий ракурс и нажимаем:

```text
Windows: Ctrl + Shift + F
Mac: Cmd + Shift + F
```

![[Снимок экрана — 2026-04-29 в 16.25.33.png]]

Дальше:

1. Выбираем `Main Camera`.
2. Перетаскиваем скрипт `CameraNavigator` в компоненты камеры.
3. В `Overview Point` выбираем `global_view`.

![[Снимок экрана — 2026-04-29 в 16.27.11.png]]

## Сборка WebGL

Теперь билдим проект.

Открываем:

```text
File -> Build Profile -> Build
```

И сохраняем build в удобную папку.

![[Pasted image 20260427172315.png]]

После сборки открываем папку build. Нам нужны эти файлы:

![[Pasted image 20260427172758.png]]

Обычно это:

```text
.data
.framework.js
.loader.js
.wasm
```

Названия могут отличаться, но смысл один: эти файлы нужно загрузить на сервер OpenEgiz.

## Загрузка build на сервер

Через SSH или Google Cloud upload закидываем файлы build на сервер.

![[Pasted image 20260427173232.png]]

После загрузки выходим в домашнюю директорию, пока не окажемся примерно здесь:

```bash
cd ..
```

![[Pasted image 20260427174920.png]]

Командой `ls` проверяем, что файлы на месте. Потом переносим их в папку `build` внутри OpenEgiz:

```bash
mv 1.framework.js 1.loader.js 1.wasm test/openegiz/build/
```

Если у вас есть еще `.data` файл, его тоже переносим туда же:

```bash
mv 1.data test/openegiz/build/
```

Названия файлов могут быть другими. Главное, чтобы все файлы WebGL build оказались здесь:

```text
test/openegiz/build/
```

Дальше переходим в проект OpenEgiz и загружаем build:

```bash
cd test/openegiz
make upload-build
```

Пример успешного вывода:

```bash
developerbeck5@super:~/test/openegiz$ make upload-build 
Uploading to pod: openegiz-unity-webgl-server-55647b7cc5-hdxfb

  Uploading: 1.data
  Uploading: 1.framework.js
  Uploading: 1.loader.js
  Uploading: 1.wasm

Access links:
  http://10.186.0.2:30530/build/1.data
  http://10.186.0.2:30530/build/1.framework.js
  http://10.186.0.2:30530/build/1.loader.js
  http://10.186.0.2:30530/build/1.wasm
```

![[Pasted image 20260427175119.png]]

Эти ссылки потом понадобятся в Grafana, чтобы Unity-панель поняла, где лежит WebGL build.

## Подключение Unity в Grafana

Теперь идем в Grafana, создаем дашборд, нажимаем `Add visualisation` и выбираем визуализацию OpenEgiz.

![[Pasted image 20260427175228.png]]

В типе визуализации выбираем `Unity`.

![[Pasted image 20260427175307.png]]

Внизу вставляем Flux-запрос, который берет последние значения лампочки:

```flux
from(bucket: "default")
  |> range(start: -30s)
  |> filter(fn: (r) =>
       r["_field"] == "value_brightness_properties_value" or
       r["_field"] == "value_power_consumption_properties_value" or
       r["_field"] == "value_voltage_properties_value" or
       r["_field"] == "value_temperature_properties_value"
  )
  |> last()
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> keep(columns: [
       "thingId",
       "value_brightness_properties_value",
       "value_power_consumption_properties_value",
       "value_voltage_properties_value",
       "value_temperature_properties_value"
  ])
```

Справа в настройках `Visualization Unity` заполняем адреса нашего build.

![[Pasted image 20260427181903.png]]

Заполняем поля:

![[Pasted image 20260427182032.png]]

Еще раз проверяем `thingId`. Он должен совпадать с названием объекта в Unity:

```text
summerschool:lightbulb-01
```

Лишний пробел, другая буква или другое имя объекта могут сломать отображение данных.

![[Pasted image 20260429110536.png]]

В конце у нас должна получиться 3D-сцена, которая открывается в Grafana и получает актуальные данные лампочки из OpenEgiz.
