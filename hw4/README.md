# Задание 4. Linux. Память и процессы

Работу выполнял на macOS, используя виртуальную машину с Ubuntu в Multipass.
Установил пакеты из условия: `build-essential`, `python3`, `stress`, `htop` и `numactl`.

## 1. Сервис systemd

Создал `/usr/local/bin/homework_service.sh` с содержимым из условия:

```bash
#!/bin/bash
echo "My custom service has started."
while true; do
  echo "Service heartbeat: $(date)" >> /tmp/homework_service.log
  sleep 15
done
```

В `/etc/systemd/system/homework_service.service` записал:

```ini
[Unit]
Description=Homework service

[Service]
Type=simple
ExecStart=/usr/local/bin/homework_service.sh
Restart=always
RestartSec=15s

[Install]
WantedBy=multi-user.target
```

Юнит в `/etc/systemd/system` и скрипт в `/usr/local/bin` не заменяются обычными обновлениями пакетов.
`Restart=always` включает повторный запуск после завершения процесса, в том числе после падения, а `RestartSec=15s` задаёт задержку 15 секунд.
При штатной остановке через `systemctl stop` сервис не перезапускается.

Сделал скрипт исполняемым, перечитал настройки и включил сервис вместе с автозапуском:

```text
ubuntu@hw4:~$ sudo chmod +x /usr/local/bin/homework_service.sh
ubuntu@hw4:~$ sudo systemctl daemon-reload
ubuntu@hw4:~$ sudo systemctl enable --now homework_service
Created symlink /etc/systemd/system/multi-user.target.wants/homework_service.service → /etc/systemd/system/homework_service.service.
ubuntu@hw4:~$ systemctl show homework_service -p MainPID -p NRestarts -p Restart -p RestartUSec -p ActiveState
Restart=always
RestartUSec=15s
MainPID=2686
NRestarts=0
ActiveState=active
```

Для проверки падения принудительно завершил основной процесс:

```text
ubuntu@hw4:~$ sudo kill -9 2686
ubuntu@hw4:~$ systemctl show homework_service -p ActiveState -p SubState -p MainPID -p NRestarts
MainPID=0
NRestarts=0
ActiveState=activating
SubState=auto-restart
```

После ожидания процесс запустился снова с новым PID:

```text
ubuntu@hw4:~$ systemctl show homework_service -p ActiveState -p MainPID -p NRestarts -p RestartUSec
RestartUSec=15s
MainPID=2771
NRestarts=1
ActiveState=active
ubuntu@hw4:~$ sudo journalctl -u homework_service --no-pager -o short-precise -n 8
Nov 01 14:19:59.708672 hw4 systemd[1]: Started homework_service.service - Homework service.
Nov 01 14:19:59.709617 hw4 homework_service.sh[2686]: My custom service has started.
Nov 01 14:20:29.712158 hw4 systemd[1]: homework_service.service: Main process exited, code=killed, status=9/KILL
Nov 01 14:20:29.713037 hw4 systemd[1]: homework_service.service: Failed with result 'signal'.
Nov 01 14:20:44.819916 hw4 systemd[1]: homework_service.service: Scheduled restart job, restart counter is at 1.
Nov 01 14:20:44.826853 hw4 systemd[1]: Started homework_service.service - Homework service.
Nov 01 14:20:44.828692 hw4 homework_service.sh[2771]: My custom service has started.
```

Между падением и повторным запуском прошло примерно 15 секунд. Сервис продолжил писать в файл:

```text
ubuntu@hw4:~$ tail -n 2 /tmp/homework_service.log
Service heartbeat: Sun Nov  1 14:21:29 MSK 2026
Service heartbeat: Sun Nov  1 14:21:45 MSK 2026
ubuntu@hw4:~$ systemctl is-enabled homework_service
enabled
```

Топ-5 юнитов по длительности запуска:

```text
ubuntu@hw4:~$ systemd-analyze blame --no-pager | head -n 5
7.489s cloud-final.service
1.341s snapd.seeded.service
1.142s cloud-init.service
1.045s systemd-networkd-wait-online.service
 671ms cloud-init-local.service
```

## 2. Разделяемая память

Создал `shm_creator.c` по примеру из условия. Добавил проверку ошибок `ftok` и удаления сегмента:

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

