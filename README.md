# Старт, Стоп и Перезапуск на AWS
Разработано   **[STARTSPACE - корпоративное облако приложений 1С, SQL & WEB] (https://www.startspace.ru)**

Вашему вниманию представлена коллекция скриптов автоматизации запуска, остановки и перезапуска инстанций AWS.
Характерной особенностью коллекции является то, что мы создавали ее для использования без
внешних переменных, хранимых в отдельном файле или базе данных. Такое решение было продиктовано желанием
обеспечить наших клиентов инструментом, не требующем дополнительных настроек. Все, что необходимо - это **правильно
выставить ключи тегов и их соответствующие значения для автоматизируемых инстанций AWS,** используя для этого собственную панель AWS, стороннее решение или AWS CLI язык.

Кроме этого, каждый из скриптов можно использовать как по отдельности, так и в составе предлагаемых групп.
# Описание технического решения
Архитектура технического решения выглядит следующим образом:
- папка ```start``` содержит скрипты для запуска инстанций
- папка ```stop``` содержит скрипты для остановки инстанций
- папка ```reboot``` содержит скрипты для перезапуска инстанций

Внутри каждой из папок есть дополнительные папки:
- ```ad``` - ALL DAYS - скрипты, которые запускаются ежедневно
- ```wd``` - WORK DAYS - скрипты, которые запускаются по рабочим дням (понедельник - пятница)

*Под РАБОЧИМИ ДНЯМИ подразумеваются обычные рабочие дни, без учета праздничных дней.*

Файл ```import-cron-jobs-full-list.sh``` содержит полный комплект расписаний для исполнения всех необходимых заданий по запуску,
остановке и перезапуску инстанций. Расписание сформировано таким образом, что задания выполняются каждый час, например в 12.00,
13.00, 14.00 и т.д. Расписание покрывает полные сутки, 7 дней в неделю. Данный файл предназначен для импорта (см. ниже)
в предлагаемом или измененном виде.

# Порядок установки

**Предварительные действия**

В первую очередь необходимо создать права доступа, группу и технического пользователя.

**Шаг 1.** Для создания прав доступа необходимо войти в Панель управления AWS и средствами IAM (Identity and Access Management) создать "Policy" (меню слева "Policies", кнопка "Create Policy", выбрать "Create Own Policy" и нажать на кнопку "Select"). В открывшееся окно вставить код, представленный ниже (он включает в себя разрешения для резервного копирования путем создания "snapshot"-ов, см. [AWS Snapshots with tags] (https://github.com/STARTSPACE/aws-snapshot-with-tags)):
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ec2actionsforstartstoprebootbackupsecure",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceAttribute",
                "ec2:DescribeInstanceStatus",
                "ec2:ReportInstanceStatus",
                "ec2:StopInstances",
                "ec2:StartInstances",
                "ec2:RebootInstances",
                "ec2:DescribeRegions",
                "ec2:CreateSnapshot",
                "ec2:CreateTags",
                "ec2:DeleteSnapshot",
                "ec2:DescribeSnapshots",
                "ec2:DescribeSnapshotAttribute",
                "ec2:ResetSnapshotAttribute",
                "ec2:DescribeVolumes",
                "ec2:DescribeVolumeAttribute",
                "ec2:DescribeVolumeStatus",
                "ec2:ResetSnapshotAttribute"
            ],
            "Resource": "*"
        }
    ]
}
```
В поле "Policy name" внести имя, например ```ec2-actions```, сохранить документ, нажав кнопку "Create policy"

**Шаг 2.** Теперь создадим группу и прикрепим к ней только что созданные права. Для этого возвращаемся в IAM Dashboard (щелкнуть "Dashboard" слева вверху), выбираем "Groups", "Create New Group". В поле "Group Name" впишите название, например ```ec2-actions-all-instances```

На следующей странице в полученном списке прав выберите те права, которые создали на предыдущем шаге (для ускорения поиска можно воспользоваться полем поиска, например используя "ec2-actions-all-instances").

На следующей странице Вам будет представлена краткая сводка создаваемой группы, Вам необходимо подтвердить создание, нажав на кнопку "Create Group".

**Шаг 3.** Теперь создаем технического пользователя. Возвращаемся в IAM Dashboard (щелкнуть "Dashboard" слева вверху), выбираем "Users", нажимаем на кнопку "Create New Users". Присваиваем имя, например ```ec2-actions-user``` и нажимаем кнопку "Create". По окончанию процесса создания пользователя необходимо сохранить файл с учетными данными (ключ доступа и секретный ключ). Учетные данные НЕОБХОДИМОО СОХРАНИТЬ ЛОКАЛЬНО, так как скачать их в дальнейшем будет невозможно.

Возвращаемся в IAM Dashboard, выбираем "Users" и находим созданного пользователя. Открываем запись и нажимаем на кнопку "Add User to Groups". В открывшемся окне выбираем только что созданную группу и нажимаем кнопку "Add to Groups".

**На этом предварительные действия завершены**

**Порядок установки на RPM системы**

ВНИМАНИЕ! Порядок установки на RPM системы (Amazon Linux, CentOS и т.д.) будет уточнен позднее

**Порядок установки на DEB системы**

*ВНИМАНИЕ! Данный порядок установки предполагает, что Вы настраиваете задания для инстанций из одного региона. Если Вам необходимо управлять инстанциями из разных регионов, то смотрите комментарии ниже*

Средствами AWS Console или AWS CLI создайте инстанцию с операционной системой Ubuntu 14.04 (LTS). Подключиться к ней можно с использованием PuTTY или MobaXterm. Более подробно можно почитать на портале документации AWS, [Connect to Your Instance] (http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-connect-to-instance-linux.html)

Обновляем операционную систему и установленные утилиты
```
$ sudo apt-get update && sudo apt-get upgrade
```

Устанавливаем необходимые пакеты
```
$ sudo apt-get install libwww-perl libdatetime-perl
```

Устанавливаем Python PIP
```
$ curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
$ sudo python get-pip.py
```

Устанавливаем AWS CLI
```
$ sudo pip install awscli
```

Настраиваем AWS CLI
```
$ aws configure
```

Для настройки AWS CLI потребуется файл с учетной записью технического пользователя. Последовательно в строку консоли внесите следующие данные:
- ключ доступа (Access Key)
- секретный ключ (Security Key)
- регион (например ```eu-west-1``` или ```eu-central-1```)
- формат вывода данных - ПРОПУСТИТЬ (нажать ENTER)

Проверяем установку AWS CLI
```
$ aws ec2 describe-regions --output text
```

Вы должны получить примерно следующий ответ
```
REGIONS ec2.sa-east-1.amazonaws.com     sa-east-1
REGIONS ec2.eu-central-1.amazonaws.com  eu-central-1
REGIONS ec2.ap-northeast-1.amazonaws.com        ap-northeast-1
REGIONS ec2.eu-west-1.amazonaws.com     eu-west-1
REGIONS ec2.us-east-1.amazonaws.com     us-east-1
REGIONS ec2.us-west-1.amazonaws.com     us-west-1
REGIONS ec2.us-west-2.amazonaws.com     us-west-2
REGIONS ec2.ap-southeast-2.amazonaws.com        ap-southeast-2
REGIONS ec2.ap-southeast-1.amazonaws.com        ap-southeast-1
```

Устанавливаем Git
```
$ sudo apt-get install git
```

Создаем рабочие директории
```
$ mkdir ~/aws && mkdir ~/aws/automation && mkdir ~/tmp
```

Создаем директорию для логов
```
$ sudo mkdir /awslog
```

Создаем файл логов
```
$ sudo touch /awslog/automation-instances.log
```

Назначаем для директории логов права доступа
```
$ sudo chmod -R 777 /awslog
```

Устанавливаем комплект скриптов
```
$ git clone https://github.com/STARTSPACE/aws-ec2-start-stop-reboot-by-timetable.git ~/aws/automation
```

Переходим в рабочую директорию
```
$ cd ~/aws
```

Устанавливаем права на исполнение файлов
```
$ find . -type f -exec chmod +x {} \;
```

Устанавливаем правильный часовой пояс (мы используем московское время)
```
$ sudo dpkg-reconfigure tzdata
```

Назначаем приложение для редактирования заданий CRON (мы используем Nano)
```
$ crontab -e
```

Импортируем задания для CRON
```
$ crontab ~/aws/automation/import-cron-jobs-combined.sh
```

**На этом установка завершена**

# Настройка расписания

Управлять запуском, остановкой и перезапуском инстанций необходимо с использованием тегов. Для этого используются следующие теги:
- ```bash-starttime```, значения от ```00-00```, ```01-00```, ```02-00``` и т.д. до ```23-00```
- ```bash-stoptime```, значения от ```00-00```, ```01-00```, ```02-00``` и т.д. до ```23-00```
- ```bash-reboottime```, значения от ```00-00```, ```01-00```, ```02-00``` и т.д. до ```23-00```
- ```bash-profile```, значения ```ad``` и ```wd```, что является сокращением от All Days и Work Days

**ПРИМЕРЫ**

**Пример 1** - *Установить время запуска инстанции по рабочим дням в 7 часов утра*

На выбранной инстанции установить:
- тег ```bash-starttime```, значение ```07-00```
- тег ```bash-profile```, значение ```wd```

**Пример 2** - *Установить время остановки инстанции ежедневно в 9 часов вечера*

На выбранной инстанции установить:
- тег ```bash-stoptime```, значение ```21-00```
- тег ```bash-profile```, значение ```ad```

**Пример 3** - *Установить время перезапуска инстанции ежедневно в 3 часа ночи*

На выбранной инстанции установить:
- тег ```bash-reboottime```, значение ```03-00```
- тег ```bash-profile```, значение ```ad```

# Часто задаваемые вопросы

**Нужно ли удалять неиспользуемые задания CRON?**

По нашему мнению не нужно. Даже самая легкая инстанция абсолютно без проблем справляется с такой нагрузкой, кроме этого Вы всегда можете легко и быстро менять расписание для инстанции

**Как настроить расписание для нескольких регионов одного аккаунта?**

Есть два способа. Первый - сделать копию скрипта и в командную строку выборки по тегам, а также команды на действие вставить дополнительный ключ в формате ```--region значение```. Второй способ (который мы используем) - создать дополнительного пользователя и настроить задания от его имени и с его настройками.

**Можно ли запускать задания для другого аккаунта?**

Да, можно. Для этого Вам нужно пройти предварительные шаги на другом аккаунте и перенастроить пользователя от имени которого запускаются CRON задания.

**Какие еще задания могут выполнятся по расписанию?**

Их так много, что здесь просто не имеет смысла их всех описывать. Более подробно со потенциальным списком заданий можно ознакомиться здесь: [AWS CLI documentation] (http://docs.aws.amazon.com/cli/latest/reference/)

**Могу ли я использовать скрипты в коммерческих проектах?**

Да, без ограничений. Мы их создавали как пример автоматизации использования инфраструктуры Amazon Web Services. Если Вы заинтересованы в более глубокой интеграции с Вашей инфосистемой, то связывайтесь с нами по электронной почте ```contact@startspace.ru```, постараемся помочь.

# Дополнительные команды

**Установка обновлений скриптов**

Иногда мы вносим некоторые изменения в скрипты, это связано с тем, что язык AWS CLI развивается, появляются новые возможности. Ну и ошибки исправляем, как без этого :) Вот команды для обновления скриптов:
- Переходим в директорию скриптов ```cd ~/aws/automation```
- Сбрасываем изменения (иначе может возникнуть ошибка) ```sudo git stash```
- Устанавливаем обновления ```sudo git pull```
- Обновляем права на установленные обновления ```sudo find . -type f -exec chmod +x {} \;```
- Наличие или отсутствие ошибок можно посмотреть командой ```sudo nano /var/log/syslog``` - имейте ввиду, что ошибки там будут появляться только после срабатывания CRON, то есть по прошествии полного часа (12:00, 13:00 и т.д.)

**Доустановка на сервер скриптов резерного копирования**

- Создаем файл логов ```sudo touch /awslog/ebs-snapshot.log```
- Назначаем для директории логов права доступа ```sudo chmod -R 777 /awslog```
- Устанавливаем комплект скриптов ```$ git clone https://github.com/STARTSPACE/aws-ebs-snapshot-with-tags-by-timetable.git ~/aws/snapshot/```
- Переходим в рабочую директорию ```$ cd ~/aws```
- Устанавливаем права на исполнение файлов ```$ find . -type f -exec chmod +x {} \;```
- Импортируем задания для CRON ```$ crontab ~/aws/snapshot/import-cron-jobs-combined.sh```
- Наличие или отсутствие ошибок можно посмотреть командой ```sudo nano /var/log/syslog``` - имейте ввиду, что ошибки там будут появляться только после срабатывания CRON, то есть по прошествии полного часа (12:00, 13:00 и т.д.)
