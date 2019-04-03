# Infortrend-DS-Proxmox
Подключение SAN InforTrend к Proxmox через iSCSI LVM


Чутка теории и терминов для понятия и вникания в суть: 

В iSCSI существуют Таргеты и Инициаторы.
Сервер или SAN который раздает дисковые тома называется Таргетом, а тот кто их запрашивает и потребляет называется Инициатором.

Пример: Сервер Инициатор обращается на полку за Таргетом, полка отдает ему Таргет с (LUN) Адресом дискового устройства в полке.

LUN - Logical Unit Number это адрес дискового устройства в SANе,

Тома хранения, созданные под управлением менеджера логических томов (LVM), могут быть изменены и перемещены практически по желанию.

PV - Physical Volumes, физические тома, LUN как раз его и образует на Инициаторе и его первым надо создать.
VG - Volume Groups, Группа томов
LV : Logical Volumes, Логические тома (LV), находятся внутри группы томов (VG) и образуют виртуальные разделы,
PE : Physical Extents,  физические блоки данных
LE : Logical Extents, логические блоки данных

Таким образом, полный адрес диска (физического раздела жёсткого диска) на SCSI-устройстве складывается из SCSI Target ID и LUN, уникального в пределах SCSI-устройства и назначаемого ему в настройках или автоматически по порядку.

Пример для конкретной полки
Partition - LV - LUN - IQN - Target - (Network FC / iSCSI) - Initiator - LVM - PV - VG - LV

Ближе к делу...

Полка:
Перво наперво надо настроить дисковую полку, в моём случае это InforTrend DS1012 начиная с установки дисков в корзины, затем дополнительный контроллер и сетевые платы расширения каналов с SFP - iSCSI 10GB,
В сервер также устанавливаем сетевые платы с SFP.

Далее настраиваем сетевые интерфейсы "управления" и "каналов" передачи данных Полки.

Управление:
 Device -> System settings -> Communication -> LAN0
Каналы данных:
 Device -> Channels -> Host Channel Settings -> Channel0 - Channel5

Когда заказывали полку я сделал ошибку и в целях экономии заказал платы на iSCSI, а не на FC, посчитав, что FC выходит гораздо дороже, и как оказалось после покупки, есть ограничениях в этих платах iSCSI, их нельзя собрать в Bond.

Далее надо провести настройка жестких дисков,
Собрать в RAID необходимого уровня, объединить их в логические тома LV, создать именованные разделы Partition .

Далее Partition надо замапить на LUN, делается так:
Device -> Logical Volumes -> Partitions -> ИмяРаздела -> Host LUN Mapping -> Create
я создавал на автомате для каналов SFP на обоих контроллерах, но можно и вручную задать:
Device -> Logical Volumes -> Partitions -> ИмяРаздела -> Port Scanning.

Сервер Proxmox

Необходимо настроить сетевые интерфейсы, можно можно по отдельности, или как я объединить их в одном OVS бридже, "PVE / System / Network " и на бридж уже назначать IP адреса той-же сети, что и на каналах Полки.

Устанавливаем пакеты open-iscsi и multipath-tools
# apt-get install open-iscsi multipath-tools
# systemctl enable iscsid
# systemctl enable multipathd
# systemctl start iscsid
# systemctl start multipathd

Применяем настройки сети, можно просто перезагрузить, можно вручную заменить /etc/network/interfaces на /etc/network/interfaces.new и выключить/включить необходимый интерфейс/бридж.

Далее надо настроить хранилища.
Хранилища настраиваются на уровне датацентра
 "Datacenter -> Storage -> Add -> iSCSI"
 где:
 ID - Название хранилища (например storage.ip или storage.por_lun)
 Portal - ip адрес канала полки
 Target - Если все нормально настроили и применили настройки сети то в ниспадающем списке должны увидеть Таргеты iqn с полки, выбираем который нужен (InforTrend DS1012 отдает их в виде )
 Nodes - ноды которым будет доступно хранилище, ведь нету смысла давать удаленным сервакам.
 Enabled - ставим галку, включаем хранилище в использование
 Shared - убираем галку если планируем использовать LVM поверх хранилища, а если только для хранения образов машин напрямую, поставьте галку, но это не мой случай!

И так надо настроить хранилище на каждый SFP канал, внимательно смотрите за галками в разных версиях Proxmox они то стоят то не стоят!

Проверьте настройки iscsi автостарта нод
#cat /etc/iscsi/iscsid.conf | grep "node.startup = manual"
Автостарт нод при загрузке включается через
"node.startup = automatic"

Также здесь можно поменять настройки для работы "multipath" по мануалу производителя:

node.conn[0].timeo.noop_out_interval = 30
node.conn[0].timeo.noop_out_timeout = 90
по дефолту стоит 5
Не забываем перезапускать сервисы!

У меня стоит два активных канала, и из предыдущего шага я получил два хранилища, каждое из них указывает на один и тот-же том, для решения вопроса подключения не по конкретному адресу хранилища и используется multipath, он определяет по ID одинаковые диски и обрабатывает их как один, давая возможность объединить их и с балансировкой нагрузки или настроить отказоустойчивость. 

Посмотреть какие тома подключены через multipath можно посмотреть : 
#multipath -ll

