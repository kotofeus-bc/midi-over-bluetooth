# MIDI over Bluetooth (BLE-MIDI)
1. [Software Part](#software-part)  
1.1 [Linux (Ubuntu/Debian)](#linux-(ubuntu/debian))  
1.1.1 [Собираем BlueZ с поддержкой BLE-MIDI](#собираем-bluez-с-поддержкой-ble-midi)  
1.1.2 [Перепаковываем оригинальный пакет BlueZ](#перепаковываем-оригинальный-пакет-bluez)  
1.1.3 [Установка и мелкие фиксы ](#установка-и-мелкие-фиксы)  
1.1.4 [Удаление и откат. Uninstall and rollback](#удаление-и-откат-uninstall-and-rollback)  
1.2 [Macosx](#macosx)  
1.3 [Windows](#windows)  
2. [Hardware part]()  
***
## Software part
## Linux (Ubuntu/Debian)
Идея данного подхода в том, чтобы не слишком сильно ломать дефолтную инсталляцию. По-умолчанию скорее всего вы используете ALSA и PulseAudio звуковой сервер. Однако я проверял это на ALSA + PipeWire что тоже вполне работоспособно. В тоже время попытаемся сделать это аккуратно, чтобы было проще откатится назад. 
### Собираем BlueZ с поддержкой BLE-MIDI
Для сборки я решил использовать Docker, но не в концепии "докер ради докера", а именно чтобы произвести сборку на удаленном узле (buildnode) без лишнего мусора в системе (по-возможности). 
Определяем текущую версию bluez установленного в системе  
```bash
$ sudo dpkg -l | grep -i bluez | grep "ii"
ii  bluez   5.64-0ubuntu1   amd64        Bluetooth tools and daemons
...
``` 
Здесь мы имеем версию 5.64 установленную в системе, от нее и будем танцевать дальше. Сама система на текущий момент это Ubuntu 22.04 LTS.
Для сборки bluez внутри docker-image был написан следующий dockerfile. Его нет смысл слишком оптимизировать, т.к. нам нужен только результат сборки, а не полностью работающий контейнер на основе образа:
```dockerfile
FROM ubuntu:22.04
# Устанавливаем на всякий случай переменные для сборки
ARG LANG=C.UTF-8
ARG DEBIAN_FRONTEND=noninteractive
# Устанавливаем нужный тулинг для сборки набора утилит bluez
RUN apt-get update && apt dist-upgrade -y && \
    apt-get install -y --no-install-recommends apt-utils build-essential \
    ca-certificates sudo xz-utils systemd udev wget automake autoconf python3-docutils
# Устанавливаем необходимые для сборки библиотеки
RUN apt-get install -y --no-install-recommends libreadline-dev bison \
    flex libssl-dev libncurses-dev glib2.0 libdbus-1-dev libudev-dev \
    udev libtool libasound2-dev libical-dev
# Скачиваем исходники подготовленного релиза (лучше их, чем напрямую с git, т.к. иногда бывают проблемы с autoconf). Берем той версии которую получили выше.
RUN wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.64.tar.xz && tar -xvf bluez-5.64.tar.xz
# Конфигурируем и собираем Bluez с поддержкой MIDI. Опционально можно добавить enable depricated для сборки hcitool
RUN cd bluez-5.64 && autoconf && ./configure --enable-midi
RUN cd bluez-5.64 && make
```
Постарался необходимые пояснения дать в комментариях. Главное не завтыкать версию. Я это все сохранил в bluez.dockerfile.  
Собираем имедж с нужным нам содержимым:  
`$ docker build -t bluez:5.64 -f bluez.dockerfile .`  
Ждем успешного билда и однократно запускаем контейнер, чтобы не ломать голову с поиском нужного в overlay:  
`$ docker run bluez:5.64 --name builed_bluez `  
Он сразу упадет, да и фиг с ним. Свою задачу он выполнил. Извлекаем из него собранный bluez в заранее выбранный project_dir:  
`$ docker cp builded_bluez:bluez-5.64 ./project_dir/`  
### Перепаковываем оригинальный пакет BlueZ
Этот шаг в целом неочень обязательный, можно все и руками сделать или если мы собираем прям в системе, то и make install для тех кто привык пачкать руки. Но мы пойдем другим путем.
Скачиваем оригинальный пакет:  
```bash
$ cd .../project_dir/
$ apt download bluez
Get:1 http://ru.archive.ubuntu.com/ubuntu jammy/main amd64 bluez amd64 5.64-0ubuntu1 [1 106 kB]
Fetched 1 106 kB in 0s (8 469 kB/s)
```
Распаковываем пакет вместе с метоинформацией в поддиректорию pkg_original:
```bash
$ dpkg-deb -R bluez_5.64-0ubuntu1_amd64.deb pkg_original
```
В файле с контрольной информацией сразу поправим версию пакета и описание: 
**`./pkg_original/DEBIAN/control`**
```
...
Version: 5.64-0ubuntu2 #change whatever afer - to something newer up
...
...
Description: Bluetooth tools and daemons WITH BLE-MIDI # Added WITH descr
...
```
Теперь у нас есть немного нудная задача смержить то что мы собрали с тем что у нас было изначально. Сравнивать это все руками не самая благодарная работа, но можно помочь себе скриптом:
``` bash
#!/bin/bash
# I know, that's stupid, but fast
# get executable files in original package with paths
TARGETLIST=$(find pkg_original/ -type f -perm 755 | awk -F "/" '{print $NF}')

for TARGET in $TARGETLIST
do
  # for each founded in orginal package elf check present in builded sources
  lines=$(find ./bluez-5.64/ -iname "$TARGET" -type f | grep -v profiles | grep -v init.d | wc -l)
  if [ $lines -gt 0 ];
  then
    # that is very stupid too, but just print founed pair of files
    find ./bluez-5.64/ -iname "$TARGET" -type f
    find ./org_pkg/ -iname "$TARGET" -type f
    # and calc md5 hash
    find ./bluez-5.64/ -iname "$TARGET" -type f | md5sum
    #Uncomment for auto-merge, but beware broken init.d files
    #cp "$(find ./bluez-5.64/ -iname "$TARGET" -type f)" "$(find ./pkg_original/ -iname "$TARGET" -type f)"
  fi
done
#If i'll figure out how to auto-modifiy rows in md5sum, that would be added
```
Последнее действие лучше раскомментить после "холодного прогона", обратите внимание, чтобы туда не попали разные файлы с одинаковым именем. В моем случае это был profile для cups и init.d файл для сервиса.
В результате мы получим вывод примерно такой:  
```
/bluez-5.64/monitor/btmon
./org_pkg/usr/bin/btmon
4dbe1eb37d947e2629217ae3ca1aee44  -
./bluez-5.64/client/bluetoothctl
./org_pkg/usr/bin/bluetoothctl
d88fd76173ecca8ddd3eed9e2eb0e8d4  -
./bluez-5.64/tools/bluemoon
./org_pkg/usr/bin/bluemoon
...
```
Идем в файл **`./pkg_original/DEBIAN/md5sums`** и для обновленных бинарников заменяем значения md5 хеш-сумм. Скорее всего это можно автомазировать через sed/awk, но скрипт было писать дольше чем сделать руками и протестировать. В будущем возможно я его реализую.  
Как только все будет готово, перепаковываем обновленный оригинальный пакет:
``` bash
fakeroot sh -c '
dpkg-deb -b pkg_original bluez_5.64-0ubuntu2_amd64.deb 
'
```
### Установка и мелкие фиксы  
А далее уже можем ставить наш патченный пакет обычным путем. Если что просто удалим и поставим из официального репозитория:  
``` bash
$ sudo dpkg -i bluez_5.64-0ubuntu2_amd64.deb && sudo systemctl restart bluetooth.service
```  
Все должно заработать ок. Однако в зависимость от ситуации можно доставить пакет **`bluez-alsa-utils`** и создать директорию `/root/.pulse/` :  
``` bash 
$ sudo mkdir /root/.pulse && sudo apt install bluez-alsa-utils
```
Подключаем наше BLE-MIDI устройство и проверяем не появилось ли оно в системе:  
```
$ amidi -l
```
Если оно сразу появилось в списке - прекрасно. Если же нет, то все чуть сложнее. В моем случае устройство называлось Esp32-BLE-MIDI  
``` bash
$ aconnect -lio
...
client 129: 'Esp32-BLE-MIDI' [type=user,pid=1356597]
    0 'Esp32-BLE-MIDI Bluetooth'
```
Можем проверить поступают ли от устройства собственно коды и комманды:
``` bash
$ aseqdump -p 129
Waiting for data. Press Ctrl+C to end.
Source  Event                  Ch  Data
129:0   Note on                 0, note 60, velocity 100
129:0   Note on                 0, note 60, velocity 100
129:0   Note on                 0, note 60, velocity 100
129:0   Note on                 0, note 60, velocity 100
129:0   Note on                 0, note 60, velocity 100
```
Создадим виртуальное MIDI-устройство которое нам поможет справится со всем этим:  
``` bash
$ sudo modprobe snd_virmidi midi_devs=1
```
Проверяем:
```bash 
...
client 28: 'Virtual Raw MIDI 3-0' [type=kernel,card=3]
    0 'VirMIDI 3-0     '
client 129: 'Esp32-BLE-MIDI' [type=user,pid=1356597]
    0 'Esp32-BLE-MIDI Bluetooth'
```
И подключаем наше BLE-MIDI устройство к виртуальному MIDI:
``` bash
$ aconnect 129:0 28:0
$ aconnect -lio
...
client 28: 'Virtual Raw MIDI 3-0' [type=kernel,card=3]
    0 'VirMIDI 3-0     '
	Connected From: 129:0
client 129: 'Esp32-BLE-MIDI' [type=user,pid=1356597]
    0 'Esp32-BLE-MIDI Bluetooth'
	Connecting To: 28:0
```
Теперь можете идти в свой любимый DAW, перевести наш виртуальный midi в состояние enabled и использовать в треках или управлении.
### Удаление и откат. Uninstall and rollback
Проще некуда:
``` bash
$ sudo dpkg -r bluez && sudo apt update && sudo apt install bluez
```  
***
## Macosx
IDK
## Windows
IDK