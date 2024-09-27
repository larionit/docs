# Виртуальная машина с Open Media Vault в Proxmox

Что будем делать:

* Загрузим ISO с официального сайта и сверим хэш-суммы.

* Создадим виртуальную машину и установим систему из iso.

* Пробросим отдельный диск с хоста гипервизора в виртуальную машину.

Почему VM а не LXC?

В первую очередь по тому, что в виртуальную машину можно подключить отдельный накопитель с помощью `scsi attach`, что недоступно при использовании LXC. В целом - с VM меньше проблем.

## Загружаем ISO

Открываем в браузере страницу загурзки: https://www.openmediavault.org/download.html.

Традиционно предлагается два варианта, `Stable` и `Oldstable`, предлагаю использовать первый.

Копируем адрес ссылки:

![alt text](<assets/Proxmox + OMV (VM)/3I9qBCOsr8.gif>)

*Рекомендую для удобства сохранить сссылку куда-нибудь (в тексовый файл или заметку).*

Так же нам понадобится эталонная хэш-сумма загружаемого образа, что-бы мы могли убидиться в подлиности загруженного файла и отстувии ошибок в процессе загрузки. 

На данный момент, хэш-сумма `SHA256` размещена прямо под кнопкой загурзки:

![Хэш-сумма](<assets/Proxmox + OMV (VM)/image.png>)

*Рекомендую сохранить её рядом со ссылкой для загрузки.*

**Итого**, у нас есть:

```
Ссылка для загрузки:

https://sourceforge.net/projects/openmediavault/files/iso/7.0-32/openmediavault_7.0-32-amd64.iso

Хэш-сумма:

8587c71ce8845b1ff501e6c33b9ee033345f95b8328ea91129f82d686dc9d7e5
```

Теперь можно приступать к загрузке.

### Загрузка образа через графический интерфейс

В графическом интерфейсе Proxmox:

* Выбираем нашу ноду.

* Кликаем по хранилищу, в котором разрешено хранение ISO образов.

* Переходим в `ISO Images`.

* Нажимаем на кнопку `Download from URL`.

![alt text](<assets/Proxmox + OMV (VM)/image-1.png>)

Далее, в открывшемся окне:

* Убедись, что в правом нижнем углу, напротив `Advanced`, **стоит галочка**.

    ![alt text](<assets/Proxmox + OMV (VM)/image-14.png>)

    Это нужно для того, что бы были отображались все параметры.

* ` *URL:` - сюда вставляем ссылку.

* Нажимаем кнопку `Query URL`

* `Hash algorithm:` - тут необходимо выбрать подходящий алгоритм, в нашем случае это `SHA-256`.

* `Checksum:` - вставляем сюда хэш-сумму.

* Нажимаем кнопку `Download`.

![alt text](<assets/Proxmox + OMV (VM)/image-2.png>)

После начала загрузки **окно можно закрыть**, загрузка продолжится в **фоновом** режиме.

Чтобы **снова открыть окно** со статусом загрузки - кликните два раза по соотвествуй строке **списка задач**:

![alt text](<assets/Proxmox + OMV (VM)/image-3.png>)

Список задач в Proxmox распологается в нижней части страницы, его можно **развернуть** и **свернуть**, нажав на кнопку со стрелочкой:

![ ](<assets/Proxmox + OMV (VM)/image-4.png>) ![alt text](<assets/Proxmox + OMV (VM)/image-5.png>)

В случае успешной загрузки вы увидите соотвествующее сообщение:

```bash
calculating checksum...OK, checksum verified
download of 'https://sourceforge.net/projects/openmediavault/files/iso/7.0-32/openmediavault_7.0-32-amd64.iso' to '/var/lib/vz/template/iso/openmediavault_7.0-32-amd64.iso' finished
TASK OK
```

### Загрузка образа через консоль

Перейдем в директорию:

```bash
cd /var/lib/vz/template/iso/ 
```

*Примечание: по умолчанию, iso-бразы предпологается хранить в директории `'/var/lib/vz/template/iso/`, я показываю этот вариант. Если вы хотите загрузить образ в другое хранилище, необходимо будет перейти в соотвествующую директорию. Учтите, что бы Proxmox "увидел" iso-файлы расположенные в другой директории, необходимо произвести соответсвующие настройки.*