int main(void) {
    key_t key = ftok("homework_key", 65);
    if (key == -1) {
        perror("ftok");
        return 1;
    }
    int shmid = shmget(key, 1024, 0666 | IPC_CREAT);
    if (shmid == -1) {
        perror("shmget");
        return 1;
    }
    printf("Shared memory segment created.\n");
    printf("ID: %d\nKey: 0x%x\n", shmid, key);
    printf("Run 'ipcs -m' to see it. Process will exit in 60 seconds...\n");
    sleep(60);
    if (shmctl(shmid, IPC_RMID, NULL) == -1) {
        perror("shmctl");
        return 1;
    }
    printf("Shared memory segment removed.\n");
    return 0;
}
```

Скомпилировал программу, создал файл для `ftok` и запустил:

```text
ubuntu@hw4:~$ gcc shm_creator.c -o shm_creator
ubuntu@hw4:~$ touch homework_key
ubuntu@hw4:~$ ./shm_creator
Shared memory segment created.
ID: 0
Key: 0x41010031
Run 'ipcs -m' to see it. Process will exit in 60 seconds...
```

Пока программа ожидала, в соседнем терминале посмотрел сегменты:

```text
ubuntu@hw4:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x41010031 0          ubuntu     666        1024       0
```

Сегмент создан, его размер - 1024 байта. `nattch` показывает число подключений сегмента к адресным пространствам процессов.
Здесь `nattch=0`: программа вызывает `shmget`, но не вызывает `shmat`, поэтому сама не подключает созданный сегмент для чтения и записи.
Для доступа к данным потребовался бы [shmat](https://man7.org/linux/man-pages/man2/shmop.2.html).

Через 60 секунд программа удалила сегмент:

```text
Shared memory segment removed.
ubuntu@hw4:~$
```

Повторная проверка показала пустой список:

```text
ubuntu@hw4:~$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
```

## 3. Память процесса: VSZ и RSS

В первом терминале запустил команду из условия, которая создаёт строку размером 250 MB и ждёт две минуты:

```text
ubuntu@hw4:~$ python3 -c "print('Allocating memory...'); a = 'X' * (250 * 1024 * 1024); import time; print('Memory allocated. Sleeping...'); time.sleep(120);"
Allocating memory...
Memory allocated. Sleeping...
```

Во втором терминале нашёл PID и посмотрел память процесса:

```text
ubuntu@hw4:~$ pgrep -af '^python3 -c'
2910 python3 -c print('Allocating memory...'); a = 'X' * (250 * 1024 * 1024); import time; print('Memory allocated. Sleeping...'); time.sleep(120);
ubuntu@hw4:~$ ps -o pid,user,%mem,rss,vsz,comm -p 2910
    PID USER     %MEM   RSS    VSZ COMMAND
   2910 ubuntu   13.2 265312 273620 python3
```

`RSS` и `VSZ` в этом выводе указаны в KB. 250 MB = 256000 KB.
`RSS=265312 (KB)` показывает объём страниц процесса, находящихся в оперативной памяти. Он не близок к нулю, поскольку Python заполнил строку символами `X`, то есть действительно обратился к её страницам.
Кроме строки память занимают сам интерпретатор, библиотеки и другие данные.

`VSZ=273620 (KB)` показывает общий размер виртуального адресного пространства процесса. Оно включает также отображения и зарезервированные области, которые не обязаны целиком находиться в RAM.
Поэтому в этом запуске VSZ больше RSS.

## 4. NUMA и cgroups

Посмотрел NUMA-топологию и наличие swap:

```text
ubuntu@hw4:~$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1
node 0 size: 1955 MB
node 0 free: 848 MB
node distances:
node   0
  0:  10
ubuntu@hw4:~$ free -m
               total        used        free      shared  buff/cache   available
