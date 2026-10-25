# Задание 2. WEB. Балансировка нагрузки

Работу выполнял на macOS, используя виртуальную машину с Ubuntu в Multipass.
В дополнение к пакетам из условия установил `dnsutils` для команды `dig` и `tshark` для просмотра HTTP-трафика.

## 1. Создание серверов backend

В домашнем каталоге создал две папки и страницы с текстом из условия:

```text
ubuntu@hw2:~$ mkdir -p ~/backend1 ~/backend2
ubuntu@hw2:~$ printf '<h1>Response from Backend Server 1</h1>\n' > ~/backend1/index.html
ubuntu@hw2:~$ printf '<h2>*** Response from Backend Server 2 ***</h2>\n' > ~/backend2/index.html
ubuntu@hw2:~$ ls -1 ~/backend1 ~/backend2
/home/ubuntu/backend1:
index.html

/home/ubuntu/backend2:
index.html
```

В первом терминале запустил сервер на порту `8081`:

```text
ubuntu@hw2:~$ cd ~/backend1
ubuntu@hw2:~/backend1$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
```

Во втором терминале запустил сервер на порту `8082`:

```text
ubuntu@hw2:~$ cd ~/backend2
ubuntu@hw2:~/backend2$ python3 -m http.server 8082
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
```

Проверил оба сервера из третьего терминала:

```text
ubuntu@hw2:~$ curl http://127.0.0.1:8081/
<h1>Response from Backend Server 1</h1>
ubuntu@hw2:~$ curl http://127.0.0.1:8082/
<h2>*** Response from Backend Server 2 ***</h2>
```

## 2. Балансировка через DNS

В `/etc/dnsmasq.conf` записал две A-записи для одного имени:

```text
ubuntu@hw2:~$ sudo cat /etc/dnsmasq.conf
listen-address=127.0.0.1
bind-interfaces
host-record=my-awesome-highload-app.local,127.0.0.1
host-record=my-awesome-highload-app.local,127.0.0.2
```

dnsmasq слушает `127.0.0.1:53`. Проверил настройку и запустил службу:

```text
ubuntu@hw2:~$ sudo dnsmasq --test
dnsmasq: syntax check OK.
ubuntu@hw2:~$ sudo systemctl restart dnsmasq
```

Дважды запросил A-записи через `dig`:

```text
ubuntu@hw2:~$ dig @127.0.0.1 my-awesome-highload-app.local A +noall +answer
my-awesome-highload-app.local. 0 IN A 127.0.0.1
my-awesome-highload-app.local. 0 IN A 127.0.0.2
ubuntu@hw2:~$ dig @127.0.0.1 my-awesome-highload-app.local A +noall +answer
my-awesome-highload-app.local. 0 IN A 127.0.0.2
my-awesome-highload-app.local. 0 IN A 127.0.0.1
```

В ответе две A-записи с TTL `0`. При повторном запросе их порядок изменился.
Какой адрес использовать, решает клиент, поэтому это не гарантирует строго равное число HTTP-запросов к серверам.

Для проверки отказа остановил backend2 во втором терминале и повторил запросы:

```text
ubuntu@hw2:~$ curl --max-time 2 http://127.0.0.1:8082/
curl: (7) Failed to connect to 127.0.0.1 port 8082 after 0 ms: Couldn't connect to server
ubuntu@hw2:~$ dig @127.0.0.1 my-awesome-highload-app.local A +noall +answer
my-awesome-highload-app.local. 0 IN A 127.0.0.1
my-awesome-highload-app.local. 0 IN A 127.0.0.2
```

Несмотря на отказ backend2, обе записи остались.
dnsmasq не проверяет работоспособность HTTP-серверов и не удаляет A-запись при их остановке.
Также A-запись содержит только IP-адрес, без порта.
Поэтому эти DNS-записи сами по себе не связывают `127.0.0.1` с портом `8081`, а `127.0.0.2` с портом `8082`.

После проверки снова запустил backend2:

```text
ubuntu@hw2:~/backend2$ python3 -m http.server 8082
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
```

## 3. Балансировка L4 с IPVS

На время этой проверки остановил установленный NGINX. Создал интерфейс `dummy1` и назначил ему виртуальный адрес:

```text
ubuntu@hw2:~$ sudo systemctl stop nginx
ubuntu@hw2:~$ sudo ip link add dummy1 type dummy
ubuntu@hw2:~$ sudo ip addr add 192.168.100.1/32 dev dummy1
ubuntu@hw2:~$ sudo ip link set dummy1 up
ubuntu@hw2:~$ ip -br -4 addr show dummy1
dummy1           UNKNOWN        192.168.100.1/32
```

Для TCP-порта `80` настроил виртуальный сервер с алгоритмом `rr` и двумя серверами назначения:

```text
ubuntu@hw2:~$ sudo ipvsadm -A -t 192.168.100.1:80 -s rr
ubuntu@hw2:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m
ubuntu@hw2:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m
ubuntu@hw2:~$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0
  -> 127.0.0.1:8082               Masq    1      0          0
```

`rr` означает round-robin: новые TCP-соединения по очереди направляются на серверы с одинаковым весом.
Параметр `-m` выбирает пересылку с изменением адреса и порта назначения. Все проверки выполнял внутри Ubuntu,
где находятся оба backend.

Выполнил шесть отдельных запросов:

```text
ubuntu@hw2:~$ for i in $(seq 1 6); do curl -sS --max-time 3 http://192.168.100.1/; done
<h2>*** Response from Backend Server 2 ***</h2>
<h1>Response from Backend Server 1</h1>
<h2>*** Response from Backend Server 2 ***</h2>
<h1>Response from Backend Server 1</h1>
<h2>*** Response from Backend Server 2 ***</h2>
<h1>Response from Backend Server 1</h1>
ubuntu@hw2:~$ sudo ipvsadm -Ln --stats --exact
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  192.168.100.1:80                    6       39       34     2532     3190
  -> 127.0.0.1:8081                      3       18       18     1188     1635
  -> 127.0.0.1:8082                      3       21       16     1344     1555
```

Ответы чередуются, а счётчик `Conns` показывает по три соединения на каждый backend.
Каждый вызов `curl` создавал новое соединение. IPVS распределяет именно соединения,
а не отдельные HTTP-запросы внутри одного соединения.

## 4. Балансировка L7 с NGINX

В `/etc/nginx/conf.d/hw2.conf` настроил основной сервер на `8081` и резервный на `8082`:

```text
ubuntu@hw2:~$ sudo cat /etc/nginx/conf.d/hw2.conf
log_format hw2 '$request_uri $status $upstream_addr $upstream_status';

upstream backend_pool {
    zone backend_pool 64k;
    server 127.0.0.1:8081 max_fails=7 fail_timeout=60s;
    server 127.0.0.1:8082 backup;
}

server {
    listen 127.0.0.1:8888;
    access_log /var/log/nginx/hw2-access.log hw2;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header X-high-load-test 123;
        proxy_next_upstream off;
    }
}
```

`max_fails=7` исключает основной сервер после семи неудачных попыток за `60` секунд. После этого он считается недоступным на `60` секунд.
`zone` делает счётчик ошибок общим для рабочих процессов NGINX. Это соответствует [описанию параметров NGINX](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server).

`proxy_next_upstream off` отключает повторную отправку того же запроса другому серверу. Это позволяет показать переключение именно после семи ошибок: первые семь запросов получат `502`, а следующий пойдёт на резервный сервер.
С повторной отправкой NGINX мог бы обратиться к резервному уже при первой ошибке основного (см. [proxy_next_upstream](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)).

Отключил стандартный сайт, проверил настройку и запустил NGINX:

```text
ubuntu@hw2:~$ sudo rm /etc/nginx/sites-enabled/default
ubuntu@hw2:~$ sudo nginx -t && sudo systemctl restart nginx
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
ubuntu@hw2:~$ sudo ss -lntp 'sport = :80 or sport = :8888'
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      511        127.0.0.1:8888      0.0.0.0:*    users:(("nginx",pid=3083,fd=6),("nginx",pid=3082,fd=6),("nginx",pid=3081,fd=6))
```