Запустим загрузку файла:

```bash
wget https://sourceforge.net/projects/openmediavault/files/iso/7.0-32/openmediavault_7.0-32-amd64.iso
```

По окончании загрузки, проверим наличие файла в директории:

```bash
ls /var/lib/vz/template/iso | grep openmediavault
```

В моем случае вывод выглядит так:

```bash
# ls /var/lib/vz/template/iso | grep openmediavault
openmediavault_7.0-32-amd64.iso
```

#### Сверка с эталонной хэш-суммой

Передадим имя файла в переменную `$file`, выполнив команду:

```bash
file="имя-файла.iso"
```

*Примечание: вместо **"имя-файла.iso"**, неоходимо указать имя файла, полученное вами после копирования ссылки со страницы загрузки.*

Передадим хэш-сумму в переменную `$hash`, выполнив команду:

```bash
hash="хэш-сумма"
```

*Примечание: вместо **"хэш-сумма"**, необходимо указать хэш-сумму, полученную вами со страницы загрузки.*

Вычислим хэш-сумму загруженного файла и сравним с эталонной:

```bash
echo "$hash $file" | sha256sum -c -
```

Если проверка пройдет успешно, вы увидите:

```bash
имя-файла.iso: OK
```

## Создаем виртуальную машину

### Создание VM через графический интерфейс

#### General

---

* Нажимаем на кнопку `Create VM`

    * **Важно:** убедись, что в правом нижнем углу, напротив `Advanced`, **стоит галочка**:

        ![alt text](<assets/Proxmox + OMV (VM)/image-14.png>)

        Это нужно для того, что бы были отображались все параметры.

* `Node:` - на каком сервере нужно создать VM (актуально для кластеров).

* `VM ID:` - номер виртуальной машины.

    * Можно указать любой свободный номер.

    * **После** создания VM изменить его **не получится**.

* `Name:` - имя виртуальной машины.

    * Поле можно оставить пустым.

    *  **Можно** изменить в дальнейшем.

* `Start at boot:` - ставим галочку, если хотим что бы VM запускалась автоматически после загрузки хоста.

* `Resource Pool:` — это набор виртуальных машин, контейнеров и устройств хранения. Это полезно для обработки разрешений в тех случаях, когда определенные пользователи должны иметь контролируемый доступ к определенному набору ресурсов, поскольку оно позволяет применять одно разрешение к набору элементов, а не управлять этим для каждого ресурса отдельно.

* Кликаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-6.png>)

#### OS

---

* `Storage:` - выбираем хранилище.

* `ISO image:` - выбираем образ.

* `Type:` - тип (семейство) операционной системы, в нашем случае это Linux.

* `Version:` - для систем на базе ядра Linux в этом поле указывается версия ядра.

* Нажимаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-7.png>)

#### System

---

* `Machine:` - q35.

* `SCSI Controller:` - VirtIO SCSI single.

* `Qemu Agent` - ставим галочку.

* Кликаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-8.png>)

#### Disks

---

* `Bus/Device:` - у Proxmox есть различные варианты виртуальных дисковых контроллеров, наилучшей производительностью обладает виртуальный SCSI, остальные нужны, по большей части, для совместимости.  

* `Storage:` - расположение диска (по умолчанию это local-lvm), расположить лучше на nvme/ssd диске, т.к. это будет диск "под систему", диски для данных подключим позже.

* `Disk sizq (GiB):` - 16-32 ГБ будет достаточно.

* `SSD emulation:` - ставим галочку.

* `Discard:` - ставим галочку.

* `IO thread:` - ставим галочку.

* Нажимаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-9.png>)

#### CPU

---

* `Cores:` - Сколько ядер выделить VM (потоков, если поддерживается процессором), указываем сколько не жалко.

* `Type:` - Если указать неподдерживаемый процессором хоста вариант - VM не запустится. C большей вероятностью заработает `kvm64` или `host`.

* Кликаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-10.png>)

#### Memory

---

* `Memory (MiM):` - 1024 ГБ будет достаточно.

* Нажимаем `Next` в правом нижнем углу.

![alt text](<assets/Proxmox + OMV (VM)/image-11.png>)

