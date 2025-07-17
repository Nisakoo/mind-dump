---
title: "Повышение личной безопасности в Интернете"
date: 2025-07-16T11:19:40+05:00
author: Nisakoo
draft: false
ShowToc: true
---

*Не является инвестиционной рекомендаций, делайте все на свой страх и риск!*

Все было выполенено на *Ubuntu*. Статья без конкретики.

## Выбор провайдера

Для начала необходимо арендовать сервер, провайдера найти легко, никого рекомендовать не буду. Скажу лишь, что можно взять самый дешевый облычный сервер, но обязательно с **публичным IP-адресом**, иначе доступ к нему будет через встроенную консоль на сайте провайдера - нам такое не надо.

Операционную систему выбираем **Ubuntu**. Обращу ваше внимание, что некоторые компании *сразу предлагают установиь необходимое ПО*, можно этим воспользоваться и, в целом, не читать большую часть моей писанины, но обязательно прочтите часть про настройку сервера (я, конечно, не шибко опытный в этом, но базовые вещи расскажу).

## Настройка сервера

### Cloud-init скрипт

Еще на момент покупки некоторые провайдеры предоставляют возможность указать **SSH-ключи** и/или **Cloud-init скрипт**, например, ~~я пользуюсь следующим~~:

```yml {linenos=true}
#cloud-config
users:
  - name: <Username>
    ssh_authorized_keys:
      - "Public SSH key"
    sudo: ALL=(ALL:ALL) ALL
    groups: sudo
    shell: /bin/bash
chpasswd:
  expire: true
  users:
    - name: <Username>
      password: changeme
      type: text
```

Он создает нового пользователя, устанавливает SSH-ключ для него и при первом заходе принудительно заставляет указать новый пароль.

{{< collapse summary="*UPD: Расширенный Cloud-init скрипт (теперь пользуюсь этим)*" >}}

```yml {linenos=true}
#cloud-config
users:
  # Создание нового пользователя
  - name: <Username>
    ssh_authorized_keys:
      - "Public SSH key"
    sudo: ALL=(ALL:ALL) ALL
    groups: sudo
    shell: /bin/bash
chpasswd:
  # Настройка пароля для пользователя
  expire: true
  users:
    - name: <Username>
      password: changeme
      type: text

runcmd:
  # Обновление и установка необходимых пакетов
  - apt-get update -y && apt-get upgrade -y
  - apt-get install -y curl ufw fail2ban
  # Настройка SSH
  - |
    if grep -qE '^#?Port ' /etc/ssh/sshd_config; then
      sed -i 's/^#\?Port .*/Port <SSH Port>/' /etc/ssh/sshd_config
    else
      echo 'Port <SSH Port>' >> /etc/ssh/sshd_config
    fi
  - |
    if grep -qE '^#?PasswordAuthentication ' /etc/ssh/sshd_config; then
      sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
    else
      echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config
    fi
  - |
    if grep -qE '^#?PermitRootLogin ' /etc/ssh/sshd_config; then
      sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
    else
      echo 'PermitRootLogin no' >> /etc/ssh/sshd_config
    fi
  - systemctl daemon-reload && systemctl restart ssh.socket && systemctl restart ssh
  # Настройка FireWall
  - ufw reset
  - ufw default deny incoming
  - ufw default allow outgoing
  - ufw allow <SSH port>/tcp
  - ufw enable
```

Создает пользователя, настраивает ему пароль и SSH-ключи. Обновляет и устанавливает пакеты, настраивает SSH и FireWall.

{{< /collapse >}}

### Получаем SSH-ключ

1. На рабочей машине:
```bash
ssh-keygen
```

2. Выводим публичный ключ:
```bash
cat ~/.ssh/id_rsa.pub
```

3. Копируем и вставляем его в Cloud-init скрипт, либо на удаленной машине:
```bash
echo "Public SSH key" >> ~/.ssh/authorized_keys
```

**Учтите**, что у `authorized_keys` права доступа должны быть `-rw-------`, для утсановки:
```bash
chmod 600 ~/.ssh/authorized_keys
```

Подключаемся так:
```bash
ssh <Username>@<Server IP>
```
Публичный IP-адрес узнаете на сайте провайдера

