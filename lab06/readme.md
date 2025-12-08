# Лабораторная работа №6  
**Тема:** Балансирование нагрузки в облаке и авто-масштабирование  
---

## Цель работы

Закрепить навыки работы с AWS EC2, Elastic Load Balancer, Auto Scaling и CloudWatch, создав отказоустойчивую и автоматически масштабируемую архитектуру.

**Задачи лабораторной работы:**

- Развёртывание VPC с публичными и приватными подсетями.  
- Создание EC2-инстанса с веб-сервером и PHP.  
- Настройка Application Load Balancer.  
- Настройка Auto Scaling Group с политикой масштабирования.  
- Тестирование отказоустойчивости и автоматического масштабирования.

**Структура проекта:**
```
lab6/
├── init.sh           # Скрипт настройки EC2
├── main.tf           # Основной Terraform конфиг
├── variables.tf      # Переменные Terraform
├── outputs.tf        # Выводы Terraform
└── terraform.tfvars  # Значения переменных
```
---

## Ход выполнения работы

### 1. Подготовка инфраструктуры с Terraform

#### 1.1 `main.tf`
```terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  required_version = ">= 1.5.0"
}

provider "aws" {
  region = var.aws_region
}

# Доступные зоны (чтобы разбросать подсети по разным AZ)
data "aws_availability_zones" "available" {
  state = "available"
}

# Ищем последнюю Amazon Linux 2 AMI (x86_64, gp2)
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# ========== VPC ==========
resource "aws_vpc" "lab6_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "lab6-vpc"
    Lab  = "cloud-computing-6"
  }
}

# ========== Internet Gateway ==========
resource "aws_internet_gateway" "lab6_igw" {
  vpc_id = aws_vpc.lab6_vpc.id

  tags = {
    Name = "lab6-igw"
  }
}

# ========== Публичные подсети ==========
resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.lab6_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true

  tags = {
    Name = "lab6-public-1"
    Type = "public"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.lab6_vpc.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = data.aws_availability_zones.available.names[1]
  map_public_ip_on_launch = true

  tags = {
    Name = "lab6-public-2"
    Type = "public"
  }
}

# ========== Приватные подсети (для Auto Scaling позже) ==========
resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.lab6_vpc.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "lab6-private-1"
    Type = "private"
  }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.lab6_vpc.id
  cidr_block        = "10.0.12.0/24"
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name = "lab6-private-2"
    Type = "private"
  }
}

# ========== Route table для публичных подсетей ==========
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.lab6_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lab6_igw.id
  }

  tags = {
    Name = "lab6-public-rt"
  }
}

resource "aws_route_table_association" "public_1_assoc" {
  subnet_id      = aws_subnet.public_1.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_2_assoc" {
  subnet_id      = aws_subnet.public_2.id
  route_table_id = aws_route_table.public.id
}

# ========== Security Group для веб-сервера ==========
resource "aws_security_group" "web_sg" {
  name        = "lab6-web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.lab6_vpc.id

  # SSH — по-хорошему тут должен быть твой IP, но для удобства сейчас 0.0.0.0/0
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP для всех
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "lab6-web-sg"
  }
}

# ========== EC2: Amazon Linux 2 + init.sh ==========
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public_1.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  associate_public_ip_address = true
  monitoring                  = true # Detailed CloudWatch monitoring (Enable)

  key_name = "lab6-key"

  # UserData — скрипт init.sh, который тебе дал препод
  user_data = file("${path.module}/init.sh")
  user_data_replace_on_change = true
  tags = {
    Name = "lab6-web-ec2"
    Role = "web"
  }
}
```

#### 1.2 `variables.tf`
```tf
variable "aws_region" {
  description = "AWS region for lab 6"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type for web server"
  type        = string
  default     = "t3.micro"
}
```

#### 1.3 `terraform.tfvars`
```tfvars
aws_region    = "us-east-1"
instance_type = "t3.micro"
```