#### Network

---

* `Bridge:` - через какой мост подключить VM к сети (по умолчанию это vmbr0), если у вас он один - оставьте значение по умолчанию.

* `Model:` - VirtIO (paravirtualized).

![alt text](<assets/Proxmox + OMV (VM)/image-12.png>)

* Кликаем `Next` в правом нижнем углу.

#### Finish

---

На последнем шаге мы видим сводку всех выбранных нами параметров.

Если нужно что-то изменить: можно быстро перейти в нужный раздел, кликнув по  соотвествующему названию в верхней части окна.

Нажимем `Finish` что бы запустить создание VM.

![alt text](<assets/Proxmox + OMV (VM)/image-13.png>)

### Создание VM через консоль

Для удобства, сразу передадим все  параметры VM в переменные:

```bash
vm_num=201
vm_name=ilon-nas
vm_ram=1024
vm_core=2
vm_cpu=kvm64
vm_scsihw=virtio-scsi-single
vm_machine=q35
vm_net_bridge=vmbr0
vm_net_firewall=1
vm_onboot=1
vm_agent=1
vm_iso_path=/var/lib/vz/template/iso/openmediavault_7.0-32-amd64.iso
vm_storage=local-lvm
vm_drive_gb=16
vm_drive_discard=on
vm_drive_iothread=1
vm_drive_ssd=1
```

Где:

* `vm_num` - номер VM
* `vm_name`- имя VM
* `vm_ram` - максимальный доступный VM объем оперативной памяти (в мегабайтах)
* `vm_core`- количество ядер (потоков) процессора, доступных VM
* `vm_cpu` - модель эмулируемого процессора
* `vm_scsihw` - режим работы виртуального SCSI контроллера
* `vm_machine` - тип VM, `q35` поддерживает больше возможностей
* `vm_net_bridge` - имя виртуального моста
* `vm_net_firewall` - примненять или нет настройки firewall в Proxmox к VM
* `vm_onboot` - вкл./выкл. автозапуск VM (1=автозапуск)
* `vm_agent` - вкл./выкл. использование гостевых дополнений (1=использовать)
* `vm_iso_path` - путь к iso-файлу
* `vm_storage` - хранилище для диска VM (по умолчанию local-lvm)
* `vm_drive_gb` - размер диска VM в гигабайтах
* `vm_drive_discard` - включать **обязательно, если**: ssd/nvme, lvm-thin, zfs, btrfs, ceph (1=вкл).
* `vm_drive_ssd` - эмуляция ssd накопителя (что бы гостевая ОС понимала что она на ssd, 1=вкл).

Создадим виртуальную машину с параметрами заданными в переменных:

```bash
qm create $vm_num --name $vm_name --memory $vm_ram --core $vm_core --cpu $vm_cpu --scsihw $vm_scsihw --machine $vm_machine --net0 virtio,bridge=$vm_net_bridge,firewall=$vm_net_firewall --onboot $vm_onboot --agent $vm_agent
```

Подключим iso-образ:

```bash
qm set $vm_num --cdrom $vm_iso_path
```

Пример вывода:

```bash
# qm set $vm_num --cdrom $vm_iso_path
update VM 201: -cdrom /var/lib/vz/template/iso/openmediavault_7.0-32-amd64.iso
```

Добавим диск для системы:

```bash
qm set $vm_num --scsi0 $vm_storage:$vm_drive_gb,discard=$vm_drive_discard,iothread=$vm_drive_iothread,ssd=$vm_drive_ssd
```

Пример вывода:

```bash
# qm set $vm_num --scsi0 $vm_storage:$vm_drive_gb,discard=$vm_drive_discard,iothread=$vm_drive_iothread,ssd=$vm_drive_ssd
update VM 201: -scsi0 local-lvm:16,discard=on,iothread=1,ssd=1
  Logical volume "vm-201-disk-0" created.
scsi0: successfully created disk 'local-lvm:vm-201-disk-0,discard=on,iothread=1,size=16G,ssd=1'
```

Изменим порядок загрузки:

```bash
qm set $vm_num --boot order="scsi0;ide2;net0"
```

Пример вывода:

```bash
# qm set $vm_num --boot order="scsi0;ide2;net0"
update VM 201: -boot order=scsi0;ide2;net0
```

