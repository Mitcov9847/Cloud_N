# Лабораторная работа №3: Облачные сети AWS

## Цель работы

Научиться создавать виртуальную сеть (VPC) в AWS, настраивать подсети, таблицы маршрутов, интернет-шлюз (IGW) и NAT Gateway. Проверить взаимодействие веб-сервера в публичной подсети и сервера базы данных в приватной.

**После выполнения работы студент:**
- понимает, как работает изоляция сетей в AWS;
- умеет создавать и связывать сетевые компоненты;
- знает, как EC2-инстансы разных подсетей взаимодействуют между собой;
- различает публичные и приватные маршруты.


## Условия

**Amazon VPC (Virtual Private Cloud)** — это изолированная виртуальная сеть в облаке AWS с возможностью управления адресным пространством, подсетями, шлюзами и безопасностью.

Типичная архитектура:
- **Публичная подсеть** — для веб-сервера, с выходом в Интернет.
- **Приватная подсеть** — для базы данных, без прямого доступа извне.
- **NAT Gateway** — обеспечивает интернет-доступ для приватных ресурсов.
- **Route Tables** — маршрутизация трафика.
- **Security Groups** — управление доступом на уровне инстансов.
- **EC2-инстансы** — веб-сервер, сервер базы данных, Bastion Host.

---

## Ход выполнения

### 1. Подготовка среды
1. Войти в **AWS Management Console**.
2. Установить регион: `Frankfurt (eu-central-1)`.

<img width="409" height="659" alt="{F03F8CD6-A809-429C-9F43-567A214AF80A}" src="https://github.com/user-attachments/assets/93c19006-04f7-4fea-9780-adb2ef4c6a5a" />

### 2. Создание VPC
1. Открыть `Your VPCs → Create VPC`.
2. Заполнить поля:
   - **Name tag**: `student-vpc-kXX`
   - **IPv4 CIDR block**: `10.(k%30).0.0/16`
   - **Tenancy**: Default
3. Нажать **Create VPC**.

<img width="1583" height="505" alt="{A08AB396-CA15-4950-BA64-4570F51D1376}" src="https://github.com/user-attachments/assets/062acf94-97de-4399-9be3-1dcb9a28928d" />

### Шаг 3. Создание Internet Gateway (IGW)

Internet Gateway позволяет ресурсам внутри VPC выходить в Интернет. _Без него публичные IP-адреса не будут работать_. Если виртуальной машине (EC2) назначен публичный IP, но нет IGW, она не сможет общаться с внешним миром.

1. Открыть `Internet Gateways → Create internet gateway`.
2. Имя: `student-igw-kXX`.
3. Прикрепить к VPC: `Attach to VPC → student-vpc-kXX`.

<img width="1610" height="473" alt="{13FFE28D-32D8-4EEC-BBBC-C31750855755}" src="https://github.com/user-attachments/assets/40c6647b-9bce-49a6-a8a4-eea6e649434c" />

3. Теперь нужно “прикрепить” (Attach) шлюз к сети:
   - Выберать созданный IGW.
   - Нажать `Actions` → `Attach to VPC`.
   - В списке выбрать `student-vpc-kXX`.
   - Подтвердить действие.
<img width="1799" height="405" alt="{9BF62EE6-A141-4497-A3FE-AEDE60873982}" src="https://github.com/user-attachments/assets/b699f51f-f31a-49c4-bb21-00abe78878d1" />


### Шаг 4. Создание подсетей

_Подсети (subnets)_ — это сегменты внутри VPC, которые позволяют изолировать ресурсы. То есть, подсети создаются для разделения ресурсов по функционалу и уровню доступа и для более гибкого управления трафиком.

#### Шаг 4.1. Создание публичной подсети

Теперь, когда есть VPC и Internet Gateway, необходимо создать первую подсеть — публичную.
Эта подсеть будет содержать ресурсы, которым нужен прямой доступ из Интернета.

1. `Subnets → Create subnet`.
2. Параметры:
   - **VPC**: `student-vpc-kXX`
   - **Subnet name**: `public-subnet-kXX`
   - **Availability Zone**: `eu-central-1a`
   - **IPv4 CIDR block**: `10.(k%30).1.0/24`
3. **Создать**.
   
<img width="1580" height="367" alt="{9BA202ED-B2CF-4EF4-B296-5700F11E3F6F}" src="https://github.com/user-attachments/assets/10eece77-7315-4299-8ece-1f272e77a468" />