NGINX слушает только `127.0.0.1:8888`. При работающем основном сервере возвращается его страница:

```text
ubuntu@hw2:~$ curl http://127.0.0.1:8888/
<h1>Response from Backend Server 1</h1>
```

### Проверка заголовка и HTTP-ответа через tshark

В отдельном терминале запустил захват обмена между NGINX и backend1, затем повторил `curl http://127.0.0.1:8888/`.
Ниже фрагменты вывода с HTTP-запросом и ответом. Остальные строки опущены:

```text
ubuntu@hw2:~$ sudo tshark -i lo -f 'tcp port 8081' -d tcp.port==8081,http -Y http -O http
...
Frame 4: 183 bytes on wire (1464 bits), 183 bytes captured (1464 bits) on interface lo, id 0
...
Transmission Control Protocol, Src Port: 35120, Dst Port: 8081, Seq: 1, Ack: 1, Len: 117
Hypertext Transfer Protocol
    GET / HTTP/1.0\r\n
...
    X-high-load-test: 123\r\n
    Host: backend_pool\r\n
    Connection: close\r\n
    User-Agent: curl/8.5.0\r\n
    Accept: */*\r\n
...
Frame 8: 106 bytes on wire (848 bits), 106 bytes captured (848 bits) on interface lo, id 0
...
Transmission Control Protocol, Src Port: 8081, Dst Port: 35120, Seq: 186, Ack: 118, Len: 40
...
Hypertext Transfer Protocol
    HTTP/1.0 200 OK\r\n
...
    Server: SimpleHTTP/0.6 Python/3.12.3\r\n
    Date: Sun, 25 Oct 2026 15:56:08 GMT\r\n
    Content-type: text/html\r\n
    Content-Length: 40\r\n
...
    File Data: 40 bytes
...
^C2 packets captured
```

Заголовок `X-high-load-test: 123` присутствует в запросе к backend1 на порт `8081`. Его добавил NGINX, поскольку в команде `curl` такого заголовка не было.
В ответ пришёл `HTTP/1.0 200 OK`. Заголовок добавляется именно к запросу к backend, поэтому в ответе клиенту его наличие не требуется.

### Переключение после семи ошибок

В первом терминале остановил backend1. Выполнил девять запросов подряд, выводя только их номера и HTTP-коды:

```text
ubuntu@hw2:~$ for i in $(seq 1 9); do printf '%s: ' "$i"; curl -s -o /dev/null -w '%{http_code}\n' "http://127.0.0.1:8888/?test=$i"; done
1: 502
2: 502
3: 502
4: 502
5: 502
6: 502
7: 502
8: 200
9: 200
```

Проверил журнал: после семи ошибок адрес назначения меняется с `8081` на `8082`.
Код `502` здесь формирует сам NGINX, поскольку подключиться к backend1 не удалось:

```text
ubuntu@hw2:~$ sudo tail -n 9 /var/log/nginx/hw2-access.log
/?test=1 502 127.0.0.1:8081 502
/?test=2 502 127.0.0.1:8081 502
/?test=3 502 127.0.0.1:8081 502
/?test=4 502 127.0.0.1:8081 502
/?test=5 502 127.0.0.1:8081 502
/?test=6 502 127.0.0.1:8081 502
/?test=7 502 127.0.0.1:8081 502
/?test=8 200 127.0.0.1:8082 200
/?test=9 200 127.0.0.1:8082 200
ubuntu@hw2:~$ curl http://127.0.0.1:8888/
<h2>*** Response from Backend Server 2 ***</h2>
```

Первые семь попыток к backend1 закончились ошибкой подключения, и NGINX вернул `502`. Начиная с восьмого запроса использовался backend2.
Таким образом, переключение произошло после семи неудачных попыток, как требовалось в условии.
