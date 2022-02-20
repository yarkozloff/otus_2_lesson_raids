# Подготовка Vagrantfile, запуск машины

Vagrant Box используется centos/7, взято из примера https://github.com/erlong15/otus-linux
В Vagrntfile добавил 5ый диск, решил запуститься и поймал ошибку с ssh таймаутом. Решил изменить сетевые настройки на те которые были в Vagrantfile из дз1.
делаем vagrant up, получаем ошибку:

A customization command failed:
["createhd", "--filename", "sata1.vdi", "--variant", "Fixed", "--size", 50]
The following error was experienced:
#<Vagrant::Errors::VBoxManageError: There was an error while executing `VBoxManage`, a CLI used by Vagrant
for controlling VirtualBox. The command and stderr is shown below.
Command: ["createhd", "--filename", "sata1.vdi", "--variant", "Fixed", "--size", "50"]
Stderr: 0%...
Progress state: VBOX_E_FILE_ERROR
VBoxManage: error: Failed to create medium
VBoxManage: error: Could not create the medium storage unit '/otus/raids/otus-linux/sata1.vdi'.
VBoxManage: error: VDI: cannot create image '/otus/raids/otus-linux/sata1.vdi' (VERR_ALREADY_EXISTS)
VBoxManage: error: Details: code VBOX_E_FILE_ERROR (0x80bb0004), component MediumWrap, interface IMedium
VBoxManage: error: Context: "RTEXITCODE handleCreateMedium(HandlerArg*)" at line 510 of file VBoxManageDisk.cpp
>
Please fix this customization and try again.

Vbox уже успел создать некоторые диски при первом vagrant up, удалил из каталога vdi с названием sata* и каталог .vagrant, снова запустил и потрпел такую же неудачу.
Смотрим на инфо на стороне Vbox:
```
sudo VBoxManage list vms
```
Довольно много машин, удаляем все через:
```
sudo VBoxManage unregistervm --delete <uuid machine>
```
Снова vagrant up, снова такая же ошибка. Почистил снова машины и sata файлы в каталоге откуда апускаюсь, смотрим диски на стороне Vbox:
sudo VBoxManage list hdds
наблюдается много дисков с названием sata* из Vagrantfile, получается из Vbox они не удаляеются после vagrant halt или удаления из директории, удалил вручную:
sudo VBoxManage closemedium disk <disk uuid> --delete
Успех, машишна запустилась после vagrant up
подключился по vagrant ssh, далее lsblk выдал 5 блочных устройств как и было прописано в Vagrantfile

# Настройки RAID внутри машины
##Создание RAID 
Занулим на всякий случай суперблоки:
```
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
```
Создаем RAID 6го уровня для 5 устройств:
```
mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
```
RAID собрался нормально, команда cat /proc/mdstat вывела 512К - размер чанка, и размер одного чанка [UUUUU]

## Создание конфигурационного файла mdadm.conf
```
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
```
Ошибка:
-bash: /etc/mdadm/mdadm.conf: No such file or directory
Действительно, каталога mdadm не существует, и файла тоже нет, создаем вручную, выполняем действия дальше:
```
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

## Сломать RAID
Фейлим блочное устройство f
```
mdadm /dev/md0 --fail /dev/sdf
mdadm -D /dev/md0
```
Состояние RaidDevice с номером 4 - removed
Удаляем "сломанный" диск из массива:
```
sudo mdadm /dev/md0 --remove /dev/sdf
```
mdadm: hot removed /dev/sdf from /dev/md0

## Починить RAID
Добавляем "вставленный новый диск" в RAID:
```
mdadm /dev/md0 --add /dev/sdf
```
mdadm: added /dev/sdf
Смотрим состояние:
```
mdadm -D /dev/md0
```
Состояние RaidDevice с номером 4 - active. Стадия rebuilding была незаметной из-за остутствия данных на диске, иначе бы увидели состояние rebuilding

## Создать GPT раздел, пять партиций и смонтировать их на диск
Создаем GPR раздел
```
parted -s /dev/md0 mklabel gpt
```
Создаем партиции:
```
parted /dev/md0 mkpart primary ext4 0% 20%
parted /dev/md0 mkpart primary ext4 20% 40%
parted /dev/md0 mkpart primary ext4 40% 60%
parted /dev/md0 mkpart primary ext4 60% 80%
parted /dev/md0 mkpart primary ext4 80% 100%
```
Создаем на партициях ФС:
Монитруем их к созданным каталогам:
```
mkdir -p /raid/part{1,2,3,4,5}
for i in $(seq 1 5); do sudo mount /dev/md0p$i /raid/part$i; done
```

# Скрипт для автосборки рейда при загрузке
```
sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f};
sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f};
sudo mkdir /etc/mdadm;
sudo bash -c 'echo "DEVICE partitions" > /etc/mdadm/mdadm.conf';
sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf;
sudo parted -s /dev/md0 mklabel gpt;
sudo parted -a optimal /dev/md0 mkpart primary ext4 0% 20%;
sudo parted -a optimal /dev/md0 mkpart primary ext4 20% 40%;
sudo parted -a optimal /dev/md0 mkpart primary ext4 40% 60%;
sudo parted -a optimal /dev/md0 mkpart primary ext4 60% 80%;
sudo parted -a optimal /dev/md0 mkpart primary ext4 80% 100%;
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done;
sudo mkdir -p /raid/part{1,2,3,4,5};
for i in $(seq 1 5);do sudo mount /dev/md0p$i /raid/part$i;done;
```
Проблемы, с которыми столкнулся:
* Команды sudo parted останавливают работу скрипты предупреждением:
Warning: The resulting partition is not properly aligned for best performance.
Ignore/Cancel?
Приходится скипать вручную
* Если прописать скрипт в Vagrantfile: config.vm.provision "shell", path: "./start.sh"
то при первом запуске vagrant up получу ошибку mdadm: command not found. Видимо скрипт начинает работу раньше чем установится mdadm
* По-хорошему нужны дополнительные проверки для создания директорий /etc/mdadm и /raid/part{1,2,3,4,5}. А также проверку на наличие RAID, чтобы можно было использовать при перезагрузки машины
