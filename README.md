# mtproto-docker-haproxy
# Гайд как сбалансировать нагрузку на mtproto (гайд c faketls и без него)
## 0. Если нету докера ставим скприптом на все ноды
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
```
## 1. Поднимаем в докере mtproto
### 1.1 Генерация единого секрета, по которому будут подключатся клиенты

```bash
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
P.S: `-p 443:443` порт, на который будет стучатся балансировщик. -p 8888:8888: Внутренний порт статистики (опционально).

---
## 2. Установка  и настройка HAProxy.
### 2.1 Установка
```bash
sudo apt update && sudo apt install haproxy -y
```
### 2.2 Настройка конфига c rounrobin
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
### 2.3 Если работает не очень стабильно, то можно использовать с leastconn.
сам файл будет выглядеть так:
```haproxy
frontend mtproto_in
    bind *:443
    mode tcp
    option tcplog
    default_backend mtproto_nodes

backend mtproto_nodes
    mode tcp
    balance leastconn
    option tcp-check
    server node_local 1.1.1.1:443 check inter 2s rise 2 fall 3
    server node2 2.2.2.2:443 check inter 2s rise 2 fall 3
    server node3 3.3.3.3:443 check inter 2s rise 2 fall 3
```
