# Задание 3. Linux. Архитектура и файловые системы

Работу выполнял на MacOS, используя виртуальную машину с Debian 12 в QEMU.
К виртуальной машине подключил отдельный диск на 2 GB, который определился как `/dev/sdb`.
Для выполнения установил `strace`, `lvm2`, `xfsprogs` и `parted`.

## 1. Ядро и модули

Посмотрел версию ядра и полный список загруженных модулей:

```text
debian@hw3:~$ uname -r
6.1.0-54-amd64
debian@hw3:~$ lsmod
Module                  Size  Used by
cfg80211             1155072  0
rfkill                 40960  2 cfg80211
isofs                  53248  0
kvm_amd               163840  0
ccp                   118784  1 kvm_amd
kvm                  1146880  1 kvm_amd
irqbypass              16384  1 kvm
ghash_clmulni_intel    16384  0
sha512_ssse3           49152  0
sha512_generic         16384  1 sha512_ssse3
sha256_ssse3           32768  0
sha1_ssse3             32768  0
binfmt_misc            28672  1
nls_ascii              16384  1
nls_cp437              20480  1
vfat                   24576  1
fat                    90112  1 vfat
aesni_intel           393216  0
ppdev                  24576  0
iTCO_wdt               16384  1
intel_pmc_bxt          16384  1 iTCO_wdt
crypto_simd            16384  1 aesni_intel
iTCO_vendor_support    16384  1 iTCO_wdt
cryptd                 28672  2 crypto_simd,ghash_clmulni_intel
parport_pc             40960  0
watchdog               45056  1 iTCO_wdt
parport                73728  2 parport_pc,ppdev
button                 24576  0
evdev                  28672  2
serio_raw              20480  0
sg                     40960  0
fuse                  176128  1
drm                   618496  0
loop                   32768  0
efi_pstore             16384  0
dm_mod                184320  0
configfs               57344  1
qemu_fw_cfg            20480  0
ip_tables              36864  0
x_tables               61440  1 ip_tables
autofs4                53248  2
ext4                  991232  1
crc16                  16384  1 ext4
mbcache                16384  1 ext4
jbd2                  167936  1 ext4
crc32c_generic         16384  0
sd_mod                 65536  2
t10_pi                 16384  1 sd_mod
ahci                   49152  0
crc64_rocksoft         20480  1 t10_pi
sr_mod                 28672  0
crc64                  20480  1 crc64_rocksoft
crc_t10dif             20480  1 t10_pi
libahci                49152  1 ahci
cdrom                  81920  2 isofs,sr_mod
crct10dif_generic      16384  0
libata                401408  2 libahci,ahci
virtio_scsi            28672  2
virtio_net             73728  0
net_failover           24576  1 virtio_net
failover               16384  1 net_failover
crct10dif_pclmul       16384  1
virtio_pci             24576  0
crct10dif_common       16384  3 crct10dif_generic,crc_t10dif,crct10dif_pclmul
scsi_mod              286720  5 virtio_scsi,sd_mod,libata,sg,sr_mod
virtio_pci_legacy_dev    16384  1 virtio_pci
crc32_pclmul           16384  0
psmouse               184320  0
virtio_pci_modern_dev    20480  1 virtio_pci
crc32c_intel           24576  2
virtio                 20480  3 virtio_scsi,virtio_pci,virtio_net
i2c_i801               36864  0
scsi_common            16384  4 scsi_mod,libata,sg,sr_mod
virtio_ring            49152  3 virtio_scsi,virtio_pci,virtio_net
i2c_smbus              20480  1 i2c_i801
lpc_ich                28672  0
```

Конфигурация этого ядра находится в `/boot/config-6.1.0-54-amd64`.
Проверил параметры поддержки CD-ROM и XFS:

```text
debian@hw3:~$ grep -E '^(CONFIG_CDROM|CONFIG_XFS_FS)=' /boot/config-$(uname -r)
CONFIG_CDROM=m
CONFIG_XFS_FS=m
```

`CONFIG_XFS_FS=m` означает, что поддержка файловой системы XFS собрана отдельным модулем.
Он может загружаться по необходимости, например при монтировании XFS.
Значение `y` означало бы встроенную в ядро поддержку.

Чтобы отключить автозагрузку `cdrom`, создал файл в `/etc/modprobe.d`:

```text
debian@hw3:~$ printf 'blacklist cdrom\ninstall cdrom /bin/false\n' | sudo tee /etc/modprobe.d/blacklist-cdrom.conf
blacklist cdrom
install cdrom /bin/false
```