> **Вопрос:** Является ли подсеть публичной?  
> **Ответ:** Нет, маршрут в IGW ещё не добавлен.

#### Шаг 4.2. Создание приватной подсети

Подсеть называется приватной, если её трафик не направляется напрямую в Интернет.

1. `Subnets → Create subnet`.
2. Параметры:
   - **VPC**: `student-vpc-kXX`
   - **Subnet name**: `private-subnet-kXX`
   - **Availability Zone**: другая зона
   - **IPv4 CIDR block**: `10.(k%30).2.0/24`
3. **Создать**.

<img width="1578" height="386" alt="{4DAED59F-BD30-4E0B-8B64-67684F99FC2E}" src="https://github.com/user-attachments/assets/f8f5413b-27b7-4311-b022-33ff09b6cf0a" />


> **Вопрос:** Является ли подсеть приватной?  
> **Ответ:** Пока нет, маршруты через IGW отсутствуют, но NAT ещё не настроен.

### Шаг 5. Создание таблиц маршрутов (Route Tables)

Теперь, когда есть две подсети (публичная и приватная), необходимо настроить маршруты (Route Tables), которые определяют, как сетевой трафик будет двигаться внутри нашей VPC.

По умолчанию каждая новая VPC имеет одну основную таблицу маршрутов, и все новые подсети автоматически к ней подключаются. Если зайти в `Route Tables`, можно увидеть одну таблицу, связанную с VPC.

Но для лучшей структуры и изоляции необходимо создать:

- отдельную таблицу маршрутов для публичной подсети (с доступом к Интернету через IGW),
- отдельную таблицу маршрутов для приватной подсети (с доступом к Интернету через NAT Gateway).

#### Шаг 5.1. Создание публичной таблицы маршрутов

1. `Route Tables → Create route table`.
2. Имя: `public-rt-kXX`, VPC: `student-vpc-kXX`.
3. В `Routes` добавить:
   - **Destination**: `0.0.0.0/0`
   - **Target**: `student-igw-kXX`
4. В `Subnet associations` подключить `public-subnet-kXX`.

<img width="1584" height="591" alt="{06E1AD55-F25F-438B-A9FC-E1EF0996FEF2}" src="https://github.com/user-attachments/assets/d532d6f2-d22b-4064-8b48-18a32da66892" />

   - Перейти на вкладку `Routes` и нажать `Edit routes` → `Add route`.
   - Заполнить:
      - `Destination`: 0.0.0.0/0
         > Это означает “весь остальной трафик, не относящийся к внутренним адресам VPC”.
      - `Target`: выбрать Internet Gateway (`student-igw-kXX`).
   - Нажать `Save changes`.

<img width="1616" height="674" alt="{6E4A5B37-EF92-4C7B-B63F-4E5D7BB0461A}" src="https://github.com/user-attachments/assets/358c4f19-8b2d-49e6-9f28-f2bfed2d9d19" />


   - Перейти на вкладку `Subnet associations` → `Edit subnet associations`.
   - Отметить `public-subnet-kXX` и нажать `Save associations`.

<img width="1613" height="644" alt="{2BA07A2E-2222-45F5-9DCF-0BCC48C350F9}" src="https://github.com/user-attachments/assets/1c62c9de-d504-4212-8f39-74264c67b96b" />

Теперь трафик из публичной подсети (например, от веб-сервера или NAT Gateway) будет отправляться наружу через Internet Gateway. Связь `“0.0.0.0/0 → IGW”` — именно то, что делает подсеть публичной.


> **Вопрос:** Зачем привязывать таблицу маршрутов?  
> **Ответ:** Чтобы подсеть знала, как направлять трафик, в том числе в Интернет.

#### Шаг 5.2. Создание приватной таблицы маршрутов

1. `Subnets → Create subnet`.
2. Параметры:
   - **VPC**: `student-vpc-kXX`
   - **Subnet name**: `private-subnet-kXX`
   - **Availability Zone**: другая зона
   - **IPv4 CIDR block**: `10.(k%30).2.0/24`
3. **Создать**.

<img width="1581" height="600" alt="{79310D90-E157-424E-8471-5BD20CBE8285}" src="https://github.com/user-attachments/assets/d8b11980-9c80-4d3f-a701-cb5b2452113a" />

На данный момент все ресурсы, которые будут созданы в приватной подсети, не смогут выходить в Интернет, так как на данный момент нет NAT Gateway и соответствующего маршрута.

### Шаг 6. Создание NAT Gateway