### Меняем настройки SSH

Никогда не стоит заходить из-под *root* дистанционно, да и вообще лучше отключить в принципе эту возможность.

Но если очень хочется, то логинетесь из-под своего основного пользователся и пишите в терминале `su root`, после вводите пароль *root*, творите грязь и выходите `exit`

На удаленной машине (используем текстовый редактор *nano*, не vim, но как выйти почитайте заранее):
```bash
sudo nano /etc/ssh/sshd_config
```

Ищем в файле строки и делаем так:
```{linenos=true}
PermitRootLogin no
PasswordAuthentication no
```
Запрещаем вход из-под *root*, запрещаем вход по паролю.

Перезапускаем SSH:
```bash
sudo systemctl restart ssh
```

Также иногда меняют порт SSH, можете почитать про это.

### Настраиваем FireWall

К этому пункту стоит вернутся в самом конце, но он обязателен. FireWall позволяет указать: кто может подключаться (подсети), куда могут подключаться (порты) и каким образом (протоколы).

Я обычно использую FireWall на сайте провайдера, там, как правило, написано, какие порты закрывать не стоит для корректного мониторинга, но если такой возможности нет, используем `ufw` (гайда на него тут не будет).

Не закрывайте порт, который используется для SSH, если же закрыли, то подключаться надо через консоль провайдера.

## Устанавливаем

Не забываем обновить пакеты:
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
```

Будем использовать *WireGuard Easy* (установится сам WireGuard и графическая панель управления для него), для этого надо сначала *[установить Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)*. После этого заходим на удаленный сервер и выполняем:

```bash
mkdir ~/wireguard-easy
touch ~/wireguard-easy/docker-compose.yml
```

Открываем файл и вставляем туда (взял из *[официального репозитория](https://github.com/wg-easy/wg-easy)* и немного изменил), не забудьте поменять пароль и указать IP-адресс сервера:
```yml {linenos=true}
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      - WG_HOST=<Server IP>
      - PASSWORD_HASH=<Password Hash>
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "127.0.0.1:51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1

volumes:
  etc_wireguard:
```

{{< collapse summary="*Получаем PASSWORD_HASH*" >}}
*PASSWORD_HASH* создаем [так](https://github.com/wg-easy/wg-easy/blob/v14/How_to_generate_an_bcrypt_hash.md). Важно убрать одинарные кавычки и продублировать `$` (экранировать)

То есть это:
```bash
PASSWORD_HASH='$2a$12$9Xb3xJrUXuCOfuvgYqRPYO4muK2TzNv3IsriEmNaZwgL.xivV6lgy'
```
Превращаем в это:
```bash
PASSWORD_HASH=$$2a$$12$$9Xb3xJrUXuCOfuvgYqRPYO4muK2TzNv3IsriEmNaZwgL.xivV6lgy
```

{{< /collapse >}}

Запускаем в фоне:
```bash
cd ~/wireguard-easy
sudo docker compose up -d
```

Чтобы остановить, пишем:
```bash
sudo docker compose down
```

**Важно!** Если вы будете использовать мой `docker-compose.yml` для запуска WireGuard Easy, то Web UI для настройки запуститься локально, она не будет доступна извне, открыть ее можно так:
```bash
ssh -N -L 8080:localhost:51821 <Your user name>@public-ip
```
В браузере переходим по `http://localhost:8080`

Эта команда открывает защищенный SSH-туннель между вашей машиной и удаленной.

Зачем так усложнять? Для безопасности. Так как по умолчанию используется незащищенное HTTP соединение, чтобы сделать его защищенным и получать досутп в браузере напрямую по IP-адресу сервера, нужно приложить больше усилий (как-нибудь не в этой статье).

В интерфейсе Web UI уже несложно разобраться. Скачайте приложение WireGuard с официального сайта, скачайте конфигурацию из панели управления и добавте новый туннель в WireGuard - готово!

## Виедо-инструкции

Настройка SSH:
{{< youtube IVHv3eVQa14 >}}

Общий туториал по настройкe удаленного сервера (на английском):
{{< youtube Q1Y_g0wMwww >}}
