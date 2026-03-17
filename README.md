# mtproto-docker-haproxy
# Гайд как сбалансировать нагрузку на mtproto (гайд c faketls и без него)
## 0. Если нету докера ставим скриптом на все ноды
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
```
---
## 1. Поднимаем в докере MTProto
### 1.1 Генерация единого секрета, по которому будут подключатся клиенты

```bash
# на всякий случай ставим xxd не везде он есть
apt install xxd
head -c 16 /dev/urandom | xxd -ps
```
### 1.2 Запуск MTProto ноды (без faketls)
```bash
docker run -d --name mtproto-proxy --restart always \
  -p 443:443 -p 8888:8888 \
  -e SECRET=ТВОЙ_СЕКРЕТ_ИЗ_ШАГА_1.1 \
  -e AES_PWD=$(openssl rand -hex 16) \
  telegrammessenger/proxy:latest
```
### 1.3 Запуск MTProto ноды c faketls
Описание как это работает: 
Секрет для Fake-TLS состоит из трех частей:
    * Префикс ee (включает режим Fake-TLS).
    * Твой текущий секрет: ТВОЙ_СЕКРЕТ_ИЗ_ШАГА_1.3.1.
#### 1.3.1 Как сделать свой домен для Fake-TLS
Используем python (для примера взял `music.yandex.ru`)
```bash
docker run --rm nineseconds/mtg:2 generate-secret --hex music.yandex.ru
```
#### 1.3.2 Поднимаем MTProto
```bash
docker run -d \
  --name mtproto-proxy \
  --restart unless-stopped \
  -p 443:443 \
  nineseconds/mtg:2 \
  simple-run -n 1.1.1.1 -i prefer-ipv4 0.0.0.0:443 секрет_из_пункта_1.3.1
```
P.S: `-p 443:443` порт, на который будет стучатся балансировщик. -p 8888:8888: Внутренний порт статистики (опционально).

---
## 2. Установка  и настройка HAProxy.
### 2.1 Установка
```bash
sudo apt update && sudo apt install haproxy -y
```
### 2.2 Базовая Настройка конфига c rounrobin (Советую использовать настройку в пункте 2.3)
* В конец файла `/etc/haproxy/haproxy.cfg ` добавляем то что написано снизу, конечно же меняя ip нод на свои

то что нужно добавить в конец файла `/etc/haproxy/haproxy.cfg`: 
```haproxy
frontend mtproto_in
    bind *:443
    mode tcp
    option tcplog
    default_backend mtproto_nodes

backend mtproto_nodes
    mode tcp
    balance roundrobin
    # Проверка доступности порта (check)
    server node1 1.1.1.1:443 check
    server node2 2.2.2.2:443 check
    server node3 3.3.3.3:443 check
```
### 2.3 Немного хитрая настройка rounrobin (советую испоьзовать ее)
```haproxy
frontend mtproto_in
    bind *:443
    mode tcp
    option tcplog
    default_backend mtproto_nodes

backend mtproto_nodes
    mode tcp
    balance roundrobin
    # Проверка доступности каждые 2 секунды (inter 2s)
    # Если 2 проверки успешны — нода UP (rise 2)
    # Если 3 проверки провалены — нода DOWN (fall 3)
    server node1 1.1.1.1:443 check inter 2s rise 2 fall 3
    server node2 2.2.2.2:443 check inter 2s rise 2 fall 3
    server node3 3.3.3.3:443 check inter 2s rise 2 fall 3
```

#### 2.3.1 Если у вас есть основные и бекап ноды и хотите сбалансировать нагрузку, то исользуйте данный конфиг
```haproxy
frontend mtproto_in
    bind *:443
    mode tcp
    option tcplog
    default_backend mtproto_nodes

backend mtproto_nodes
    mode tcp
    balance roundrobin
    server node1 1.1.1.1:443 check weight 100 inter 2s rise 2 fall 3
    server node2 2.2.2.2:443 check weight 100 inter 2s rise 2 fall 3
    #Резервная нода, начнет работать когда упадут 1 и 2.
    server node3 3.3.3.3:443 check backup inter 2s rise 2 fall 3
```
p.s-2. bind *:443 тут вы указываете порт который буде открываться как раз с сервера на котором haproxy

### 2.4 Перезапускаете HAProxy
```bash
systemctl restart haproxy
```
---

### 3. Подключаемся 
https://t.me/proxy?server=ip-server&port=443&secret=Секрет_который_вы_сгенерировали_ранее
