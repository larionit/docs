# Пошаговая инструкция по настройке сервера 1С на Linux (Proxmox + Cloud-init)

Данное руководство состоит из двух частей:

* [1. Виртуальная машина из Cloud-образа](#виртуальная-машина-из-cloud-образа)

* [2. Сервер 1С:Предприятие 8 + PostgreSQL на Ubuntu 24.04 LTS](#развертывание-сервера-1спредприятие-8)

Если вы не используете Proxmox или у вас уже есть готовая к работе система, можете сразу переходить ко второй части.

<details>

<summary>Что такое Proxmox?</summary>

---

Proxmox VE - система виртуализации. Она значительно облегчает эксплуатацию и сопровождение сервера. Ключевые преимущества, в сравнении с работой **без** системы виртуализации:

1. **Консолидация ресурсов**: Proxmox VE позволяет запускать несколько виртуальных машин (VM) на одном физическом сервере, что оптимизирует использование ресурсов и снижает затраты на оборудование.

2. **Управление и мониторинг**: Proxmox VE предоставляет удобный веб-интерфейс для управления виртуальными машинами, контейнерами и сетями, а также для мониторинга производительности и состояния системы.

3. **Высокая доступность (HA)**: Proxmox VE поддерживает функции высокой доступности, что позволяет минимизировать время простоя и обеспечить непрерывность работы сервисов. Как это работает: два и более физических сервера объединяются в единую систему (кластер), в случае недоступности одного из узлов все автоматически запускается на другом сервере.

4. **Миграция виртуальных машин**: Proxmox VE позволяет выполнять живую миграцию виртуальных машин между физическими серверами без прерывания работы, что упрощает обслуживание и обновление оборудования.

5. **Резервное копирование и восстановление**: Proxmox VE включает встроенные инструменты для создания резервных копий виртуальных машин и их восстановления, что повышает надежность и защищенность данных.

6. **Контейнеризация**: Помимо виртуальных машин, Proxmox поддерживает контейнеры LXC, что позволяет запускать изолированные приложения с меньшими накладными расходами по сравнению с традиционными VM.

7. **Открытый исходный код**: Proxmox VE является проектом с открытым исходным кодом, опубликованным под свободной лицензией. Нет необходимости в покупке лицензий, нет рисков блокировки в связи с санкциями.

</details>

## Виртуальная машина из Cloud-образа

Для ускорения развертывания и повышения удобства дальнейшей работы мы воспользуемся cloud-образом, они есть у всех популярных дистрибутивов, включая Ubuntu.

Созданную виртуальную машину мы **конвертируем в шаблон**, что избавит нас от необходимости повторять процесс создания и настройки виртуальной машины каждый раз, когда нам понадобится новая.

<details>

<summary>Про cloud-образы</summary>

---

Cloud-образы представляют из себя образ диска с **уже установленной** системой внутри. Создание виртуальной машины из такого образа значительно сокращает время развертывания.

Еще одно достоинство таких образов - [cloud-init](https://cloud-init.io/).

Благодаря cloud-init через **графический интерфейс** Proxmox можно:

* Добавить свой ssh-ключ

* Задать имя пользователя и пароль

* Настроить сеть

* Вкл./Выкл. автоматическую установку обновлений

**внутри** виртуальной машины, как **до** её **запуска**, так и после.

Подробнее про cloud-init в Proxmox можно почитать тут -> [ссылка.](https://pve.proxmox.com/wiki/Cloud-Init_Support)

</details>

### Создание шаблона виртуальной машины

---

Дальнейшие действия выполняются в консоли Proxmox.

Установим curl, если его нет:

```bash
apt update && apt install -y curl
```

Загрузим cloud-init образ Ununtu 24.04 LTS:

```bash
curl -fsSL https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O
```

Создадим виртуальную машину:

```bash
qm create 9010 --memory 512 --core 1 --name tmpl-ubuntu-noble --net0 virtio,bridge=vmbr0
```

Импортируем загруженный образ диска в хранилище:

```bash
qm importdisk 9010 noble-server-cloudimg-amd64.img local-lvm
```

Подключим образ диска к виртуальной машине:

```bash
qm set 9010 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9010-disk-0
```

Увеличим размер диска на 13ГБ:

```bash
qm resize 9010 scsi0 +13G
```

Подключим отдельный диск для cloudinit:

```bash
qm set 9010 --ide2 local-lvm:cloudinit
```

Настроим порядок загрузки:

```bash
qm set 9010 --boot c --bootdisk scsi0
```

Конвертируем виртуальную машину в шаблон:

```bash
qm template 9010 
```

Удалим загруженный ранее образ:

```bash
rm noble-server-cloudimg-amd64.img
```

### Настройка шаблона виртуальной машины

---

#### Создание ssh-ключа

---

Перейдем в каталог `~/.ssh` :

```bash
cd ~/.ssh
```

Запустим:

```bash
ssh-keygen
```

Утилита задаст два вопроса:

1. Куда и с каким именем сохранить ключ. Если указать только имя - ключ будет сохранен в текущем каталоге.

2. Пароль для использования ключа (дважды). Можно ничего не вводить, просто два раза нажать `Enter`.

Для файлов ssh-ключей принято указывать `id_` в начале имени.

Я буду использовать имя: `id_srv_1c`, пароль задавать не буду.

Пример того, как выглядит процесс:

```bash
root@pve:~/.ssh# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): id_srv_1c
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_srv_1c
Your public key has been saved in id_srv_1c.pub
The key fingerprint is:
SHA256:gaoCfjpY+Capj4sKpU0J9c6kE4O/RlMRkXcN6vg43Yc root@pve
The key's randomart image is:
+---[RSA 3072]----+
|  . ++  .o       |
| o ....o. .      |
|o o +.o..        |
| o X +   .       |
|..X = . S        |
|+*.= + . .       |
|===.o o E .      |
|*==  .   .       |
|O*o              |
+----[SHA256]-----+
```

Посмотрим что получилось:

```bash
ls | grep id_srv_1c
```

В моем случае результат выглядит так:

```bash
root@pve:~/.ssh# ls | grep id_srv_1c
id_srv_1c
id_srv_1c.pub
```

Как видно из вывода выше, в результате созданна пара ключей, приватный и публичный.

Публичный ключ отличается от приватного наличием расширения `.pub` в имени файла.

Выведем в консоль содержимое **публичного** ключа:

```bash
cat ~/.ssh/id_srv_1c.pub
```

Выделим и скопируем **весь** вывод команды, он **понадобится** нам на следующем шаге.

#### Добавление SSH-ключа в виртуальную машину

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Cloud-init` -> `SSH public key`

* Вставляем **публичный** ключ в поле под заголовком "Edit: SSH Keys"

* Cохраняем изменения нажав кнопку  `OK`

#### Настройка учетной записи

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Cloud-init` -> `User`

* В поле `User:` указываем имя пользователя

* Cохраняем изменения нажав кнопку  `OK`

* Переходим в: `Cloud-init` -> `Password`

* В поле `Password:` указываем пароль

* Cохраняем изменения нажав кнопку  `OK`

#### Настройка сети

---

Так как мы создаем шаблон, из которого в дальнейшем будут разворачиваться новые виртуальные машины, я считаю разумным настроить шаблон на получение настроек сети по DHCP.

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Cloud-init` -> `IP Config (net0)`

* Переключаем `IPv4:` в положение: `DHCP`

* Cохраняем изменения нажав кнопку  `OK`

#### Настройка виртуального дискового контроллера

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware` -> `SCSI Controller`

* В поле "Type:" меняем `VirtIO SCSI` на `VirtIO SCSI single`

* Cохраняем изменения нажав кнопку  `OK`

#### Настройка виртуального диска

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware` -> `Hard Disk (scsi 0)`

* В правом нижнем углу ставим галочку напротив `Advanced`, далее:

    * `Discard` - ставим галочку
    * `IO-thread` - ставим галочку
    * `SSD emulation` - ставим галочку, если используется SSD/NVME

Отдельно стоит отметить возможность задействования RAM для ускорения операций записи, включается в настройках диска: `Hardware` -> `Hard Disk (scsi 0)` -> `Cache:` -> `Write Back`.

Включайте эту опцию только если у вас **есть ИБП** и настроенно резервное копирование! При использовании этой опции, в случае непредвиденного завершения работы виртуальной машины, вы рискуете потерять часть данных. 

#### Изменение максимального размера виртуального диска

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware`

* Выделяем диск (один клик ЛКМ) `Hard Disk (scsi 0)`

* Нажимаем на кнопку `Disk Action`

* Выбираем пункт `+ Resize`

* В поле `Size increment (GiB):` указываем в гигабайтах сколько **прибавить** к текущему объему диска

* Применяем изменения нажав `Resize disk`

#### Настройка vCPU

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware` -> `Processors`

* В поле `Cores:` указываем сколько **ядер (потоков)** выделить виртуальной машине

***Важно:*** *практически любой современный процессор (за редким исключением) поддерживает многопоточность. Если в CPU она есть и её не отключили в BIOS: 2 виртуальных ядра в Proxmox = 1 физическому ядру. В таком случае, чтобы выделить виртуальной машине `4`* ***физических*** *ядра, необходимо в поле `Cores` указать `8`*.

#### Привязка виртуальной машины к определенным физическим ядрам

---

Если у вас процессор с большими и малыми ядрами, то вы наверняка хотите управлять тем, на каких ядрах будет работать ваша виртуальная машина.

В Proxmox это можно сделать следующим образом:

1. Узнаем порядковые номера интересующих нас ядер

2. Указываем эти номера в параметрах виртуальной машины

Как это сделать? Просто!

##### 1. Получаем информацию о ядрах

---

Выполним в консоли Proxmox комманду:

```bash
lscpu -e
```

Пример вывода:

```bash
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ   MINMHZ       MHZ
  0    0      0    0 0:0:0:0          yes 5500.0000 800.0000 4081.2539
  1    0      0    0 0:0:0:0          yes 5500.0000 800.0000 2292.8340
  2    0      0    1 4:4:1:0          yes 5500.0000 800.0000  799.0860
  3    0      0    1 4:4:1:0          yes 5500.0000 800.0000  801.2080
  4    0      0    2 8:8:2:0          yes 5500.0000 800.0000 1733.6060
  5    0      0    2 8:8:2:0          yes 5500.0000 800.0000  800.0000
  6    0      0    3 12:12:3:0        yes 5500.0000 800.0000 1521.9240
  7    0      0    3 12:12:3:0        yes 5500.0000 800.0000  800.0000
  8    0      0    4 16:16:4:0        yes 5600.0000 800.0000 4177.7720
  9    0      0    4 16:16:4:0        yes 5600.0000 800.0000 5500.0000
 10    0      0    5 20:20:5:0        yes 5600.0000 800.0000 2625.6001
 11    0      0    5 20:20:5:0        yes 5600.0000 800.0000 3733.9170
 12    0      0    6 24:24:6:0        yes 5500.0000 800.0000  799.9020
 13    0      0    6 24:24:6:0        yes 5500.0000 800.0000  800.0000
 14    0      0    7 28:28:7:0        yes 5500.0000 800.0000  800.0000
 15    0      0    7 28:28:7:0        yes 5500.0000 800.0000  800.0000
 16    0      0    8 32:32:8:0        yes 4300.0000 800.0000 3358.0439
 17    0      0    9 33:33:8:0        yes 4300.0000 800.0000 4295.9282
 18    0      0   10 34:34:8:0        yes 4300.0000 800.0000  800.0000
 19    0      0   11 35:35:8:0        yes 4300.0000 800.0000  796.4930
 20    0      0   12 36:36:9:0        yes 4300.0000 800.0000 1008.9540
 21    0      0   13 37:37:9:0        yes 4300.0000 800.0000 1686.9561
 22    0      0   14 38:38:9:0        yes 4300.0000 800.0000 1182.7000
 23    0      0   15 39:39:9:0        yes 4300.0000 800.0000  800.0000
 24    0      0   16 40:40:10:0       yes 4300.0000 800.0000  800.0000
 25    0      0   17 41:41:10:0       yes 4300.0000 800.0000  800.0000
 26    0      0   18 42:42:10:0       yes 4300.0000 800.0000 4299.9922
 27    0      0   19 43:43:10:0       yes 4300.0000 800.0000  800.0000
```

*Это вывод команды на хосте с процессором Intel Core i7-14700K.*

Большие ядра легко найти по более высокой максимльной частоте.

Нас интересуют **номера** в колонке `CPU` и **частоты** в колонке `MAXMHZ`, в примере выше:

* С `0` по `15` - так называемые "большие" ядра, или `P-Cores`

* С `16` по `27` - "малые" ядра, или `E-Cores`

##### 2. Настраиваем виртуальную машину на использование определенных ядер

---

Вооружившись **информацией выше**, в графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware` -> `Processors`

* В поле `CPU Affinity` :

    * Указываем `0-15` - виртуальная машина использует только `P-Cores`

    * Указываем `16-27`- виртуальная машина использует только `E-Cores`

    * Указываем `0,1,2,5` - виртуальная машина использует ядра 0,1,2 и 5

    * Указываем `1-3,8-11` - виртуальная машина использует ядра 1,2,3 и 8,9,10 и 11

*Примечание: будьте внимательны, у вас номера ядер могут отличаться! Хотя, по моему опыту, в процессорах Intel 12 поколения и новее, порядок всегда аналогичен примеру выше: сначала идут большие ядра, потом малые. Но в любом случае, смотрите на вывод команды `lscpu -e` в вашей системе.*

#### Настройка RAM

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Hardware` -> `Memory`

* В строке `Memory (MiB)` указываем желаемый объем в мегабайтах

Операционные системы традиционно считают так: один гигабайт = 1024 мегабайт, соотвественно, если хотите выделить `8 ГБ` указываете `8192`. При этом, помните: производители модулей памяти считают подругому, у них один гигабайт = 1000 мегабайт.

### Создание и запуск виртуальной машины

---

Создадим виртуальную машину из нашего шаблона.

Выполним в консоли proxmox команду:

```bash
qm clone 9010 1234 --name srv-1c-test --full
```

Запустим виртуальную машину:

```bash
qm start 1234
```

### Подключение к виртуальной машине по ssh

---

По умолчанию виртуальная машина получит ip-адрес по dhcp. Узнать какой адрес был выдан можно выполнив действия ниже.

#### Узнаем ip-адрес виртуальной машины

---

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Открываем окно с консолью, нажав на кнопку `>_ Console`

* Вводим логин и пароль, который указали ранее

* Выполняем команду:

    ```bash
    ip addr
    ```

    Если не лень вводить (с фильтрацией, выведет только ip):

    ```bash
    ip addr | grep eth0 | grep inet | awk '{print $2}'
    ```

#### Подключаемся по ssh

---

Ранее мы создали пару [ssh-ключей](#создание-ssh-ключа), для подключения нам нужно указать **приватный** ключ, сделать это можно с мощью ключа `-i`:

```bash
ssh имя-пользователя@ip-адрес -i ~/.ssh/id_srv_1c
```

#### Фиксация ip-адреса виртуальной машины

---

Поскольку ip-адрес был получен по dhcp, можно зафиксировать его в настройках dhcp-сервера. Если вам не нравится такой вариант и вы хотите указать ip-адрес вручную, выполните шаги ниже.

В графическом интерфейсе Proxmox:

* Выбираем нашу виртуальную машину

* Переходим в: `Cloud-init` -> `IP Config (net0)`

* Переключаем `IPv4:` в положение: `Static`

* В строке `IPv4/CIDR:` вводим: ваш-ip-адрес/24

* В строке `Gateway (IPv4):` указываем ip-адрес вашего шлюза (обычно это маршрутизатор)

* Cохраняем изменения нажав кнопку  `OK`

* Применеяем изменения нажав кнопку `Regenerate Image`

**Важно**: чтобы изменения вступили в силу, необходимо **выключить** и **включить** виртуальную машину.

На этом подготовка виртуальной машины окончена.

## Сервер 1С:Предприятие 8 + PostgreSQL на Ubuntu 24.04 LTS

### Настройка системы

---

Систему необходимо подготовить: установить обновления и необходимые зависимости, настроить локаль.

#### Обновление системы и установка зависимостей

---

Обновим систему:

```bash
sudo apt update && sudo apt upgrade -y
```

Установим пакеты:

```bash
sudo apt update && sudo apt install -y curl wget jq tar unzip fontconfig micro
```

#### Установка штрифтов Microsoft

---

Как установить штрифты от Microsoft на Linux - писали многие. Мы, в свою очередь, рассказываем: как сделать так, что бы установщик сделал все молча, не задавая вопросов.

Заранее отвечаем на вопросы установщика:

```bash
sudo sh -c "echo ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true | debconf-set-selections"
```

Запускаем установку c ключами для "тихой" установки:

```bash
sudo apt install -y msttcorefonts -qq
```

Обновляем кэш штрифтов:

```bash
sudo fc-cache –fv
```

#### Настройка локали

---

Создадим резервную копию файла:

```bash
sudo cp /etc/locale.gen /etc/locale.gen.bak
```

Проверим:

```bash
ls /etc | grep locale.gen
```

Должно быть так:

```bash
~$ ls /etc | grep locale.gen
locale.gen
locale.gen.bak
```

Расскоментируем в файле `/etc/locale.gen` строку с Русской локалью:

```bash
sudo sed -i "s/# ru_RU.UTF-8 UTF-8/ru_RU.UTF-8 UTF-8/g" /etc/locale.gen
```

Проверим результат:

```bash
cat /etc/locale.gen | grep ru_RU.UTF-8
```

Должно быть так:

```bash
~$ cat /etc/locale.gen | grep ru_RU.UTF-8
ru_RU.UTF-8 UTF-8
```

Генерируем локаль:

```bash
sudo locale-gen ru_RU.UTF-8
```

В случае успешного выполнения увидим это:

```bash
~$ sudo locale-gen ru_RU.UTF-8
Generating locales (this might take a while)...
  ru_RU.UTF-8... done
Generation complete.
```

Задаем локаль:

```bash
sudo update-locale LANG=ru_RU.UTF-8
```

Применяем изменения:

```bash
. /etc/default/locale
```

Проверяем:

```bash
locale
```

Должно быть так:

```bash
~$ locale
LANG=ru_RU.UTF-8
LANGUAGE=
LC_CTYPE="ru_RU.UTF-8"
LC_NUMERIC="ru_RU.UTF-8"
LC_TIME="ru_RU.UTF-8"
LC_COLLATE="ru_RU.UTF-8"
LC_MONETARY="ru_RU.UTF-8"
LC_MESSAGES="ru_RU.UTF-8"
LC_PAPER="ru_RU.UTF-8"
LC_NAME="ru_RU.UTF-8"
LC_ADDRESS="ru_RU.UTF-8"
LC_TELEPHONE="ru_RU.UTF-8"
LC_MEASUREMENT="ru_RU.UTF-8"
LC_IDENTIFICATION="ru_RU.UTF-8"
LC_ALL=
```

### Установка oneget

---

Во всех руководствах по настройке сервера 1С:Предприятие 8 на Linux что я видел, предлагается предварительно загрузить установчные файлы с releases.1c.ru к себе на компьютер, после чего скопировать их на сервер с помощью sshfs (sftp) или scp.

Мы же воспользуемся замечательной консольной утилитой [oneget](https://github.com/v8platform/oneget) (а точнее её [форком](https://github.com/Pringlas/oneget)) и загрузим необходимые файлы сразу на сервер.

Создадим директорию и перейдем в неё:

```bash
mkdir oneget && cd oneget
```

Загрузим архив:

```bash
curl -fsSL https://github.com/Pringlas/oneget/releases/download/v0.7.0/oneget_linux_x86_64.tar.gz -O
```

Распакуем архив:

```bash
tar -xvzf oneget_linux_x86_64.tar.gz
```

Удалим архив:

```bash
rm oneget_linux_x86_64.tar.gz
```

На этом установка oneget окончена. Запустить утилиту можно так:

```bash
./oneget
```

### Загрузка установочных файлов 1С Предприятие 8

Зададим имя учетной записи 1C ИТС:

```bash
onec_its_user="тут_имя_вашей_учетной_записи"
```

Зададим пароль от учетной записи 1C ИТС:

```bash
onec_its_pass="тут_пароль_от_вашей_учетной_записи"
```

Запустим скачивание единого дистрибутива платформы с помощью утилиты oneget:

```bash
./oneget -u $onec_its_user -p $onec_its_pass get --filter platform=server64_8 platform:linux.x64@latest
```
Пример вывода утилиты в случае успешной загрузки:

```bash
2024-09-10T10:36:41.369Z INFO github.com/v8platform/oneget/downloader Getting a file: server64_8_3_25_1394.zip
2024-09-10T10:39:14.106Z INFO github.com/v8platform/oneget Downloaded <1> releases, files <1>
```

По умолчанию, oneget создает рядом с собой директорию `downloads` и загружает файлы в неё. 

Выведем содержимое директории командой:

```bash
ls downloads/platform83
```

В моем случае вывод выглядит вот так:

```bash
~/oneget$ ls downloads/platform83
8.3.25.1394
```

Загрузилася установщик платформы версии `8.3.25.1394`, последней на данный момент.

Посмотрим что внутри директории с номером релиза в названии:

```bash
ls downloads/platform83/8.3.25.1394
```

Результат выполнения команды в моем случае:

```bash
~/oneget$ ls downloads/platform83/8.3.25.1394
server64_8_3_25_1394.zip
```

Видим что в директории с номером платформы располагается .zip архив `server64_8_3_25_1394.zip`

Распакуем этот архив командой:

```bash
unzip downloads/platform83/8.3.25.1394/server64_8_3_25_1394.zip -d downloads/platform83/8.3.25.1394/
```

Пример вывода команды:

```bash
~/oneget$ unzip downloads/platform83/8.3.25.1394/server64_8_3_25_1394.zip -d downloads/platform83/8.3.25.1394/
Archive:  downloads/platform83/8.3.25.1394/server64_8_3_25_1394.zip
  inflating: downloads/platform83/8.3.25.1394/setup-full-8.3.25.1394-x86_64.run
  inflating: downloads/platform83/8.3.25.1394/installAsRoot
  inflating: downloads/platform83/8.3.25.1394/readme.htm
  inflating: downloads/platform83/8.3.25.1394/Liberica-Notice.txt
  inflating: downloads/platform83/8.3.25.1394/LibericaJDK-8-9-10-licenses.pdf
```

Проверим результат:

```bash
ls downloads/platform83/8.3.25.1394 | grep .run
```

В результате получим все файлы содержащие .run в имени:

```bash
~/oneget$ ls downloads/platform83/8.3.25.1394 | grep .run
setup-full-8.3.25.1394-x86_64.run
```

Файл `setup-full-8.3.25.1394-x86_64.run` - так называемый "единый установщик" 1С:Предприятие 8 для Linux.

### Установка 1С

---

Запустим установку:

```bash
sudo downloads/platform83/8.3.25.1394/setup-full-8.3.25.1394-x86_64.run --mode unattended --enable-components server,ws,liberica_jre,server_admin
```

Пояснение:

* `sudo .../setup-full-8.3.25.1394-x86_64.run` - запуск установщика

* `--mode unattended` - пакетный режим, вопросы в процессе задаваться **не будут**

* `--enable-components` - указывает, какие компоненты необходимо установить:

    * **server** - кластер серверов 1С:Предприятия
    * **ws** - модули расширения веб-сервера
    * **liberica_jre** - Java Runtime Environment (JRE)
    * **server_admin** - сервер администрирования кластера серверов 1С:Предприятия

Подробнее о пакетном режиме установки в официальной документации от 1С -> [ссылка](https://its.1c.ru/db/v8320doc#bookmark:adm:TI000001075).

По окончании процесса установки, по пути `/opt/1cv8/x86_64/` должна появится директория с номером платформы, проверить можно с помощью команды:

```bash
ls /opt/1cv8/x86_64/
```

Если в выводе команды видим директорию, имя которой соотвествует версиии устанавливаемоей платформы - значит все прошло успешно.

#### Создание символических ссылок и запуск служб

---

Теперь необходимо создать символические ссылки для служб, добавить их в автозагрузку и запустить.

##### Запуск службы сервера 1С

---

Создадим символическую ссылку для службы сервера 1С:

```bash
sudo systemctl link /opt/1cv8/x86_64/8.3.25.1394/srv1cv8-8.3.25.1394@.service
```

В результате доложны получить сообщение об успешном создании символической ссылки:

```bash
~/oneget$ sudo systemctl link /opt/1cv8/x86_64/8.3.25.1394/srv1cv8-8.3.25.1394@.service
Created symlink /etc/systemd/system/srv1cv8-8.3.25.1394@.service → /opt/1cv8/x86_64/8.3.25.1394/srv1cv8-8.3.25.1394@.service.
```

Включим автозапуск службы при загрузке:

```bash
sudo systemctl enable srv1cv8-8.3.25.1394@
```

Запустим службу сервера 1С:

```bash
sudo systemctl start srv1cv8-8.3.25.1394@default
```

Проверим:

```bash
sudo systemctl status srv1cv8-8.3.25.1394@default
```

Пример вывода в случае успешного запуска службы:

```bash
~/oneget$ sudo systemctl status srv1cv8-8.3.25.1394@default
● srv1cv8-8.3.25.1394@default.service - 1C:Enterprise Server 8.3 (8.3.25.1394) (default)
     Loaded: loaded (/etc/systemd/system/srv1cv8-8.3.25.1394@.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-09-10 11:58:53 UTC; 6s ago
   Main PID: 226211 (ragent)
      Tasks: 17 (limit: 48049)
     Memory: 15.8M (peak: 18.4M)
        CPU: 27ms
     CGroup: /system.slice/system-srv1cv8\x2d8.3.25.1394.slice/srv1cv8-8.3.25.1394@default.service
```

##### Запуск службы сервера администрирования 1С (RAS)

---

Создадим символическую ссылку:

```bash
sudo systemctl link /opt/1cv8/x86_64/8.3.25.1394/ras-8.3.25.1394.service
```

Включим автозапуск службы:

```bash
sudo systemctl enable ras-8.3.25.1394
```

Запустим службу администрирования сервера 1С:

```bash
sudo systemctl start ras-8.3.25.1394
```

Проверим:

```bash
sudo systemctl status ras-8.3.25.1394
```

Пример вывода в случае успешного запуска службы:

```bash
~/oneget$ sudo systemctl status ras-8.3.25.1394
● ras-8.3.25.1394.service - 1C:Enterprise Remote Administration Server 8.3 (8.3.25.1394)
     Loaded: loaded (/etc/systemd/system/ras-8.3.25.1394.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-09-10 12:39:24 UTC; 6min ago
   Main PID: 6387 (ras)
      Tasks: 21 (limit: 48049)
     Memory: 14.3M (peak: 16.0M)
        CPU: 28ms
     CGroup: /system.slice/ras-8.3.25.1394.service
```

### СУБД Postgresql

---

Я предлагаю использовать сборку [PostgreSQL для 1С от компании Postgres Pro](https://postgrespro.ru/products/1c), по двум причинам:

* Они поддерживают своим репозитории с пакетами для различных дистрибутивов Linux

* Они предоставляют скрипт для автоматической настройки репозиториев

#### Подключение репозиториев Postgres Pro

---

Создадим директорию и перейдем в неё:

```bash
mkdir ~/postgrespro && cd ~/postgrespro
```

Загрузим скрипт подключающий репозитории:

```bash
curl -fsSL https://repo.postgrespro.ru/1c/1c-16/keys/pgpro-repo-add.sh -O
```

Запустим скрипт:

```bash
sudo sh pgpro-repo-add.sh
```

Проверим:

```bash
ls /etc/apt/sources.list.d | grep postgresql
```

В директории должен появится файл:

```bash
~/postgrespro$ ls /etc/apt/sources.list.d | grep postgresql
postgresql-1c-16.list
```

Проверим содержимое файла:

```bash
cat /etc/apt/sources.list.d/postgresql-1c-16.list
```

У меня так:

```bash
~/postgrespro$ cat /etc/apt/sources.list.d/postgresql-1c-16.list
# Repositiory for 'Postgresql for 1C 16'
deb http://repo.postgrespro.ru/1c/1c-16/ubuntu noble main
```

Убедившись что версии соотвествуют, продолжаем установку.

#### Установка СУБД

---

Запустим установку командой:

```bash
sudo apt update && sudo apt install -y postgrespro-1c-16
```

Проверим, запустилась ли служба:
  
```bash
sudo systemctl status postgrespro-1c-16
```

У меня служба успешно запустилась:

```bash
~/postgrespro$ sudo systemctl status postgrespro-1c-16
● postgrespro-1c-16.service - Postgres Pro 1c 16 database server
     Loaded: loaded (/usr/lib/systemd/system/postgrespro-1c-16.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-11 08:02:10 UTC; 1min 11s ago
    Process: 6902 ExecStartPre=/opt/pgpro/1c-16/bin/check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
   Main PID: 6905 (postgres)
      Tasks: 7 (limit: 9487)
     Memory: 71.2M (peak: 75.4M)
        CPU: 116ms
     CGroup: /system.slice/postgrespro-1c-16.service
```

Узнать, какая версия postgresql установлена можно командой:

```bash
postgres --version | awk '{print $3}'
```

В моем случае это:

```bash
~/postgrespro$ postgres --version | awk '{print $3}'
16.4
```

#### Установка пароля

---

Укажем пароль для пользователя postgres в переменной:

```bash
onec_dbms_pass="тут_ваш_пароль"
```

Установим пароль из переменной пользователю:

```bash
sudo su - postgres -c "psql -c \"ALTER USER postgres WITH PASSWORD '$onec_dbms_pass';\""
```

#### Настройка подключения к серверу СУБД

Что бы сервер 1С смог подключится к СУБД PostgreSQL, необходимо в конфигруационном файле разрешить подключения, даже для подключений в рамках одного хоста.

##### Проверяем текущие настройки

---

Найдем в конфигурационном файле `postgresql.conf` все строки, отвечающие за адреса, на которых "слушает" сервер СУБД PostgreSQL:

```bash
sudo cat /var/lib/pgpro/1c-16/data/postgresql.conf | grep "listen_addresses ="
```

В моем случае, сразу после установки, вывод команды выглядит так:

```bash
~/postgrespro$ sudo cat /var/lib/pgpro/1c-16/data/postgresql.conf | grep "listen_addresses ="
#listen_addresses = 'localhost'         # what IP address(es) to listen on;
listen_addresses = '*'
```

Строка `listen_addresses = '*'` означает, что сервер PostgreSQL будет слушать на всех доступных сетевых интерфейсах. Это включает в себя как локальные (например, `localhost`), так и внешние IP-адреса, назначенные сетевым интерфейсам машины. Так как эта строка **не закоментирвоанна** - настройка применяется.

Теперь проверим настройки `pg_hba.conf`, выполнив команду:

```bash
sudo awk '/Put your actual configuration here/ {flag=1} flag' /var/lib/pgpro/1c-16/data/pg_hba.conf
```

Команда выше выведет все строки из файла `/var/lib/pgpro/1c-16/data/pg_hba.conf`, расположенные после строки `Put your actual configuration here`.

В моем случае вывод выглядит так:

```bash
~/postgrespro$ sudo awk '/Put your actual configuration here/ {flag=1} flag' /var/lib/pgpro/1c-16/data/pg_hba.conf
# Put your actual configuration here
# ----------------------------------
#
# If you want to allow non-local connections, you need to add more
# "host" records.  In that case you will also need to make PostgreSQL
# listen on a non-local interface via the listen_addresses
# configuration parameter, or via the -i or -h command line switches.



# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
host            all             all             0.0.0.0/0       md5
```

Последняя строка:

```bash
host            all             all             0.0.0.0/0       md
```

разрешает подключение к любой базе данных, любому пользователю, с любого ip-адреса.

Выше этой строки раположены правила, разрешающие подключения с того же хоста (127.0.0.1 и ::1/128), на котором запущен сервер СУБД.

Подробнее про `pg_hba.conf` можно почитать тут -> [ссылка](https://postgrespro.ru/docs/postgrespro/16/auth-pg-hba-conf).

Итого: текущие настройки никак не ограничивают подключение к СУБД. Если нет необходимости подключаться к СУБД с других ip-адресов, разумно будет **ограничить** подключение **локальным**.

##### Ограничиваем доступ к серверу СУБД

---

Ограничить подключения к СУБД PostgreSQL можно тремя способами:

* С помощью сетевого экрана (он же *брандауэр* или *firewall*)

* Внести соотвествующие изменения в конфигруационный файл `postgresql.conf`

* Внести изменения в файл `pg_hba.conf`

Настройка средсвами самой СУБД:

<details>

<summary>Ограничение доступа в postgresql.conf</summary>

---

Создадим резервную копию файла:

```bash
sudo cp /var/lib/pgpro/1c-16/data/postgresql.conf /var/lib/pgpro/1c-16/data/postgresql.conf.bak
```

Проверим:

```bash
sudo ls /var/lib/pgpro/1c-16/data/ | grep postgresql.conf
```

Должно быть так:

```bash
~/postgrespro$ sudo ls /var/lib/pgpro/1c-16/data/ | grep postgresql.conf
postgresql.conf
postgresql.conf.bak
```


Найдем и закоментируем строку:

```bash
sudo sed -e "/listen_addresses = '*'/ s/^#*/#/" -i /var/lib/pgpro/1c-16/data/postgresql.conf
```

Найдем и раскоментируем строку:

```bash
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = 'localhost'/g" /var/lib/pgpro/1c-16/data/postgresql.conf
```

Проверим:

```bash
sudo cat /var/lib/pgpro/1c-16/data/postgresql.conf | grep "listen_addresses ="
```

Должно быть так:

```bash
~/postgrespro$ sudo cat /var/lib/pgpro/1c-16/data/postgresql.conf | grep "listen_addresses ="
listen_addresses = 'localhost'          # what IP address(es) to listen on;
#listen_addresses = '*'
```

Перезапустим службу PostgreSQL, что бы изменения вступили в силу:

```bash
sudo systemctl restart postgrespro-1c-16
```

Убедимся, что служба запущена:

```bash
sudo systemctl status postgrespro-1c-16
```

</details>

<details>

<summary>Ограничение доступа в pg_hba.conf</summary>

---

Создадим резервную копию файла:

```bash
sudo cp /var/lib/pgpro/1c-16/data/pg_hba.conf /var/lib/pgpro/1c-16/data/pg_hba.conf.bak
```

Откроем файл в редакторе micro:

```bash
sudo micro /var/lib/pgpro/1c-16/data/pg_hba.conf
```
 
Найдем строку:

```bash
host            all             all             0.0.0.0/0       md
```

Разместим символ `#` в начале этой строки, сохраним изменения и закроем файл.

Подсказка по micro:

* `CTRL` + `S` - сохранить изменения

* `CTRL` + `Q` - выйти из редактора micro

Проверим результат:

```bash
sudo awk '/Put your actual configuration here/ {flag=1} flag' /var/lib/pgpro/1c-16/data/pg_hba.conf
```

В начале последний строки должен быть символ `#`:

```bash
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
# host          all             all             0.0.0.0/0       md5
```

Перезапустим службу PostgreSQL, что бы изменения вступили в силу:

```bash
sudo systemctl restart postgrespro-1c-16
```

Убедимся, что служба запущена:

```bash
sudo systemctl status postgrespro-1c-16
```

</details>