#### 1.4 `outputs.tf`
```output "vpc_id" {
  description = "ID созданной VPC"
  value       = aws_vpc.lab6_vpc.id
}

output "public_subnet_ids" {
  description = "IDs публичных подсетей"
  value = [
    aws_subnet.public_1.id,
    aws_subnet.public_2.id,
  ]
}

output "private_subnet_ids" {
  description = "IDs приватных подсетей"
  value = [
    aws_subnet.private_1.id,
    aws_subnet.private_2.id,
  ]
}

output "web_instance_public_ip" {
  description = "Public IP веб-сервера"
  value       = aws_instance.web_server.public_ip
}

output "web_instance_public_dns" {
  description = "Public DNS веб-сервера"
  value       = aws_instance.web_server.public_dns
}
```

#### 1.5 Скрипт `init.sh`
```#!/bin/bash
set -xe

# Ждём, пока yum не будет занят другим процессом
while sudo fuser /var/run/yum.pid >/dev/null 2>&1; do
  echo "YUM занят, ждём 5 секунд..."
  sleep 5
done

# Обновление системы
yum update -y

# Установка Nginx через Amazon Linux Extras + PHP-FPM
amazon-linux-extras install -y nginx1
yum install -y php php-fpm php-cli

# Включаем автозапуск сервисов
systemctl enable nginx
systemctl enable php-fpm

# Создаём директорию под сайт
mkdir -p /var/www/html

cat > /var/www/html/index.php <<'EOF'
<?php
$hostname = gethostname();

if (strpos($_SERVER['REQUEST_URI'], '/load') === 0) {
    runCpuLoad();
    exit;
}

function runCpuLoad(): void
{
    ini_set('max_execution_time', '600');

    $seconds = isset($_GET['seconds']) ? (int)$_GET['seconds'] : 60;
    $seconds = max(30, min($seconds, 600)); // от 30 до 600 сек

    $endTime = microtime(true) + $seconds;
    $dummy   = 0.0;

    while (microtime(true) < $endTime) {
        $dummy += sqrt(mt_rand(1, 1000));
    }

    echo "<h2>Load finished</h2>";
    echo "<p>Seconds: {$seconds}</p>";
    echo "<p>Dummy value: {$dummy}</p>";
}

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lab 6 Web Server</title>
</head>
<body>
<h1>Hello from <?php echo htmlspecialchars($hostname, ENT_QUOTES); ?></h1>
<p><a href="/load?seconds=60">Generate CPU load for 60 seconds</a></p>
</body>
</html>
EOF

# Конфиг Nginx для PHP
cat > /etc/nginx/conf.d/lab6.conf <<'EOF'
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

# Чистим дефолтные конфиги (если есть)
rm -f /etc/nginx/conf.d/*.default

# Меняем права на каталог сайта
chown -R nginx:nginx /var/www/html

# Проверяем конфиг и запускаем сервисы
nginx -t
systemctl restart php-fpm
systemctl restart nginx
```

### 2. Развёртывание инфраструктуры

```bash
terraform init
terraform apply
```

<img width="640" height="102" alt="{A5BD82BC-7B87-475D-B0E7-E6FEF8C30CDF}" src="https://github.com/user-attachments/assets/a36ecf9e-8ea9-4a82-a205-d91d74d35db6" />
<img width="722" height="276" alt="{43D0583B-FAE4-436D-AA1D-F3D995ABA8D3}" src="https://github.com/user-attachments/assets/33a47b8d-c5dc-433c-b383-d8b493a64c9a" />
<img width="597" height="613" alt="{3CB844BA-C922-48D2-BAFA-3A313A0156B2}" src="https://github.com/user-attachments/assets/51d89072-342f-49aa-96ab-b76fc843d5c5" />
<img width="747" height="295" alt="{5C6510F1-EB7B-4CF9-BBD6-A22988BEC477}" src="https://github.com/user-attachments/assets/623eb606-ce1e-4701-9f62-8c642503d254" />

---

### 3. Создание AMI

Для создания образа виртуальной машины (AMI) была выполнена следующая последовательность действий:

1. В консоли AWS открыт раздел **EC2 → Instances**, выбран инстанс с уже установленным и настроенным веб-сервером (`lab6-web-ec2`).  
2. Через меню **Actions → Image and templates → Create image** создан новый образ. В качестве имени образа указано `project-web-server-ami`. При необходимости добавлено описание, чтобы было понятно, что образ содержит настроенный веб-сервер с nginx и PHP. Выбрана опция перезагрузки инстанса для корректного сохранения данных файловой системы.  
3. После подтверждения создания, в списке **AMIs** появился новый объект со статусом `Pending`. Через несколько минут AWS завершил создание образа, после чего статус изменился на `Available`.  