В нем же указаны wwid блочных устройств и другая полезная информация, например производитель и контроллер или серия устройств.

В моем случае multipath установился без создания файла дефолтного конфига   в /etc/multipath.conf и пришлось лопатить интернет в поисках примера настройки параметров хранилища от производителя Полки.

Вот так выглядит готовый конфиг для InforTrend DS-1012.

# nano /etc/multipath.conf

defaults {
        polling_interval        2
        path_selector           "round-robin 0"
        path_grouping_policy    multibus
        uid_attribute           ID_SERIAL
        rr_min_io               100
        failback                immediate
        no_path_retry           queue
        user_friendly_names     no
}
devices {
        device {
        vendor "IFT"
        path_grouping_policy group_by_prio
        getuid_callout "/lib/udev/scsi_id --whitelisted --device=/dev/%n"
        path_checker readsector0
        path_selector "round-robin 0"
        hardware_handler "0"
        failback 15
        rr_weight uniform
        no_path_retry 12
        prio alua
        }
}
blacklist {
          wwid *
}
blacklist_exceptions {
          wwid 3600d0231000900651d682d0917063680
}
#

В этом примере blacklist исключаются все хранилища, но можно исключить конкретные блочные устройства указав их wwid, в blacklist_exceptions указываются все исключения и в нем как раз и указан wwid нашего partition с Полки.

Если хотите видеть пути в виде "mpatha, mpathb" и тп, измените в конфиге "user_friendly_names" с "no" на "yes"

перезагружаем  сервис
# service multipathd restart
проверяем через 
# multipath -ll
радуемся если увидели там свой том с полки :)
если не увидели перепроверьте конфиг и указанные wwid.

Далее создаем физический том (PV) ,
# pvcreate /dev/mapper/3600d0231000900651d682d0917063680
3600d0231000900651d682d0917063680 - это wwid тома с Полки, у вас может быть другой или называться mpathA и тп, смотря как настроили.

Создаем Группу Томов (VG)
# vgcreate storage00  /dev/mapper/3600d0231000900651d682d0917063680
storage00 - наименование группы томов

Дальше можно перейти в Proxmox и уже там донастраивать если вы будете использовать том только для хранения образов, ну а если на полке места много а виртуалки не такие большие... можно создать на полке логические тома под свои нужды!
например для бэкапов:
# lvcreate -n backup -L 1Tp storage00
для ISOшников
# lvcreate -n ISO -L 1Tp storage00
и тд
-n ISO - имя логического тома,
-L 1Tp - размер в один терабайт
storage00 - имя группы томов

необходимо активировать группы томов и логических томов
# vgchange -a y

Теперь можно создать файловую систему на свеженьких LV томах
# mkfs.ext4 /dev/mapper/storage00-backup
# mkfs.ext4 /dev/mapper/storage00-ISO

создаем папки для монтирования LV томов
# mkdir /mnt/storage00.backup
# mkdir /mnt/storage00.ISO

добавляем записи монтирования LV томов в
# nano /etc/fstab

/dev/mapper/storage00-backup /mnt/storage00.backup/ ext4 _netdev,defaults,auto,nofail 0 1
/dev/mapper/storage00-ISO /mnt/storage00.ISO/ ext4 _netdev,defaults,auto,nofail 0 1
сохраняем, и монтируем тома 
# mount -a
параметр _netdev говорит о том, что устройство активируется после сетевых драйверов 

Проверяем
# lsblk
напротив имени тома должна быть прописана точка монтирования

Далее уже из интерфейса Proxmox добавляем группу томов LVM 
Datacenter -> Storage -> Add -> LVM
ID - имя которым будете пользоваться
Base storage - Выбираем "Existing volume groups"
Volume group - Здесь выбираем "storage00"
Content - Что вы там размещать будете, образы ВМ или контейнеры
Nodes - на каких нодах будет доступно хранилище
Enable - включаем/выключаем
Shared - ставим галку
После добавления в списке "Storage" появится свеженькое хранилище "storage00 (pve)",  если посмотреть на Usage то оно будет частично занято созданными LV томами.

Добавляем папки Backup и ISO в Proxmox 
Datacenter -> Storage -> Add -> Directory
ID -  имя которым будете пользоваться
Directory - папка монтирования из "fstab", /mnt/xxxxxx
Content - натыкайте себе по желанию
Nodes - на каких нодах будет доступно хранилище
Enable - включаем/выключаем
Shared - ставим галку
Max Backup - Максимальное количество бэкапов для каждой ВМ, доступно если выбрали "VZDump backup file" в Content.

Кстати расписание бэкапов создается в 
Datacenter -> Backup -> Add

Еще раз все перепроверям:

Физические тома:
# pvs
Группы томов:
# vgs
Логические тома:
# lvs
Мультипаф:
# multipath -ll
и его конфиг
# cat /etc/multipath.conf
Список блочных устройств:
# lsblk
Конфиг iSCSI
# cat /etc/iscsi/iscsid.conf
Конфиг монтирования 
# cat /etc/fstab

И скрестив пальцы на ногах перезагружаем.


Для Таргета как раздатчиком на Debian'е используется пакет tgt 
# apt-get install tgt
# systemctl enable tgt
# systemctl start tgt



cat /etc/iscsi/iscsid.conf | grep "node.startup = manual"
