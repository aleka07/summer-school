https://github.com/openegiz/openegiz – основная репа

Рассказать людям что такое openegiz, ознакомить с нашей репой. может добавить архитектуру нашу.

Теперь когда мы на vps будем устанавливать


На VPS создаем папку для тестов


```bash
mkdir test

# переходим в эту папку
cd test

# далее клонируем нашу репу
git clone https://github.com/openegiz/openegiz

# после клонирования переходим в нее
cd openegiz

# для удобства можно это все длать в IDE на ваш выбор
```

далее установка  Prerequisites

Docker
```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```


Set up Docker's `apt` repository.
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

To install the latest version, run:
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


Docker без sudo
```bash
sudo usermod -aG docker $USER
```
После этого **перелогиньтесь** (или выполните `newgrp docker` в текущем терминале), чтобы изменения вступили в силу. Затем `docker ps` будет работать без `sudo`.

Install k3s по Readme от openegiz

```bash
curl -sfL https://get.k3s.io | sh -

# Check for Ready node, takes ~30 seconds

sudo k3s kubectl get node
```


```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

Post install

```bash

mkdir -p ~/.kube

sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

sudo chown $(id -u):$(id -g) ~/.kube/config

```

Далее Helm

```bash

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4

chmod 700 get_helm.sh

./get_helm.sh
```

нужно установить make

```bash
sudo apt install make
```

запускаем установку OpenEgiz находясь в директории проекта. Установка может занять некоторое время. ждемс
```bash
make install
```



> **Примечание:** Во втором терминале можете открыть для мониторинга всех подов

While waiting for the installation to complete, you can monitor the pod statuses by running:
```bash
watch -n 1 kubectl get pods
```



> **Примечание:** При первой установке MongoDB может не успеть запуститься до ditto-extended-api. Из-за этого Grafana не сможет подключиться. Перезапустите под:

```bash
kubectl rollout restart deployment/openegiz-ditto-extended-api
```



Если что то пошло не так, делаем следющее
```bash
# 1. Удалить всё если ошибки вышли
make uninstall

# 2. Проверить, что подов не осталось
kubectl get pods

# 3. Запустить заново
make install

```



![[Pasted image 20260409101354.png]]


дальше заходим по public айпи 30718 порт
пример http://34.118.106.123:30718/a/ertis-opentwins-app/policies
![[Pasted image 20260409104529.png]]

логин/пароль admin/admin

надо изменить его в целях безопастности
http://34.118.106.123:30718/a/ertis-opentwins-app/policies