<img width="1059" height="496" alt="{C626F1DE-83A6-45F2-A5D4-F236A70E51BA}" src="https://github.com/user-attachments/assets/9b2f9fae-2bbd-42a7-ae23-362057c7a1fe" />
<img width="1227" height="129" alt="{BE645F26-AC61-4A65-B6A4-516A8C60EF0C}" src="https://github.com/user-attachments/assets/c0dc76ac-4be3-46cb-9aed-c5d84812b5ce" />

---

### 4. Создание Launch Template

- EC2 → Launch Templates → Create → Название `project-launch-template`.  
- AMI: `project-web-server-ami`.  
- Instance type: `t3.micro`.  
- Security Group: `lab6-web-sg`.  
- Detailed CloudWatch monitoring и user data включены.

<img width="513" height="210" alt="{F9838F35-D577-4B54-9BC8-3B463571A28E}" src="https://github.com/user-attachments/assets/81504351-dcdf-4ec6-9190-3b4c3b85d119" />
<img width="1229" height="118" alt="{3851DEEA-E7A1-4339-AC7C-1527C4356940}" src="https://github.com/user-attachments/assets/477343fd-110d-432e-9ca8-d56d30ae0b4e" />

---

### 5. Создание Target Group

- EC2 → Target Groups → Create → Name: `project-target-group`.  
- Protocol: HTTP, Port: 80, Target type: Instances, VPC: lab6-vpc.  
- Health check: `/`, timeout 5s, interval 30s, healthy threshold 5, unhealthy threshold 2.

#### 5.1. Настройка параметров Target Group
В открывшемся окне необходимо задать следующие параметры:

- **Target type:** Instances  
- **Target group name:** project-target-group  
- **Protocol:** HTTP  
- **Port:** 80  
- **VPC:** выбрать VPC, созданную ранее с помощью Terraform  

#### 5.2. Настройка Health Checks
Health checks — это механизм проверки состояния инстансов, который позволяет Load Balancer определить, какие серверы готовы обрабатывать запросы.  

Параметры оставляем по умолчанию:  
- **Health check protocol:** HTTP  
- **Path:** /  
- **Timeout:** 5 секунд  
- **Interval:** 30 секунд  
- **Healthy threshold:** 5  
- **Unhealthy threshold:** 2  
- **Success codes:** 200  

**Пояснения:**  
- Health check отправляет HTTP-запросы на указанный путь (`/`) и проверяет код ответа.  
- Если инстанс отвечает корректно (код 200) указанное количество раз, он считается **healthy** и Load Balancer направляет на него трафик.  
- Если инстанс не отвечает корректно указанное количество раз, он помечается как **unhealthy**, и трафик на него не направляется.  

<img width="1114" height="748" alt="{8B6A027F-54CE-4BF6-A408-99B104919F46}" src="https://github.com/user-attachments/assets/f52d7398-fb68-4061-99e5-c65d9790a3ba" />

<img width="1350" height="510" alt="{63903D0F-8467-4E23-BDB6-0F6E6EDFB10E}" src="https://github.com/user-attachments/assets/c6bba4f4-5c5f-4468-bd88-56c7a5d821c6" />
<img width="1426" height="713" alt="{39B1C742-43ED-4A02-A845-A2DF1698BCBA}" src="https://github.com/user-attachments/assets/10360ee8-5d7e-45ca-919b-2a9444d4118b" />
<img width="1892" height="432" alt="{009351CA-EF78-44F7-9B6E-3391A62E275B}" src="https://github.com/user-attachments/assets/6ab3637c-a247-41aa-a123-c962f4665880" />


---

### 6. Создание Application Load Balancer

- EC2 → Load Balancers → Create → Application Load Balancer → `project-alb`.  
- Scheme: Internet-facing, IPv4.  
- Subnets: две публичные подсети.  
- Security Group: `lab6-web-sg`.  
- Listener: HTTP 80 → Forward to `project-target-group`.