NAT Gateway позволяет ресурсам в приватной подсети выходить в Интернет (например, для обновления ПО), при этом оставаясь недоступными извне.

**Вопрос:** Как работает NAT Gateway?

**Ответ:** NAT Gateway (шлюз трансляции сетевых адресов) обеспечивает выход ресурсов из приватной подсети в Интернет, сохраняя их внутренние IP-адреса скрытыми. Он принимает исходящие запросы от приватных серверов, заменяет их внутренние IP на собственный публичный адрес и пересылает трафик во внешнюю сеть. Когда ответы приходят обратно, NAT Gateway перенаправляет их к соответствующим внутренним ресурсам, обеспечивая безопасное и прозрачное взаимодействие с Интернетом.

#### Шаг 6.1. Создание Elastic IP

Elastic IP — это постоянный публичный IPv4-адрес, закреплённый за вашим аккаунтом AWS.
Он используется, например, для NAT Gateway, чтобы служить “точкой выхода” в Интернет от имени всех ресурсов в приватной подсети.

Шаги по созданию Elastic IP:
В левой панели консоли AWS перейдите в раздел Elastic IPs.
Нажмите кнопку Allocate Elastic IP address.


<img width="1590" height="520" alt="{A535EB42-31B7-4379-ADD0-33EAA2097E2C}" src="https://github.com/user-attachments/assets/339fe1bb-fa38-453c-a59c-07deff875c7f" />

#### Шаг 6.2. Создание NAT Gateway

1. В левой панели необходимо выбрать `NAT Gateways` → `Create NAT gateway`.
2. Указать:
   - `Name tag`: `nat-gateway-kXX`
   - `Subnet`: выбрать публичную подсеть (`public-subnet-kXX`)
      > NAT Gateway всегда создаётся в публичной подсети, потому что ему нужен прямой выход в Интернет через IGW.
   - `Connectivity type`: `Public`
   - `Elastic IP allocation ID`: выбрать EIP, созданный на предыдущем шаге.
3. Нажать `Create NAT gateway`.

<img width="1613" height="672" alt="{633D898E-E2EC-4294-BD0A-4901C56C20D9}" src="https://github.com/user-attachments/assets/f7b9ecb0-422f-41cd-88fe-f2112ec80a28" />


Подождать 2–3 минуты, пока статус изменится с `Pending` на `Available`. Это значит, что NAT Gateway готов к работе.

<img width="1173" height="288" alt="{A7A26657-F674-4903-813E-A6BDCF857CC1}" src="https://github.com/user-attachments/assets/0a485228-ed61-4877-ada1-885e9bd6ecfe" />

#### Шаг 6.3. Изменение приватной таблицы маршрутов

1. Вернуться в `Route Tables` и выбрать `private-rt-kXX`.
2. Перейти на вкладку `Routes` и нажать `Edit routes` → `Add route`.
3. Заполнить:
   - `Destination`: `0.0.0.0/0`
   - `Target`: выбрать NAT Gateway (`nat-gateway-kXX`).
4. Нажать `Save changes`.

<img width="1579" height="691" alt="{0B93516E-78D6-4E33-8E7F-917302F63284}" src="https://github.com/user-attachments/assets/eecca62f-4388-4ab1-98ca-17f97cb235e4" />

Теперь ресурсы в приватной подсети смогут выходить в Интернет через NAT Gateway.

### Шаг 7. Создание Security Groups

Security Group (SG) — это виртуальный брандмауэр на уровне экземпляра (EC2), контролирующий входящий (Inbound) и исходящий (Outbound) сетевой трафик.
С его помощью можно определить, какой трафик разрешён для конкретного инстанса и откуда он может поступать.

В левой панели консоли AWS откройте раздел Security Groups → выберите Create security group.

Укажите параметры:
Security group name: web-sg-kXX
Description: Security group for web server
VPC: выберите вашу сеть student-vpc-kXX
В разделе Inbound rules добавьте разрешения для следующих типов трафика:
Тип: HTTP, Протокол: TCP, Порт: 80, Источник: 0.0.0.0/0
Тип: HTTPS, Протокол: TCP, Порт: 443, Источник: 0.0.0.0/0

<img width="1906" height="364" alt="{DEEA07C7-A46C-4C8E-BEC7-FFE7C14C77B0}" src="https://github.com/user-attachments/assets/9cdc89f5-3422-4b26-bcc8-6c606173b406" />


