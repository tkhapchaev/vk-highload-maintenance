# Задание 1. Сетевой стек

Работу выполнял на MacOS, используя виртуальную машину с Ubuntu в Multipass.
Сетевой интерфейс внутри виртуальной машины называется `enp0s1`, адрес - `192.168.252.2`.

## 1. Состояния TCP-соединений

В первом терминале подготовил страницу и запустил сервер:

```text
ubuntu@hw1:~$ mkdir -p ~/www
ubuntu@hw1:~$ cd ~/www
ubuntu@hw1:~/www$ printf 'Hello from hw1\n' > index.html
ubuntu@hw1:~/www$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Во втором терминале нашёл его слушающий сокет:

```text
ubuntu@hw1:~$ sudo ss -lntp 'sport = :8080'
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=3399,fd=3))
```

Подключился через `curl` и сразу посмотрел состояния соединений:

```text
ubuntu@hw1:~$ curl -i http://127.0.0.1:8080/ && ss -tano '( sport = :8080 or dport = :8080 )'
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Date: Thu, 22 Oct 2026 08:46:35 GMT
Content-type: text/html
Content-Length: 15
Last-Modified: Thu, 22 Oct 2026 08:36:35 GMT

Hello from hw1
State      Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN     0      5            0.0.0.0:8080      0.0.0.0:*
TIME-WAIT  0      0          127.0.0.1:8080    127.0.0.1:60992 timer:(timewait,59sec,0)
```

`LISTEN` означает, что сервер ждёт подключения. `TIME-WAIT` относится к уже закрытому соединению с клиентом на порту `60992`.
При этом слушающий сокет продолжает работать.

В этом запуске `TIME-WAIT` остался на стороне сервера. Обычно в это состояние переходит сторона, которая начинает закрытие соединения.
TCP некоторое время хранит его состояние: это защищает новое соединение от старых задержавшихся пакетов и позволяет повторно подтвердить пакет с флагом FIN, если последнее подтверждение потерялось.
Здесь оставшееся время - 59 секунд.

Эту запись хранит ядро, открытого сокета приложения за ней уже нет.
Поэтому завершением процесса её не убрать: штатно она исчезает по таймеру.
Принудительно обходить ожидание нельзя без риска нарушить правильную обработку старых пакетов.

Большое количество таких записей расходует память ядра.
При большом числе коротких исходящих соединений к одному серверу также могут закончиться доступные временные порты, и новые подключения начнут завершаться ошибками.

## 2. Динамическая маршрутизация с BIRD

Создал четыре интерфейса из условия и назначил адреса. При успешном выполнении команды `ip` ничего не выводят.

```text
ubuntu@hw1:~$ sudo ip link add service_0 type dummy && sudo ip addr add 192.168.14.88/32 dev service_0 && sudo ip link set service_0 up
ubuntu@hw1:~$ sudo ip link add service_1 type dummy && sudo ip addr add 192.168.14.1/30 dev service_1 && sudo ip link set service_1 up
ubuntu@hw1:~$ sudo ip link add service_2 type dummy && sudo ip addr add 192.168.10.4/32 dev service_2 && sudo ip link set service_2 up
ubuntu@hw1:~$ sudo ip link add srv_1 type dummy && sudo ip addr add 192.168.14.4/32 dev srv_1 && sudo ip link set srv_1 up
ubuntu@hw1:~$ ip -br -4 addr
lo               UNKNOWN        127.0.0.1/8
enp0s1           UP             192.168.252.2/24 metric 100
service_0        UNKNOWN        192.168.14.88/32
service_1        UNKNOWN        192.168.14.1/30
service_2        UNKNOWN        192.168.10.4/32
srv_1            UNKNOWN        192.168.14.4/32
```

В `/etc/bird/bird.conf` записал следующую настройку:

```text
ubuntu@hw1:~$ sudo cat /etc/bird/bird.conf
router id 192.168.14.88;

protocol device {
    scan time 1;
}

protocol direct {
    ipv4;
    interface "*";
}

filter service_routes {
    if (ifname ~ "service_*") && (net ~ [192.168.14.0/24{32,32}]) then accept;
    reject;
}

protocol rip rip_services {
    ipv4 {
        import none;
        export filter service_routes;
    };
    interface "enp0s1" {
        version 2;
    };
}
```

Фильтр проверяет имя `service_*` и адрес из `192.168.14.0/24` с маской строго `/32`.
Поэтому будущие подходящие адреса тоже будут анонсироваться. RIP v2 включён на основном интерфейсе `enp0s1`.

Проверил настройку, перезапустил BIRD и посмотрел, какие маршруты он анонсирует:

```text
ubuntu@hw1:~$ sudo bird -p -c /etc/bird/bird.conf && sudo systemctl restart bird
ubuntu@hw1:~$ sudo birdc show route export rip_services
BIRD 2.14 ready.
Table master4:
192.168.14.88/32     unicast [direct1 11:41:40.637] * (240)
        dev service_0
```

Проверил сами RIP-пакеты через `tcpdump`:

```text
ubuntu@hw1:~$ sudo tcpdump -ni enp0s1 -v -c 1 'udp port 520'
tcpdump: listening on enp0s1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:42:21.556312 IP (tos 0xc0, ttl 1, id 42757, offset 0, flags [none], proto UDP (17), length 52)
    192.168.252.2.520 > 224.0.0.9.520:
        RIPv2, Response, length: 24, routes: 1 or less
          AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