<img width="1672" height="689" alt="{F7F3D452-F60A-4B2E-B76E-433D900BC548}" src="https://github.com/user-attachments/assets/5859e199-96e8-47e4-979f-6d9cc7eb0824" />
<img width="1879" height="371" alt="{40C03634-310F-4A09-8BC8-47626628A7EA}" src="https://github.com/user-attachments/assets/ce9429ed-60ac-4e96-864b-f7c38b3e3bcd" />


---

### 7. Создание Auto Scaling Group

- EC2 → Auto Scaling Groups → Create → Name: `project-auto-scaling-group`.  
- Launch Template: `project-launch-template`.  
- Subnets: две приватные подсети.  
- Size: Desired 2, Min 2, Max 4.  
- Scaling Policy: Target Tracking → Average CPU Utilization 50%, Instance warm-up 60s.
- CloudWatch Metrics включены.

<img width="1338" height="712" alt="{12A11B99-B554-4D21-8CEA-6AEEA374150C}" src="https://github.com/user-attachments/assets/c408fd22-3b9e-4008-b60e-e3696ee1fa77" />
<img width="1309" height="482" alt="{D56DCCB7-D119-4DEE-A303-574202D4D631}" src="https://github.com/user-attachments/assets/005d217f-d7e2-4d99-8c32-4f28b4b25ffe" />

---

### 8. Тестирование Load Balancer

После создания Load Balancer важно не только открыть его DNS в браузере, но и выполнить ряд проверок и дополнительных тестов, чтобы убедиться в корректной работе балансировщика и его интеграции с Target Group и EC2-инстансами.

**8.1. Проверка DNS и базовый HTTP-тест**

- Получить DNS ALB (например: `project-alb-xxxx.elb.amazonaws.com`) и проверить доступность в браузере: `http://<ALB-DNS>`.
- Быстрая проверка заголовков:

```bash
curl -I http://project-alb-xxxx.elb.amazonaws.com
```

Эта команда покажет HTTP-статус и заголовки ответа — убедитесь, что приходит `200 OK`.

**8.2. Проверка распределения трафика и round-robin**

- Обновляйте страницу несколько раз или выполните цикл запросов из терминала, чтобы убедиться, что ответы приходят от разных внутренних инстансов (в `index.php` выводится имя хоста):

```bash
for i in {1..10}; do curl -s http://project-alb-xxxx.elb.amazonaws.com | grep "Hello from"; done
```

- Видимые разные `ip-10-0-...` означают, что ALB корректно распределяет трафик между здоровыми инстансами.

**8.3. Health checks и статус Targets**

- В консоли EC2 → Target Groups откройте ваш `project-target-group` и проверьте вкладку **Targets** — там видно состояние (Healthy/Unhealthy) каждого инстанса и последние результаты health check.
- Параметры health check (path `/`, timeout, interval, thresholds) определяют, как быстро инстанс считается недоступным.

**8.4. Логирование и доступность логов ALB**

- Для детального анализа можно включить **Access Logs** у ALB (S3-бакет) — это позволит просматривать все запросы, ответные коды и задержки.
- Включение логов: Load Balancers → ваш ALB → Attributes → Access logs → Enable → указать S3-бакет.

**8.5. Поведение при deregistration и connection draining**

- При удалении инстанса из Target Group (scale-in) ALB учитывает `deregistration delay` — время, в течение которого соединения корректно завершаются и новые соединения не направляются на этот инстанс.
- Это предотвращает потерю запросов при масштабировании.

**8.6. Диагностика проблем**

- Если ALB возвращает 5xx или 504, проверьте лог `php-fpm` и `nginx` на инстансах и параметры health check.
- Проверьте security group: ALB должен иметь разрешение на исходящие соединения к порту 80 инстансов, а инстансы должны разрешать входящий HTTP от ALB.

---

### 9. Тестирование Auto Scaling

Тестирование Auto Scaling направлено на проверку корректности срабатывания масштабирования при высокой нагрузке и возврата к исходному количеству инстансов после снижения нагрузки.

**9.1. Подготовка к нагрузочному тесту**

- На каждом инстансе доступен скрипт `index.php`/`load.php`, который генерирует CPU-нагрузку по запросу `/load?seconds=XX`.
- Для генерации более реалистичной нагрузки рекомендуется использовать специальные инструменты: `ab`, `hey` или многопоточную сборку запросов через `curl.sh`.

**9.2. Настройка и проверка CloudWatch и Alarm'ов**