4. Создать еще две Security Groups:
   - `bastion-sg-kXX` для bastion host с разрешением входящего трафика на порт `22` (SSH) только из IP-адреса студента.
   - `db-sg-kXX` для базы данных с разрешением входящего трафика:
      - Тип: `MySQL/Aurora`, Протокол: `TCP`, Порт: `3306`, Источник: `web-sg-kXX` (разрешить доступ только с веб-сервера)
      - Тип: `SSH`, Протокол: `TCP`, Порт: `22`, Источник: `bastion-sg-kXX` (разрешить доступ только с bastion host)

<img width="1885" height="249" alt="{20D30C21-0DF0-44C2-BC86-F35988DDAF08}" src="https://github.com/user-attachments/assets/16f315fd-a58d-4e1e-8812-aba88faebe29" />
<img width="1605" height="374" alt="{0C2DC8D2-376A-45AF-AFEF-685F4FD364DF}" src="https://github.com/user-attachments/assets/4f63cd27-6a5a-4218-bce2-91f49d034790" />


> **Вопрос:** Что такое *Bastion Host* и зачем он нужен в архитектуре с приватными подсетями?
>
> **Ответ:** Bastion Host — это специально настроенный сервер, размещённый в публичной подсети и защищённый правилами безопасности. Он выполняет роль “точки входа” для администраторов, обеспечивая безопасное подключение к ресурсам, находящимся в приватных подсетях. Через Bastion Host можно получить SSH-доступ к внутренним серверам без необходимости открывать их напрямую в Интернет, что повышает уровень безопасности инфраструктуры.

### Шаг 8. Создание EC2-инстансов

Необходимо создать три EC2-инстанса, которые будут выполнять следующие роли:

- _Веб-сервер_ (`web-server`) - в публичной подсети, доступен из Интернета по HTTP.
- _Сервер базы данных_ (`db-server`) - в приватной подсети, недоступен напрямую извне.
- _Bastion Host_ (`bastion-host`) - в публичной подсети, для безопасного доступа к приватным ресурсам.

1. В строке поиска AWS Console ввести `EC2` и открыть консоль.
2. Создать 3 инстанса, следуя инструкциям ниже.

_Для всех инстансов использовать_:

- AMI: `Amazon Linux 2 AMI (HVM), SSD Volume Type`
- Тип инстанса: `t3.micro`
- Ключ доступа (Key Pair): создать новый ключ `student-key-kXX` и скачать его.
- Хранилище: оставить по умолчанию (8 ГБ).
- Теги: добавить тег `Name` с соответствующим именем инстанса.

<img width="745" height="701" alt="{98B14593-E266-4152-A86B-AF75A45B4449}" src="https://github.com/user-attachments/assets/7002c75a-69e5-4659-b642-3b4b6cf3bc53" />


Для `bastion-host`:

1. Выбрать сеть `VPC`: `student-vpc-kXX`
2. Подсеть `Subnet`: `public-subnet-kXX`
3. `Auto-assign Public IP`: `Enable`
4. `Security Group`: выберать `bastion-sg-kXX`
5. В разделе `Advanced Details` в поле `User data` вставить следующий скрипт для автоматической установки MySQL клиента:

   ```bash
   #!/bin/bash
   dnf install -y mariadb105
   ```

<img width="1914" height="324" alt="{C4BBCC28-3186-4A9F-94CA-6B003C789AA3}" src="https://github.com/user-attachments/assets/344f6709-ede3-4085-a87d-0e1c53495e82" />
<img width="1546" height="190" alt="{3B79E3F2-2E87-45AC-8722-1DCF231C2693}" src="https://github.com/user-attachments/assets/92b5964d-a108-4a47-8502-020004f0dc2a" />


Для `web-server`:

1. Выберать сеть `VPC`: `student-vpc-kXX`
2. Подсеть `Subnet`: `public-subnet-kXX`
3. `Auto-assign Public IP`: `Enable`
4. `Security Group`: выберать `web-sg-kXX`
5. В разделе `Advanced Details` в поле `User data` вставить следующий скрипт для автоматической установки веб-сервера:

   ```bash
   #!/bin/bash
   dnf install -y httpd php
   echo "<?php phpinfo(); ?>" > /var/www/html/index.php
   systemctl enable httpd
   systemctl start httpd
   ```


Для `db-server`:

1. Выбрать сеть `VPC`: `student-vpc-kXX`
2. Подсеть `Subnet`: `private-subnet-kXX`
3. `Auto-assign Public IP`: `Disable`
4. `Security Group`: выберать `db-sg-kXX`
5. В разделе `Advanced Details` в поле `User data` вставить следующий скрипт для автоматической установки MySQL сервера:

   ```bash
   #!/bin/bash
   dnf install -y mariadb105-server
   systemctl enable mariadb
   systemctl start mariadb
   mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword123!'; FLUSH PRIVILEGES;"
   ```

<img width="1551" height="710" alt="{4099EDBA-5464-43DB-B8CE-EEA9DE8F0BB9}" src="https://github.com/user-attachments/assets/069c2f4b-947c-4efb-923a-1544c23a6e4f" />



### Шаг 9. Проверка работы

На этом этапе уже созданы:

- виртуальная сеть (VPC)
- публичная и приватная подсети
- интернет-шлюз (IGW)
- NAT Gateway
- две таблицы маршрутов
- три экземпляра EC2 (Web, DB и Bastion)
- три Security Group

Теперь важно убедиться, что сеть функционирует корректно и что приватная подсеть действительно изолирована от внешнего мира.

1. Необходимо подождать, пока все инстансы запустятся (статус `running`).
2. Найти публичный IP-адрес `web-server` и открыть его в браузере. Должна появиться страница с информацией о PHP.