Mem:            1955         240         848           1         961        1714
Swap:              0           0           0
```

В VM одна NUMA-нода с двумя процессорами и примерно 1955 MB памяти. Swap не настроен.

Запустил тест с параметрами из условия:

```text
ubuntu@hw4:~$ sudo systemd-run --unit=highload-stress-test --slice=testing.slice --property="MemoryMax=150M" --property="CPUWeight=100" stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
Running as unit: highload-stress-test.service; invocation ID: a952ce11941845768aef20d89ac0ceee
ubuntu@hw4:~$ systemctl show highload-stress-test -p Result -p MemoryMax -p CPUWeight -p ExecMainStatus
Result=oom-kill
ExecMainStatus=1
CPUWeight=100
MemoryMax=157286400
```

`150M` у systemd означает 150 MB, то есть 157286400 байт. Запустить процесс удалось, но завершить 30-секундный тест с запросом 300 MB он не смог.
Лимит установлен для сервиса внутри `testing.slice`, а не для всей slice.
Памяти всей группы, включая дочерние процессы, стало недостаточно, и ядро завершило процесс, выделявший память:

```text
ubuntu@hw4:~$ sudo journalctl -u highload-stress-test --no-pager -n 12
Nov 01 14:23:15 hw4 systemd[1]: Started highload-stress-test.service - /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s.
Nov 01 14:23:15 hw4 stress[2814]: stress: info: [2814] dispatching hogs: 1 cpu, 0 io, 1 vm, 0 hdd
Nov 01 14:23:15 hw4 stress[2814]: stress: FAIL: [2814] (425) <-- worker 2816 got signal 9
Nov 01 14:23:15 hw4 stress[2814]: stress: WARN: [2814] (427) now reaping child worker processes
Nov 01 14:23:15 hw4 stress[2814]: stress: FAIL: [2814] (461) failed run completed in 0s
Nov 01 14:23:15 hw4 systemd[1]: highload-stress-test.service: A process of this unit has been killed by the OOM killer.
Nov 01 14:23:15 hw4 systemd[1]: highload-stress-test.service: Main process exited, code=exited, status=1/FAILURE
Nov 01 14:23:15 hw4 systemd[1]: highload-stress-test.service: Failed with result 'oom-kill'.
```

Сразу после этого посмотрел пик и события памяти родительской `testing.slice`. На тот момент в ней запускался только этот тест:

```text
ubuntu@hw4:~$ cat /sys/fs/cgroup/testing.slice/memory.peak /sys/fs/cgroup/testing.slice/memory.events
157294592
low 0
high 0
max 35
oom 1
oom_kill 1
oom_group_kill 0
```

Зафиксированный пик всей slice оказался на 8192 байта, или 8 KB, выше 150 MB. Это измерение родительской группы, отдельный пик самого сервиса здесь не снимал.
В документации Linux указано, что [memory.max допускает кратковременное превышение](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files).
Это не позволило тесту удерживать 300 MB: счётчик `oom_kill=1` и результат сервиса подтверждают завершение из-за нехватки памяти внутри группы.

Повторил тест с лимитом 400 MB и весом CPU `200`:

```text
ubuntu@hw4:~$ sudo systemd-run --unit=highload-stress-ok --slice=testing.slice --property="MemoryMax=400M" --property="CPUWeight=200" stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
Running as unit: highload-stress-ok.service; invocation ID: 4ea199a8bc62468798139efd26344f5d
```

При лимите 150 MB тест завершился почти сразу. Дерево работающих процессов посмотрел в соседнем терминале во время повторного теста с 400 MB, вместе с действующими ограничениями:

```text
ubuntu@hw4:~$ sudo systemd-cgls --no-pager /testing.slice
CGroup /testing.slice:
└─highload-stress-ok.service
  ├─3063 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
  ├─3064 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
  └─3065 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
ubuntu@hw4:~$ systemctl show highload-stress-ok -p MemoryCurrent -p MemoryMax -p CPUWeight -p ControlGroup
ControlGroup=/testing.slice/highload-stress-ok.service
MemoryCurrent=87736320
CPUWeight=200
MemoryMax=419430400
ubuntu@hw4:~$ cat /sys/fs/cgroup/testing.slice/highload-stress-ok.service/memory.max /sys/fs/cgroup/testing.slice/highload-stress-ok.service/cpu.weight
419430400
200
```

В дереве видны родительский процесс `stress` и два рабочих процесса: один нагружает CPU, другой выделяет память.
`MemoryCurrent` - значение в момент измерения. Оно меняется, поскольку `stress` выделяет, заполняет и освобождает память в цикле.

После завершения проверил результат и журнал:

```text
ubuntu@hw4:~$ systemctl show highload-stress-ok -p Result -p ExecMainStatus
Result=success
ExecMainStatus=0
ubuntu@hw4:~$ sudo journalctl -u highload-stress-ok --no-pager -n 5
Nov 01 14:34:55 hw4 systemd[1]: Started highload-stress-ok.service - /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s.
Nov 01 14:34:55 hw4 stress[3063]: stress: info: [3063] dispatching hogs: 1 cpu, 0 io, 1 vm, 0 hdd
Nov 01 14:35:25 hw4 stress[3063]: stress: info: [3063] successful run completed in 30s
Nov 01 14:35:25 hw4 systemd[1]: highload-stress-ok.service: Deactivated successfully.
Nov 01 14:35:25 hw4 systemd[1]: highload-stress-ok.service: Consumed 59.929s CPU time, 301.2M memory peak, 0B memory swap peak.
```

При лимите 400 MB тест выполнился успешно, а пик памяти составил примерно 301.2 MB.

`MemoryMax` ограничивает учитываемую память всего сервиса с его дочерними процессами. Это позволяет не дать одному сервису занять всю RAM.
Лимит относится к памяти, а не к размеру виртуального адресного пространства. Swap при необходимости ограничивается отдельно.

`CPUWeight` задаёт относительный вес при распределении CPU между конкурирующими группами. Значение `100` является стандартным; `200` даёт больший вес, но не означает 200% CPU и не задаёт жёсткий предел.
Если конкуренции нет, процесс может использовать свободный CPU. Такие веса удобны для задания приоритета сервисам без фиксированной квоты.