- Перейдите в CloudWatch → Alarms и убедитесь в наличии правил:
  - `AlarmHigh` — срабатывает при CPUUtilization > 50% в течение 3 datapoints (1 мин по дефолту) → приводит к scale-out.
  - `AlarmLow` — срабатывает при CPUUtilization < 37.5% в течение 15 datapoints → приводит к scale-in.
- Убедитесь, что в Auto Scaling Group включена агрегация метрик в CloudWatch (Group metrics collection).

**9.3. Запуск нагрузки и наблюдение за масштабированием**

- Пример генерации нагрузки через `hey` (если установлен):

```bash
hey -z 120s -c 50 http://project-alb-xxxx.elb.amazonaws.com/load?seconds=10
```

- Альтернативно, использовать `curl.sh` для запуска нескольких потоков curl.

- Наблюдайте в CloudWatch график CPU для ASG (Metrics → By Auto Scaling Group). Когда средняя CPUUtilization превысит целевой порог (50%), Target Tracking policy инициирует scale-out.

**9.4. Поведение Auto Scaling при scale-out**

- Auto Scaling создаёт новые инстансы по Launch Template. Новые инстансы проходят регистрацию в Target Group, выполняют health checks и только после перехода в статус `Healthy` получают трафик.
- Параметр `Instance warm-up` (например, 60s) задаёт период, в течение которого метрики нового инстанса не учитываются в расчёте. Это предотвращает преждевременные решения о дальнейшем масштабировании.

**9.5. Scale-in и политика удаления инстансов**

- После снижения нагрузки Auto Scaling выполняет scale-in в соответствии с заданной политикой. Рекомендуется ознакомиться с политиками завершения (termination policies), чтобы управлять тем, какие инстансы будут завершены первыми.
- Можно включить **Scale-in protection** для критичных инстансов, чтобы временно предотвратить их завершение.

**9.6. Лайфсайкл эвенты и логирование**

- В разделе Auto Scaling → Activity history видны все активности: запуски, регистрации в Target Group, ошибки health check и завершения инстансов. Это основной источник правды при диагностике.
- Полезно настроить уведомления (SNS) на события Auto Scaling для оповещений о масштабировании.

**9.7. Ожидаемая временная диаграмма**

- Запуск нагрузки → CPU растёт в течение 1–2 минут → CloudWatch фиксирует превышение порога → AlarmHigh переходит в состояние InAlarm → Auto Scaling запускает новые инстансы (обычно несколько минут на инициализацию и регистрацию) → после регистрации и перехода в Healthy трафик распределяется на новые инстансы.
- После окончания нагрузки CPU падает → AlarmLow срабатывает → Auto Scaling уменьшает количество инстансов по политике.

**9.8. Рекомендации и безопасность**

- Для точных тестов используйте Detailed Monitoring (1-minute metrics).  
- Учитывайте стоимость: масштабирование увеличивает расходы — не забывайте очищать ресурсы после тестов.  
- Настройте корректные значения `deregistration delay`, `health check` и `instance warm-up` для предотвращения фальшивых срабатываний.

<img width="1468" height="636" alt="{19D43188-64ED-451F-959F-EEEF72F34FDB}" src="https://github.com/user-attachments/assets/d8a6a96e-f209-4756-a395-1ec1ebe29ae9" />
<img width="879" height="22" alt="{15EF25A3-778E-431F-9ECF-4B1C7D049A83}" src="https://github.com/user-attachments/assets/ede21bcb-2a11-46c2-a338-e1ad574010e3" />
<img width="1542" height="40" alt="{E7871744-233F-4E1C-9ACB-09E3BFC24AB4}" src="https://github.com/user-attachments/assets/3faf5714-a618-4710-8e31-86c890cef958" />

---

## Вопросы и ответы

**Вопрос:** Что такое AMI (image) и чем он отличается от snapshot?

**Ответ:** AMI — это готовый образ виртуальной машины, содержащий ОС, установленные программы и настройки; из него можно запускать новые EC2-инстансы. Snapshot — это резервная копия EBS-тома (диска), содержащая данные, но не представляющая собой полноценный загрузочный образ с настройками.

---

**Вопрос:** Что такое Launch Template и в чём отличие от Launch Configuration?