Запустить созданную VM можно так:

```bash
qm start $vm_num
```

## Установка OMV

### Открываем VNC консоль

В графическом интерфейсе Proxmox:

* Выбираем нашу ноду.

* Кликаем по созданой VM.

* Нажимаем на кнопку `>_ Console`

* Если видим надпись: *"Guest not running"*, запускаем VM нажатием на `Start Now`.

![alt text](<assets/Proxmox + OMV (VM)/image-15.png>)

### Устанавливаем систему

![alt text](<assets/Proxmox + OMV (VM)/image-16.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-17.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-18.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-19.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-20.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-21.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-22.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-23.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-24.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-25.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-26.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-27.png>)
![alt text](<assets/Proxmox + OMV (VM)/image-28.png>)

## После установки

Необходимо узнать, какой ip-адрес был получен по dhcp.

Сразу после загрузки системы в VNC-консли будет строка с именем интерефеса и полученным по dhcp ip-адресом:

![alt text](<assets/Proxmox + OMV (VM)/image-29.png>)

В моем случае, адрес такой: `10.0.0.215`.

Введя этот адрес в страку браузера, откроется web-интерфейс Open Media Vault:

![alt text](<assets/Proxmox + OMV (VM)/image-30.png>)

Логин и пароль по умолчанию:

* Логин: **admin**

* Пароль: **openmediavault**

### Меняем пароль

В графическом интерфейсе OMV:

* Нажимаем на иконку человечка в правом верхнем углу

* Кликаем по `Change Password`

![alt text](<assets/Proxmox + OMV (VM)/image-32.png>)

Вводим новый пароль:

![alt text](<assets/Proxmox + OMV (VM)/image-33.png>)

Нажимаем `Save` в правом углу:

![alt text](<assets/Proxmox + OMV (VM)/image-34.png>)

## Проброс физического диска в VM

Что бы подключить отдельный накопитель в VM, необходимо сначала узнать `/dev/disk/by-id` нужного нам накопителя, сделать это можно выполнив в консоли Proxmox:

```bash
lsblk |awk 'NR==1{print $0" DEVICE-ID(S)"}NR>1{dev=$1;printf $0" ";system("find /dev/disk/by-id -lname \"*"dev"\" -printf \" %p\"");print "";}'|grep -v -E 'part|lvm'
```

Более коротки вариант:

```bash
find /dev/disk/by-id/ -type l|xargs -I{} ls -l {}|grep -v -E '[0-9]$' |sort -k11|cut -d' ' -f9,10,11,12

```

В моем случе вывод такой:

```bash
# lsblk |awk 'NR==1{print $0" DEVICE-ID(S)"}NR>1{dev=$1;printf $0" ";system("find /dev/disk/by-id -lname \"*"dev"\" -printf \" %p\"");print "";}'|grep -v -E 'part|lvm'
NAME                            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS DEVICE-ID(S)
sda                               8:0    0 931.5G  0 disk   /dev/disk/by-id/wwn-0x5000cca39cc0b238 /dev/disk/by-id/ata-Hitachi_HDS721010CLA330_JP2940N101JGSV
sdb                               8:16   0 111.8G  0 disk   /dev/disk/by-id/ata-ADATA_SU650_4M13270WCR0T
```

```bash
~# find /dev/disk/by-id/ -type l|xargs -I{} ls -l {}|grep -v -E '[0-9]$' |sort -k11|cut -d' ' -f9,10,11,12
/dev/disk/by-id/ata-Hitachi_HDS721010CLA330_JP2940N101JGSV -> ../../sda
/dev/disk/by-id/wwn-0x5000cca39cc0b238 -> ../../sda
/dev/disk/by-id/ata-ADATA_SU650_4M13270WCR0T -> ../../sdb
```

Мне нужно подключить диск `sda`, из вывода выше известно, что точка монтирования для него:

```bash
/dev/disk/by-id/ata-Hitachi_HDS721010CLA330_JP2940N101JGSV
```

Подключим диск к VM командой:

```bash
qm set $vm_num -scsi2 /dev/disk/by-id/ata-Hitachi_HDS721010CLA330_JP2940N101JGSV
```