`blacklist` запрещает загрузку по внутренним псевдонимам модуля, которыми пользуется автоматическое обнаружение устройств.
Строка `install` дополнительно заменяет загрузку через `modprobe` командой `/bin/false`, в том числе при явном запросе или загрузке зависимости.
Это соответствует [описанию modprobe.d](https://man7.org/linux/man-pages/man5/modprobe.d.5.html).

CD-ROM не был смонтирован. Удалил неиспользуемые модули `isofs` и `sr_mod`, которые зависели от `cdrom`, затем сам `cdrom`. Проверил блокировку:

```text
debian@hw3:~$ sudo modprobe -r isofs sr_mod cdrom
debian@hw3:~$ sudo modprobe -n -v cdrom
install /bin/false
debian@hw3:~$ sudo modprobe cdrom; echo $?
modprobe: ERROR: ../libkmod/libkmod-module.c:1047 command_do() Error running install command '/bin/false' for module cdrom: retcode 1
modprobe: ERROR: could not insert 'cdrom': Invalid argument
1
debian@hw3:~$ lsmod | grep -E '^(cdrom|sr_mod|isofs) '
debian@hw3:~$
```

Ошибка здесь ожидаемая: загрузка запрещена, команда вернула код `1`. В списке загруженных модулей `cdrom` больше нет.
Обновил начальный образ загрузки, чтобы настройка действовала и на раннем этапе запуска системы:

```text
debian@hw3:~$ sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.1.0-54-amd64
```

После перезагрузки повторная проверка `lsmod` также не нашла `cdrom`, `sr_mod` и `isofs`.

## 2. Наблюдение за VFS

Выполнил команду из условия.
Ниже фрагмент вывода, относящийся к чтению самого файла. Начальные обращения к библиотекам и настройкам локали опущены:

```text
debian@hw3:~$ strace -e trace=openat,read,write,close cat /etc/os-release > /dev/null
...
openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3
read(3, "PRETTY_NAME=\"Debian GNU/Linux 12"..., 131072) = 267
write(1, "PRETTY_NAME=\"Debian GNU/Linux 12"..., 267) = 267
read(3, "", 131072)                     = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
+++ exited with 0 +++
```

`/etc/os-release` содержит сведения о дистрибутиве: его название, версию и другие параметры.
В этой системе это ссылка на `/usr/lib/os-release`:

```text
debian@hw3:~$ readlink -f /etc/os-release
/usr/lib/os-release
```

`openat` открыл файл для чтения и вернул дескриптор `3`.
Первый `read` прочитал 267 байт, второй вернул `0`, то есть достигнут конец файла.
Затем дескрипторы закрыты через `close`, а `cat` завершился успешно.

Вопреки формулировке условия, вызов записи в выводе есть: `write(1, ..., 267) = 267`.
Оболочка заранее перенаправила стандартный вывод, дескриптор `1`, в `/dev/null`.
Поэтому `cat` по-прежнему вызывает `write`, но записанные данные отбрасываются устройством `/dev/null` и не появляются на экране.
Именно такое поведение описано в [документации /dev/null](https://man7.org/linux/man-pages/man4/null.4.html).
Сам `strace` пишет диагностику в стандартный поток ошибок, который эта команда не перенаправляет.

## 3. Управление LVM

Перед изменениями проверил диски. Системный диск - `sda`, новый пустой диск - `sdb`:

```text
debian@hw3:~$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
NAME     SIZE TYPE FSTYPE  MOUNTPOINTS
sda       12G disk
├─sda1  11.9G part ext4    /
├─sda14    3M part
└─sda15  124M part vfat    /boot/efi
sdb        2G disk
sr0      900K rom  iso9660
debian@hw3:~$ lsblk -b -d -o NAME,SIZE,SERIAL
NAME        SIZE SERIAL
sda  12884901888 HW3ROOT
sdb   2147483648 HW3DATA
sr0       921600 seed
```

Размер `sdb` - 2147483648 байт, то есть 2 GB.
Создал на нём таблицу GPT и один раздел для LVM. Команды создания завершились без вывода:

```text
debian@hw3:~$ sudo parted -s /dev/sdb mklabel gpt
debian@hw3:~$ sudo parted -s /dev/sdb mkpart primary 1MiB 100%
debian@hw3:~$ sudo parted -s /dev/sdb set 1 lvm on
debian@hw3:~$ sudo parted -s /dev/sdb unit MiB print
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sdb: 2048MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End      Size     File system  Name     Flags
 1      1.00MiB  2047MiB  2046MiB               primary  lvm
```

На разделе `sdb1` создал физический том PV, включил его в группу `vg_highload` и выделил два логических тома:

```text
debian@hw3:~$ sudo pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
debian@hw3:~$ sudo vgcreate vg_highload /dev/sdb1
  Volume group "vg_highload" successfully created
debian@hw3:~$ sudo lvcreate -L 1200M -n data_lv vg_highload
  Logical volume "data_lv" created.
debian@hw3:~$ sudo lvcreate -l 100%FREE -n logs_lv vg_highload
  Logical volume "logs_lv" created.
debian@hw3:~$ sudo lvs --units m -o lv_name,vg_name,lv_size
  LV      VG          LSize
  data_lv vg_highload 1200.00m
  logs_lv vg_highload  844.00m
debian@hw3:~$ sudo vgs --units m
  VG          #PV #LV #SN Attr   VSize    VFree
  vg_highload   1   2   0 wz--n- 2044.00m    0m
```

`data_lv` получил ровно 1200 MB, а `logs_lv` - всё оставшееся доступное место, 844 MB.
Часть исходного диска заняли разметка, служебные данные и выравнивание. Свободного места в группе не осталось: `VFree=0`.

Отформатировал `data_lv` в ext4:

```text
debian@hw3:~$ sudo mkfs.ext4 /dev/vg_highload/data_lv
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 307200 4k blocks and 76800 inodes
Filesystem UUID: 3b53c950-447d-49f4-baa9-adbb0565ce2a
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
```

Затем отформатировал `logs_lv` в XFS:

```text
debian@hw3:~$ sudo mkfs.xfs /dev/vg_highload/logs_lv
meta-data=/dev/vg_highload/logs_lv isize=512    agcount=4, agsize=54016 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=216064, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
```

Создал каталоги и смонтировал тома:

```text
debian@hw3:~$ sudo mkdir -p /mnt/app_data /mnt/app_logs
debian@hw3:~$ sudo mount /dev/vg_highload/data_lv /mnt/app_data
debian@hw3:~$ sudo mount /dev/vg_highload/logs_lv /mnt/app_logs
debian@hw3:~$ findmnt --mountpoint /mnt/app_data -o SOURCE,TARGET,FSTYPE
SOURCE                          TARGET        FSTYPE
/dev/mapper/vg_highload-data_lv /mnt/app_data ext4
debian@hw3:~$ findmnt --mountpoint /mnt/app_logs -o SOURCE,TARGET,FSTYPE
SOURCE                          TARGET        FSTYPE
/dev/mapper/vg_highload-logs_lv /mnt/app_logs xfs
```

Для автоматического монтирования при загрузке добавил две строки в `/etc/fstab`:

```text
debian@hw3:~$ printf '/dev/vg_highload/data_lv /mnt/app_data ext4 defaults 0 2\n/dev/vg_highload/logs_lv /mnt/app_logs xfs defaults 0 0\n' | sudo tee -a /etc/fstab
/dev/vg_highload/data_lv /mnt/app_data ext4 defaults 0 2
/dev/vg_highload/logs_lv /mnt/app_logs xfs defaults 0 0
debian@hw3:~$ sudo systemctl daemon-reload
debian@hw3:~$ sudo umount /mnt/app_data /mnt/app_logs
debian@hw3:~$ sudo mount -a
debian@hw3:~$ findmnt -t ext4,xfs -o SOURCE,TARGET,FSTYPE
SOURCE                          TARGET          FSTYPE
/dev/sda1                       /               ext4
/dev/mapper/vg_highload-data_lv ├─/mnt/app_data ext4
/dev/mapper/vg_highload-logs_lv └─/mnt/app_logs xfs
```

После перезагрузки оба тома также смонтировались автоматически. `data_lv` доступен в `/mnt/app_data` как ext4, а `logs_lv` - в `/mnt/app_logs` как XFS.

## 4. /proc и /sys

Извлёк модель CPU и объём памяти:

```text
debian@hw3:~$ grep -m 1 '^model name' /proc/cpuinfo
model name	: QEMU TCG CPU version 2.5+
debian@hw3:~$ grep '^MemTotal:' /proc/meminfo
MemTotal:        2012120 kB
```

Это модель виртуального процессора, которую видит Debian.
Поле `MemTotal` показывает 2012120 KB доступной ядру памяти.
Значение немного меньше выделенных VM 2 GB, поскольку часть памяти зарезервирована системой.

Посмотрел PID текущей оболочки и её родителя:

```text
debian@hw3:~$ echo $$
768
debian@hw3:~$ grep -E '^(Name|Pid|PPid):' /proc/$$/status
Name:	bash
Pid:	768
PPid:	767
debian@hw3:~$ ps -o pid,ppid,comm -p $$,$PPID
    PID    PPID COMMAND
    767     760 sshd
    768     767 bash
```

`$$` в Bash раскрывается в PID самой оболочки: в этом сеансе это `768`.
Её `PPid=767` - родительский процесс `sshd`, который запустил оболочку при подключении по SSH.

Проверил планировщик ввода-вывода основного диска:

```text
debian@hw3:~$ cat /sys/block/sda/queue/scheduler
[none] mq-deadline
```

В квадратных скобках указан активный вариант: `none`.
На этом уровне ядро не применяет отдельный планировщик для переупорядочивания запросов. `mq-deadline` доступен для выбора.
Результат относится к виртуальному диску внутри Debian.

Основной сетевой интерфейс определил по маршруту по умолчанию и прочитал его MTU:

```text
debian@hw3:~$ ip -4 route show default
default via 10.0.2.2 dev enp0s2 proto dhcp src 10.0.2.15 metric 100
debian@hw3:~$ cat /sys/class/net/enp0s2/mtu
1500
```

Основной интерфейс - `enp0s2`, его MTU - 1500 байт.