**Ответ:** Launch Template — современный шаблон конфигурации EC2 (AMI, тип, диски, сеть, user data и т.д.) с поддержкой версий; используется для ASG, Spot, ручных запусков. Launch Configuration — устаревший механизм для ASG без версий, после создания изменить его нельзя.

---

**Вопрос:** Какую роль выполняет Target Group?

**Ответ:** Target Group — список целей (EC2-инстансов, IP или Lambda), на которые ALB направляет трафик; она также управляет health checks и решает, какие инстансы считаются доступными (Healthy).

---

**Вопрос:** В чём разница между Internet-facing и Internal Load Balancer?

**Ответ:** Internet-facing имеет публичный доступ и публичный DNS; Internal доступен только внутри VPC и не имеет публичного интерфейса.

---

**Вопрос:** Что такое Default action у Load Balancer и какие бывают типы действий?

**Ответ:** Default action — действие по умолчанию, выполняемое если ни одно правило не сработало. Типы: forward (отправить на target group), redirect (перенаправление на URL), fixed-response (вернуть фиксированный ответ).

---

**Вопрос:** Почему в Auto Scaling Group выбирают приватные подсети?

**Ответ:** Приватные подсети повышают безопасность: инстансы не имеют публичного IP и доступны только через балансировщик (ALB) в публичных подсетях, что исключает обход точек входа.

---

**Вопрос:** Что такое Instance warm-up и зачем он нужен?

**Ответ:** Instance warm-up — период, в течение которого новые инстансы считаются «разогревающимися» и их метрики не учитываются в расчётах масштабирования; это предотвращает ошибочные срабатывания scaling-политик.

---

**Вопрос:** Какую роль выполняет Auto Scaling в процессе?

**Ответ:** Auto Scaling автоматически увеличивает (scale-out) или уменьшает (scale-in) количество инстансов в группе в соответствии с заданными метриками и политиками, поддерживая производительность и оптимизируя расходы.

---

## Выводы

- Лабораторная работа позволила освоить автоматизированное развёртывание AWS-инфраструктуры с помощью Terraform.  
- Созданы отказоустойчивые EC2-инстансы с веб-сервером через ALB.  
- Настроена Auto Scaling Group с динамическим масштабированием по CPU.  
- Проверена работа Load Balancer и Auto Scaling на примере генерации CPU-нагрузки.  
- Инфраструктура развёрнута полностью через Terraform без ручного создания ресурсов.

**Итог:** архитектура готова к эксплуатации, система демонстрирует отказоустойчивость и автоматическое масштабирование.

---
**Приложения:**

- Скриншоты интерфейса AWS (ALB, ASG, Target Group, AMI).  
- Логи Terraform и CloudWatch.

  ## Библиография

1. **Amazon EC2 Documentation** — официальная документация по работе с виртуальными машинами Amazon EC2, созданию и использованию AMI, Launch Templates, настройке сетевых параметров и жизненному циклу инстансов.  
   [https://docs.aws.amazon.com/ec2/](https://docs.aws.amazon.com/ec2/)

2. **Elastic Load Balancing Documentation** — руководство по созданию и настройке Application Load Balancer, target groups, listeners и распределению трафика между EC2-инстансами.  
   [https://docs.aws.amazon.com/elasticloadbalancing/](https://docs.aws.amazon.com/elasticloadbalancing/)

3. **Amazon CloudWatch Documentation** — документация по мониторингу ресурсов AWS, настройке метрик, графиков загрузки CPU, тревог (Alarms) и интеграции с Auto Scaling.  
   [https://docs.aws.amazon.com/cloudwatch/](https://docs.aws.amazon.com/cloudwatch/)

4. **Amazon VPC Documentation** — справочник по созданию VPC, подсетей, маршрутов, Internet Gateway, Security Groups и сетевой архитектуре, использованной для развертывания приложения.  
   [https://docs.aws.amazon.com/vpc/](https://docs.aws.amazon.com/vpc/)

5. **Amazon Auto Scaling Documentation** — официальное руководство по настройке Auto Scaling Group, политик масштабирования (Target Tracking), параметра warm-up и автоматическому scale-in/scale-out.  
   [https://docs.aws.amazon.com/autoscaling/](https://docs.aws.amazon.com/autoscaling/)