Альтернативный вариант команды для подключения:

```bash
update VM $vm_num: -scsi2 /dev/disk/by-id/ata-Hitachi_HDS721010CLA330_JP2940N101JGSV
```

Отключить можно так:

```bash
qm unlink $vm_num --idlist scsi2
```

Или так:

```bash
update VM $vm_num: -delete scsi2
```

Проверяем результат

В графическом интерфейсе OMV:

* Переходм в `Storage` -> `Disks`.

Устройство со значением `drive-scsi2` в столбце `Serial Number` - это тот диск, что мы пробросили ранее.

![alt text](<assets/Proxmox + OMV (VM)/image-31.png>)

## NFS

### Очистим диск от содержимого

В графическом интерфейсе OMV:

* Переходм в `Storage` -> `Disks`.

* Кликаем по нужному диску, в моем случае это `/dev/sdb`.

* Нажимаем кнопку с иконкой ластика.

* `Confirm` - cтавим галочку.

* Нажимаем кнопку `Yes`.

![alt text](<assets/Proxmox + OMV (VM)/image-35.png>)

Будет предложено два варианта:

1. `Quick` - быстрая очистка (стирает только разметку)

2. `Secure` - безопасная очистка (заполняет весь диск нулями/единицами)

Я выберу **первый**, т.к. второй имеет смысла только когда вы выкидываете диск или готовите его к тому, чтобы кому-то отдать.

![alt text](<assets/Proxmox + OMV (VM)/image-36.png>)

Процесс пройдет быстро, в конце нажимаем `Clouse` чтобы закрыть окно.

![alt text](<assets/Proxmox + OMV (VM)/image-37.png>)

### Создадим и подключим раздел

В графическом интерфейсе OMV:

* Переходм в `Storage` -> `File Systems`.

* Нажимаем на `+`

* Выбираем  ФС, я буду использовать `EXT4`

![alt text](<assets/Proxmox + OMV (VM)/image-38.png>)

Кликаем в поле `Device *`

![alt text](<assets/Proxmox + OMV (VM)/image-39.png>)

Откроется список дисков, на которых можно создать ФС, если не видите нужный диск в списке - значит на нем уже есть какая-то ФС или таблица разделов.

Выбираем из списка нужный диск, в моем случае это `/dev/sdb`:

![alt text](<assets/Proxmox + OMV (VM)/image-40.png>)

В конце нажимаем кнопку `Save` в правой части:

![alt text](<assets/Proxmox + OMV (VM)/image-41.png>)

На создание ФС уйдет какое-то время.

По окончании процесса станет активной кноика `Close`:

![alt text](<assets/Proxmox + OMV (VM)/image-42.png>)

После того, как вы нажмете `Close`, откроется диалог монтирования ФС:

![alt text](<assets/Proxmox + OMV (VM)/image-43.png>)

* Кликаем по полю `Select a file system ...`.

* Выбираем нужный раздел, в моем случае это `/dev/sdb1`

![alt text](<assets/Proxmox + OMV (VM)/image-44.png>)

* В поле `Usage Warning Threshold *` можно указать объем диска (в процентах), после заполнения которого будет отправлено уведомление с предупреждением об этом.  

* В поле `Tags` можно указать произвольные теги.

* Нажимаем кнопку `Save`.

![alt text](<assets/Proxmox + OMV (VM)/image-45.png>)

Для применения изменений необходимо нажать на галочку в появившемся сообщении:

![alt text](<assets/Proxmox + OMV (VM)/image-46.png>)

После чего необходимо дополнительно подтвердить применение изменений, нажав кнопку `Yes` во всплывшем окне:

![alt text](<assets/Proxmox + OMV (VM)/image-47.png>)


Готово, диск подключен, вы восхитительны:

![alt text](<assets/Proxmox + OMV (VM)/image-48.png>)

Если вдруг, после форматирования диска вы отошли, а вернувшись обнаружили предложение авторизоваться, авторизовались и попали на главный экран, то снова открыть раздел монтирования ФС можно так:

* Переходм в `Storage` -> `File Systems`.

* Нажимаем на кнопку `Play (треугольник)`.

![alt text](<assets/Proxmox + OMV (VM)/image-49.png>)