![image](https://i.imgur.com/Alvs7QM.png)

3. Необходимо подключититься к `bastion-host` по SSH:

   ```bash
   ssh -i <your-nickname>-key.pem ec2-user@<Bastion-Host-Public-IP>
   ```

![image](https://i.imgur.com/SfvuRNE.png)

4. Проверить подключение к интернету с `bastion-host` выполнив `ping`:

   ```bash
      ping -c 4 google.com
   ```

   > Если пинги успешны, значит публичная подсеть и IGW настроены правильно.

![image](https://i.imgur.com/So17qnB.png)

5. С `bastion-host` попробовать подключиться к `db-server`

Перед подключением к экземпляру EC2 необходимо убедиться, что приватный ключ `.pem` имеет правильные права доступа.
Если права слишком открытые, `SSH` не позволит подключиться.

```bash
chmod 400 student-key.pem
```

Команда делает файл доступным только для чтения владельцем.
Это требование системы безопасности `AWS` — иначе подключение будет отклонено как небезопасное.

После этого необходимо установить и настроить сервер базы данных `MariaDB`.

Для этого через Bastion Host выполняется подключение к приватному серверу по SSH и установка пакета:

```bash
sudo dnf install -y mariadb105-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

Если служба MariaDB работает, её статус отображается как `active (running)`.

После установки MariaDB и запуска службы можно убедиться, что она слушает порт 3306 (порт по умолчанию для MySQL/MariaDB).
Для этого используется команда:

```bash
sudo ss -ltnp | grep 3306
```

![image](https://i.imgur.com/Sj8TCHp.png)

Далее выполняется подключение к базе данных под пользователем `root` и создание отдельного пользователя для работы приложения.

```bash
mysql -u root -p
```

![image](https://i.imgur.com/zvBiSR5.png)
![image](https://i.imgur.com/hQdfJeZ.png)

После входа в консоль MariaDB создаётся пользователь `lab`, которому разрешено подключаться из подсетей `10.*.*.*` (внутренняя сеть VPC):

```sql
CREATE USER 'lab'@'10.%' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON *.* TO 'lab'@'10.%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

После успешного подключения к Bastion Host необходимо проверить доступность базы данных, расположенной в приватной подсети.

Для этого выполняется подключение к MariaDB с использованием созданного пользователя `lab`:

   ```bash
   mysql -h 10.1.2.20 -u lab -p
   ```

![image](https://i.imgur.com/4eF3xON.png)

Успешное подключение к MariaDB через Bastion Host подтверждает корректную работу маршрутов, NAT Gateway и Security Groups.

6. Выйти из `db-server` и `bastion-host`.

### Шаг 10. Дополнительные задания. Подключение в приватную подсеть через Bastion Host

Так как приватная подсеть не имеет выхода в интернет, подключение к `db-server` выполняется через промежуточный сервер — `bastion-host`.

1. На локальной машине необходимо выполнить следующие команды. Это запустит SSH Agent и добавит приватный ключ в агент:

В PowerShell необходимо запустить службу SSH Agent и добавить приватный ключ:

```bash
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add .\student-key-k1.pem
```

2. Необходимо подключиться к bastion-host с опцией `-A` и `-J`:

Подключение выполняется с опциями `-A` и `-J`, которые позволяют использовать переадресацию `SSH`-ключа и указать промежуточный хост:

```bash
ssh -A -J ec2-user@3.123.128.242 ec2-user@10.1.2.20
```

`-A` — включает переадресацию ключей (Agent Forwarding);

`-J` — указывает bastion как промежуточный узел.

![image](https://i.imgur.com/yVWNOfY.png)

3. Обновить систему на `db-server`:

```bash
sudo dnf update -y
```

4. Установить `htop`:

```bash
sudo dnf install -y htop
```

> Если обновление и установка прошли успешно, значит `NAT Gateway` работает корректно.

![image](https://i.imgur.com/RM7CsHi.png)

5. Подключиться к `MySQL` серверу:

```bash
mysql -u root -p
```

![image](https://i.imgur.com/AR7axSA.png)

> Если `MySQL` не установлен, выполните `sudo dnf install -y mariadb105`. Пароль `StrongPassword123!`.

6. Выйти из `MySQL` и затем из `db-server` и `bastion-host`.

7. На локальном компьютере, завершить работу с `SSH Agent`:

```bash
ssh-add -D
Stop-Service ssh-agent
```

## Завершение работы

После выполнения всех шагов, необходимо удалить созданные ресурсы в `AWS`, чтобы избежать ненужных затрат. Выполнить удаление в следующем порядке:

1. Удалить `EC2`-инстансы.

![image](https://i.imgur.com/rXHUkmA.png)

2. Удалить `NAT Gateway` (подождите, пока он будет удалён).

![image](https://i.imgur.com/7QMZHr5.png)

3. Удалить `Elastic IP`.

   - `VPC` -> `Elastic IPs` -> выберать `EIP` -> `Actions` -> `Release Elastic IP addresses`.

![image](https://i.imgur.com/Y6990LO.png)

4. Удалить `Security Groups`.

   - `VPC` -> `Security Groups` -> выберать нужную группу -> `Actions` -> `Delete Security Group`.

![image](https://i.imgur.com/P3VTruc.png)

5. Удалить `Internet Gateway`.

   - `VPC` -> `Internet Gateways` -> выберать `IGW` -> `Actions` -> `Detach from VPC` -> подтвердить.
   - Затем снова выберать IGW -> `Actions` -> `Delete internet gateway`.

![image](https://i.imgur.com/weNxyew.png)
![image](https://i.imgur.com/n2qekV6.png)

6. Удалить созданную `VPC`.

   - `VPC` -> `Your VPCs` -> выберите вашу VPC -> `Actions` -> `Delete VPC`.
   - Подтвердите удаление.
   - Если удаление не удаётся, проверьте, что все ресурсы (подсети, таблицы маршрутов и т.д.) были удалены.

![image](https://i.imgur.com/2TCt9b2.png)

> Удаление ресурсов в неправильном порядке может привести к ошибкам, так как некоторые ресурсы зависят от других.

Для того, чтобы кредиты не снимались достаточно удалить `EC2` инстансы, `NAT Gateway` и `Elastic IP`. Остальные ресурсы можно оставить, так как они не тарифицируются отдельно.

## Выводы

В ходе лабораторной работы №3 я создал и настроил облачную сеть (VPC) в AWS, обеспечив безопасное взаимодействие между публичной и приватной подсетями.

В работе были выполнены следующие шаги:
Создана виртуальная сеть с публичной и приватной подсетями.
Настроен интернет-шлюз (Internet Gateway) для выхода в интернет.
Развернут NAT Gateway, чтобы приватные ресурсы могли безопасно обращаться к внешним сервисам.
Настроены таблицы маршрутов для корректного движения сетевого трафика.
Созданы группы безопасности (Security Groups) для контроля входящего и исходящего трафика.
Развернуты и проверены EC2-инстансы: веб-сервер, база данных и Bastion Host.
В результате было проверено:
взаимодействие публичных и приватных ресурсов,
работа маршрутизации и NAT Gateway,
безопасность доступа через группы безопасности.
Я получил практические навыки создания и настройки виртуальной сети, подключения к приватным ресурсам через Bastion Host и понимание принципов изоляции и безопасности в облачной архитектуре.

## Библиография

1. [AWS Documentation](https://docs.aws.amazon.com/) — официальная документация по сервисам Amazon Web Services.
2. [Amazon VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) — руководство пользователя по созданию и настройке виртуальных частных сетей в AWS.
3. [Amazon EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html) — руководство пользователя по работе с виртуальными машинами EC2.







