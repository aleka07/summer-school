Регаемся в Google Cloud
Google cloud дает 300 долларов на 90 дней
не нажимаем на активацию в правом верхнем углу, она нам не нужна
https://console.cloud.google.com
![[Снимок экрана 2026-04-08 в 16.38.24.png|677]]


Заходим в пойск и ищем compute Engine
![[Pasted image 20260408163914.png]]



Если эта штука вышла, жмем Go to settings
![[Pasted image 20260408163940.png]]
попросил пройти нас двухэтапную аутентефикацию, проходим ее




После того как добавили двухэтапную аутентефикацию, переходим еще раз в Comput Engine, и выбираем VM instances
![[Снимок экрана 2026-04-08 в 16.40.34.png]]


Тут снова вылезет че то, жмем Enable, и ждем
![[Pasted image 20260408164137.png]]
тут Enable надо нажать




Create Instance, после того как выбрали VM instances
![[Pasted image 20260408164546.png]]



Конфигурируем, как на скриншоте, название свое можно задать, Регион Warsaw, 
![[Pasted image 20260408165024.png]]




e2 server 4vCPU 16Gb memory для наших целей нужно
![[Pasted image 20260408165039.png|697]]




далее Os and Storage справа, Ubuntu 24.04 LTS, обязательно x86 вариант надо выбрать.
также 40гб памяти

![[Pasted image 20260408165343.png|697]]




no backups – у вас не будет шанса на ошибку :)
![[Pasted image 20260408165500.png|697]]



Выстваляем настройки Networking как на скриншоте. дальше по идее жмем create
![[Pasted image 20260408165706.png]]





получили external ip и наша Виртуальная машина готова! 
![[Pasted image 20260408165913.png]]
1) Для нормисов – просто жмем SSH справа от виртуальной машины. и все мы в терминале, мы прекрасны
![[Pasted image 20260409100714.png]]
2) для  the людей – 

дальше что делаем, Linux/MacOS для винды надо думать

Создаем SSH-ключ на своем компе (пароль на ключ можно не ставить, просто дважды жмем Enter):

`ssh-keygen -t rsa -f ~/.ssh/gcp_key -C "developerbeck5"` – 

вот здесь developerbeck5 это имя пользователя, его можно получить просто нажав на SSH и подключившись к VM через браузерную SSH. но это только для того что бы получить username

Браузерный SSH мы не уважаем

Выводим созданный публичный ключ на экран:

`cat ~/.ssh/gcp_key.pub`

Копируем весь выведенный текст (от `ssh-rsa` до `developerbeck5`).

Идем в панель Google Cloud -> Compute Engine -> Metadata -> вкладка SSH Keys. Жмем Edit -> Add item, вставляем скопированный ключ и сохраняем.

Настраиваем быстрый вход, чтобы не вводить IP каждый раз. Открываем конфиг на своем компе:

`nano ~/.ssh/config`

Вставляем туда эти настройки:

Plaintext

```
Host gcp
    HostName 34.118.106.123 (внешний ip сервера)
    User developerbeck5 (ваш username на сервере)
    IdentityFile ~/.ssh/gcp_key
```

Сохраняем (Ctrl+O, Enter) и закрываем редактор (Ctrl+X).

Подключаемся к серверу:

`ssh gcp`

При первом подключении терминал спросит, доверяем ли мы этому хосту. Обязательно пишем целиком `yes` и жмем Enter.

Мы на сервере. Для проверки ресурсов и процессов запускаем:

`htop`


Что бы подключится к Grafana надо добавить правило для фаервола VM (virtual machine)
У VM недостаточно прав (scopes) для управления фаерволом. Есть два варианта:

**Вариант 1 — Создать правило через GCP Console (проще всего):**

1. Откройте [GCP Console → VPC → Firewall](https://console.cloud.google.com/networking/firewalls)
2. Нажмите **Create Firewall Rule**
3. Name: `allow-nodeports`
4. Direction: **Ingress**
5. Targets: **All instances in the network**
6. Source IP ranges: `0.0.0.0/0`
7. Protocols and ports: **TCP → `30000-32767`**
8. Нажмите **Create**