1 packet captured
1 packet received by filter
0 packets dropped by kernel
```

В пакете только `192.168.14.88/32`. Остальные адреса не проходят условия:

- `service_1` - маска `/30`, а нужна `/32`.
- `service_2` - адрес из другой подсети.
- `srv_1` - имя не начинается с `service_`.

Дополнительно добавил подходящий адрес `192.168.14.99/32`.
Без изменения настройки и перезапуска BIRD он появился в анонсе:

```text
ubuntu@hw1:~$ sudo ip link add service_3 type dummy && sudo ip addr add 192.168.14.99/32 dev service_3 && sudo ip link set service_3 up
ubuntu@hw1:~$ sudo tcpdump -ni enp0s1 -v -c 1 'udp port 520'
tcpdump: listening on enp0s1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:42:51.512449 IP (tos 0xc0, ttl 1, id 61117, offset 0, flags [none], proto UDP (17), length 72)
    192.168.252.2.520 > 224.0.0.9.520:
        RIPv2, Response, length: 44, routes: 2 or less
          AFI IPv4,   192.168.14.99/32, tag 0x0000, metric: 1, next-hop: self
          AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
1 packet captured
1 packet received by filter
0 packets dropped by kernel
ubuntu@hw1:~$ sudo ip link delete service_3
```

## 3. Настройка фаервола

Сервер Python продолжает работать в первом терминале. До установки правила запрос с Mac проходит:

```text
MacBook-Pro-Timur% curl http://192.168.252.2:8080/
Hello from hw1
```

В Ubuntu добавил правило, которое отбрасывает входящие TCP-пакеты на порт `8080`:

```text
ubuntu@hw1:~$ sudo nft add table inet hw1
ubuntu@hw1:~$ sudo nft 'add chain inet hw1 input { type filter hook input priority 0; policy accept; }'
ubuntu@hw1:~$ sudo nft add rule inet hw1 input tcp dport 8080 counter drop
```

Запустил `tcpdump` в Ubuntu и повторил запрос с Mac:

```text
MacBook-Pro-Timur% curl --connect-timeout 3 http://192.168.252.2:8080/
curl: (28) Failed to connect to 192.168.252.2 port 8080 after 3011 ms: Timeout was reached
```

Фрагмент вывода `tcpdump` - повторные пакеты между первой и последней строкой опущены:

```text
ubuntu@hw1:~$ sudo tcpdump -ni enp0s1 'tcp port 8080'
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:44:12.449697 IP 192.168.252.1.53741 > 192.168.252.2.8080: Flags [SEW], seq 375020319, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 4183136198 ecr 0,sackOK,eol], length 0
...
11:44:14.475518 IP 192.168.252.1.53741 > 192.168.252.2.8080: Flags [S], seq 375020319, win 65535, options [mss 1460,sackOK,eol], length 0
^C
11 packets captured
11 packets received by filter
0 packets dropped by kernel
```

Видны попытки открыть соединение с Mac (`192.168.252.1`), но ответов от сервера нет.
Правило `drop` молча отбрасывает пакеты, поэтому `curl` заканчивает работу по таймауту.

Счётчик правила подтверждает, что пакеты попали под блокировку:

```text
ubuntu@hw1:~$ sudo nft list table inet hw1
table inet hw1 {
        chain input {
                type filter hook input priority filter; policy accept;
                tcp dport 8080 counter packets 11 bytes 688 drop
        }
}
```

Сам сервер при этом остаётся запущен:

```text
ubuntu@hw1:~$ sudo ss -lntp 'sport = :8080'
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=3399,fd=3))
```

## 4. Аппаратное ускорение сетевого трафика

Посмотрел драйвер адаптера и первые 20 строк его возможностей:

```text
ubuntu@hw1:~$ sudo ethtool -i enp0s1 | head -n 3
driver: virtio_net
version: 1.0.0
firmware-version:
ubuntu@hw1:~$ sudo ethtool -k enp0s1 | head -n 20
Features for enp0s1:
rx-checksumming: on [fixed]
tx-checksumming: off
        tx-checksum-ipv4: off [fixed]
        tx-checksum-ip-generic: off [fixed]
        tx-checksum-ipv6: off [fixed]
        tx-checksum-fcoe-crc: off [fixed]
        tx-checksum-sctp: off [fixed]
scatter-gather: off
        tx-scatter-gather: off [fixed]
        tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: off
        tx-tcp-segmentation: off [fixed]
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp-mangleid-segmentation: off [fixed]
        tx-tcp6-segmentation: off [fixed]
generic-segmentation-offload: off [requested on]
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: off [fixed]
```

TCP segmentation offload (TSO) выключен: `tcp-segmentation-offload: off`.
Пометка `[fixed]` у его подфункций означает, что их нельзя переключить в текущем окружении.

TSO позволяет передать сетевому адаптеру большой блок данных, чтобы он сам разделил его на TCP-сегменты нужного размера.
Благодаря этому процессор тратит меньше работы на подготовку каждого сегмента.

Здесь используется виртуальный адаптер `virtio_net`, поэтому результат относится к нему, а не к физической сетевой карте Mac.
