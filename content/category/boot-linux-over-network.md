Title: PXE сервер для установки Ubuntu по сети.
Date: 15.04.2024 14:40
Category: Заметки
Tags: ipxe, dnsmasq, network

iPXE это сетевой загрузчик при помощи которого можно осуществлять установку операционных систем по сети.

#### Установка dnsmasq
dnsmasq это простой DNS-, DHCP- и TFTP-сервер. Именно то что нужно для организации работы сервера PXE.

Для начала останавливаем system-resolved и убираем из автозапуска чтобы освободить порт для dnsmasq, иначе они будут конфликтовать:
```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

Убираем символическую ссылку `/etc/resolv.conf`:
```
sudo unlink /etc/resolv.conf
```

Прописываем сервера имён вручную в `/etc/resolv.conf`:
```
sudo nano /etc/resolv.conf
```

Добавляем содержимое в `resolv.conf`:
```
nameserver 127.0.0.1
nameserver 8.8.8.8
```

Теперь устанавливаем `dnsmasq`:
```
sudo apt install -y dnsmasq
```

#### Сборка iPXE
Создадим структуру папок для `PXE`:
```
sudo mkdir -pv /pxeboot/{config,firmware,os-images}
```

Соберём `ipxe` из исходников. Клонируем репозиторий:
```
git clone https://github.com/ipxe/ipxe.git
```

Для зборки будут необходимы некоторые зависимости. Предварительно установим их:
```
sudo apt install -y build-essential liblzma-dev isolinux
```

Теперь зайдем в клонированный репозиторий в директорию `ipxe/src`:
```
cd ipxe/src
```

Создадим файл `bootconfig.ipxe` (имя файла может быть любым) и добавим в него конфигурации для загрузки по сети:
```
#!ipxe

dhcp
chain tftp://${next-server}/config/boot.ipxe
```

- `dhcp` - команда ipxe для получения настроек сети по DHCP.
- `${next-server}` - переменная с адресом tftp сервера с загрузчиком ipxe

Теперь необходимо выполнить сборку выполнить команду `make`:
```
make bin/ipxe.pxe bin/undionly.kpxe bin/undionly.kkpxe bin/undionly.kkkpxe bin-x86_64-efi/ipxe.efi EMBED=bootconfig.ipxe
```

После сборки копируем необходимые файлы в подготовленную директорию:
```
sudo cp -v bin/{ipxe.pxe,undionly.kpxe,undionly.kkpxe,undionly.kkkpxe} bin-x86_64-efi/ipxe.efi /pxeboot/firmware/
```

#### Настройка dnsmasq
Создадим на всякий случай копию настроек `dnsmasq`
```
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
```

Открываем `/etc/dnsmasq.conf`, удаляем всё содержимое и добавляем такое строки:
```
dhcp-range=ens38,192.168.0.50,192.168.0.150,255.255.255.255,12h
dhcp-option=option:router,192.168.0.1
dhcp-option=option:dns-server,8.8.8.8

enable-tftp

tftp-root=/pxeboot

# boot config for BIOS systems
dhcp-match=set:bios-x86,option:client-arch,0
dhcp-boot=tag:bios-x86,firmware/ipxe.pxe

# boot config for UEFI systems
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-match=set:efi-x86_64,option:client-arch,9
dhcp-boot=tag:efi-x86_64,firmware/ipxe.efi
```

Сохраняем файл. После этого необходимо перезапустить `dnsmasq`:
```
sudo systemctl restart dnsmasq
```

#### Установка NFS сервера
Устанавливаем NFS сервер. Он позволит обращаться к файлам образа ОС по сети:
```
sudo apt install nfs-kernel-server
```

Открываем файл конфигурации NFS сервера:
```
sudo nano /etc/exports
```

Для того чтобы сделать доступным `/pxeboot` директорию по сети необходимо в файле прописать следующую строку:
```
/pxeboot           *(ro,sync,no_wdelay,insecure_locks,no_root_squash,insecure,no_subtree_check)
```

Сохраняем конфигурацию и применяем настройки:
```
sudo exportfs -av
```

#### Подготовка образа ОС
Теперь подготовим образ для установки. Для начала скачаем его с оффициального сайта:
```
wget https://releases.ubuntu.com/jammy/ubuntu-22.04.4-live-server-amd64.iso
```

Смонтируем образ в систему:
```
sudo mount -o loop ubuntu-22.04.4-live-server-amd64.iso /mnt
```

Создадим директорию для файлов образа:
```
sudo mkdir -pv /pxeboot/os-images/mkdir ubuntu-22.04.4-live-server-amd64
```

Скопируем файлы образа в созданную директорию:
```
sudo rsync -avz /mnt /pxeboot/os-images/mkdir ubuntu-22.04.4-live-server-amd64
```

После этого примонтированный образ можно размонтировать:
```
sudo umount /mnt
```

#### Создание меню загрузки
Создадим файл ipxe с меню загрузки:
```
#!ipxe

set root_path /pxeboot

menu Select an OS to boot

item ubuntu-22.04.4-live-server-amd64	Install Ubuntu Server 22.04.4
choose --default exit --timeout 15000 option && goto ${option}

:ubuntu-22.04.4-live-server-amd64
set os_root os-images/ubuntu-22.04.4-live-server-amd64
kernel tftp://${next-server}/${os_root}/casper/vmlinuz
initrd tftp://${next-server}/${os_root}/casper/initrd

imgargs vmlinuz initrd=initrd boot=casper netboot=nfs ip=dhcp nfsroot=${next-server}:${root_path}/${os_root} ---
```

- `${next-server}` - переменная, которую предоставляет нам `dhcp` сервер, в данном случае `dnsmasq`. В ней находится адрес сервера `tftp`
- `${root_path}` - переменная с корневой директорией в которой находятся файлы `IPXE` и образа ОС
- `${os_root}` - корневай директория к файлам ОС

Сохраним файл и перезапустим `dnsmasq`
```
sudo systemctl restart dnsmasq
```

Теперь всё готово для осуществления загрузки ОС Linux по сети.
