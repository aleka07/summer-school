
Unity

качаем unity для нашей машины, для вашей ОС

![[Pasted image 20260427162628.png]]

скачали, теперь устанавливаем 


авторизуемся
![[Pasted image 20260427162747.png]]



Ждем загрузку Unity
![[Pasted image 20260427162904.png]]


а пока грузится unity сделаем 3д модельку нашей лампочки

заходим в нашего телеграм бота, он бесплатный @img_2_3d_bot

закидываем в него нашу картинку лампочки

скачали модельку проверели их в онлайн viewer 

https://3dviewer.net/ для obj

https://gltf-viewer.donmccurdy.com/ для glb


модели есть теперь идем в unity

заходим в install слева, жмем "Manage" дальше "add modules" и смотрим установлена ли Web Build Support поддержка, если нет то устанавливаем
![[Pasted image 20260427164014.png]]
![[Pasted image 20260427164110.png]]

слева Projects дальше New Project
![[Pasted image 20260429105831.png]]

создали 3д universal шаблон
![[Pasted image 20260429105847.png]]
Files надо выбрать вот это
![[Pasted image 20260427172033.png]]
вот здесьь свитчаемся на web
![[Pasted image 20260427172106.png]]

Player –> Publishing Settings –> Compression Formant: Disabled
![[Pasted image 20260427172220.png]]

снизу в окне Assets создадим папку models
![[Pasted image 20260427164641.png]]


закинем в нее наш obj файл лампочки (ссылку на наш объект лампочки)
![[Pasted image 20260427165235.png]]



нужно переимновать лампочку под thingid с графаны это (критично важно)
![[Pasted image 20260427165628.png]]
summerschool:lightbulb-01
в моем случае



сейчас создадим скрипт на C# (на него тоже ссылку надо дать)
![[Pasted image 20260427165718.png]]
дважды нажмите на созданный файл и он откроектся в вашем текстовом редакторе

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

перетягиваем наш код из Assets на наш объект в сцене

далее создаем text mesh pro, правая кнопка мыши на 3д объект слева, в списке

![[Pasted image 20260427170655.png]]

импортируем только верхний
![[Pasted image 20260427170313.png]]
Нажать Canvas слева в Hierarchy. В меню справа надо выбрать Render mode World Space. Также Event Camera точку справа нажать выбрать main camera
![[Pasted image 20260429160026.png]]


после этого Scale нашего Canvas делаем 0.01 по всем осям и ставим за нашей лампочкой
![[Pasted image 20260429160237.png]]




выбрали слева нашу лампу, справа надо заполнить Status Label, выбрав Text TMP который мы до этого создали только что
![[Pasted image 20260427171507.png]]
Далее сделаем анимацию для камеры. надо добавить два скрипта.

**ClickToZoom.cs**
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

**CameraNavigator.cs**
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

добавили эти два скрипта в наши Assets
![[Снимок экрана — 2026-04-29 в 16.06.27.png]]

Выбираем нашу лампу в Иерархии слева, далее add component –> Box Colider
![[Снимок экрана — 2026-04-29 в 16.07.59.png]]
![[Pasted image 20260429160858.png]]

Жмем 1ое, и далее натягиваем куб на нашу лампочку
![[Снимок экрана — 2026-04-29 в 16.09.19.png]]

Edit –> Project Settings
![[Pasted image 20260429161410.png]]


Player –> Other Settings –> Active input handiling –> Both
![[Снимок экрана — 2026-04-29 в 16.15.02.png]]
![[Снимок экрана — 2026-04-29 в 16.16.11.png]]

надо создать "Empty" внутри нашей лампочки: Hierarchy –> "summerschool:lightbulb-01" 
![[Pasted image 20260429161822.png]]


ОБЯЗАТЕЛЬНО выбрать пустой обеъкт слева в Иерархии 
![[Pasted image 20260429163351.png]]
В самой 3д сцене, ЗАЖАВ ПКМ (Правую Кнопку Мыши) летим ближе к объекту (используем WASD для управления). как только подлетели ближе и решили что ракурс удобный жмем шорткат:

- **Windows:** `Ctrl + Shift + F`
    
- **Mac:** `Cmd + Shift + F`

справа должны увидеть что координаты поменялись. на те на которые мы подлетели

объект переименовываем его в zoomed_view




далее, выбираем нашу лампу (шаг 1), перетаскиваем наш скрипт ClickToZoom (шаг 2), и выбираем ранее созданный zoomed_view на "My View Point"
![[Снимок экрана — 2026-04-29 в 16.11.25.png]]


Снова в нашей сцене зажав ПКМ отлетаем назад. дав больше места для камеры/обзора камеры. в Hierarchy слева уже глобально вне нашей лампы снова создаем Empty Object, назовем его global_view. выбрав его снова жмем шорткат

- **Windows:** `Ctrl + Shift + F`
    
- **Mac:** `Cmd + Shift + F`

![[Снимок экрана — 2026-04-29 в 16.25.33.png]]


Далее
1 – выбрали Main Camera
2 – перетянули скрипт из  "CameraNavigator" направо в компоненты Main Camera
3 – Выбрали Overview Point - global_view
![[Снимок экрана — 2026-04-29 в 16.27.11.png]]






Далее билдим наш проект

File –> Build Profile, Build и сохраняем там где нам удобно
![[Pasted image 20260427172315.png]]



заходим в наш билд и нам нужны 4 этих файла
![[Pasted image 20260427172758.png]]


___
2-ой день
![[Pasted image 20260427173232.png]]
через ssh от гугл cloude закидываем наши 4 файла в папку build в корне проекте OpenEgiz


после того как оно загрузилось надо выйтв в корень. через команду 
```bash
cd ..
```
пока мы не дойдем до ~$
![[Pasted image 20260427174920.png]]
пишем команду ls видим все наши файлы и двигаем их через mv
дальше через команду mv двигаем 
```bash
mv 1.framework.js 1.loader.js 1.wasm  test/openegiz/build/
```
названия могут быть другими но суть та же, 4 файла в test/openegiz/build/

далее пишем `make` и жмем Tab или просто пишем `make upload-build`  

```bash
developerbeck5@super:~/test/openegiz$ make 
copy-build     generate-data  status         upgrade        
endpoints      install        uninstall      upload-build   
developerbeck5@super:~/test/openegiz$ make upload-build 
Uploading to pod: openegiz-unity-webgl-server-55647b7cc5-hdxfb

  Uploading: 1.data
  Uploading: 1.framework.js
  Uploading: 1.loader.js
  Uploading: 1.wasm
  Uploading: WebGL Build.data
  Uploading: WebGL Build.framework.js
  Uploading: WebGL Build.loader.js
  Uploading: WebGL Build.wasm

Access links:
  http://10.186.0.2:30530/build/1.data
  http://10.186.0.2:30530/build/1.framework.js
  http://10.186.0.2:30530/build/1.loader.js
  http://10.186.0.2:30530/build/1.wasm
  http://10.186.0.2:30530/build/WebGL Build.data
  http://10.186.0.2:30530/build/WebGL Build.framework.js
  http://10.186.0.2:30530/build/WebGL Build.loader.js
  http://10.186.0.2:30530/build/WebGL Build.wasm
developerbeck5@super:~/test/openegiz$ 
```


![[Pasted image 20260427175119.png]]
идем в графану, создаем дашборд, Add visualisation, opentwins
![[Pasted image 20260427175228.png]]
выбираем Unity
![[Pasted image 20260427175307.png]]

снизу пишем вот это 
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


пишем адреса нашего билда в окно справа, Visualization Unity
![[Pasted image 20260427181903.png]]




Заполняем поля
![[Pasted image 20260427182032.png]]



надо быть очень внимательным на счет thingID имени потому что лишний пробел может все сломать


![[Pasted image 20260429110536.png]]