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
```tf
# Пример: провайдер AWS, VPC, подсети, SG, EC2 с user_data
terraform {
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

# Создание VPC, подсетей, Internet Gateway, Security Group и EC2
# ...
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
```tf
output "vpc_id" { value = aws_vpc.lab6_vpc.id }
output "public_subnet_ids" { value = [aws_subnet.public_1.id, aws_subnet.public_2.id] }
output "private_subnet_ids" { value = [aws_subnet.private_1.id, aws_subnet.private_2.id] }
output "web_instance_public_ip" { value = aws_instance.web_server.public_ip }
output "web_instance_public_dns" { value = aws_instance.web_server.public_dns }
```

#### 1.5 Скрипт `init.sh`
```bash
#!/bin/bash
set -xe
yum update -y
amazon-linux-extras install -y nginx1
yum install -y php php-fpm php-cli
systemctl enable nginx php-fpm
mkdir -p /var/www/html
cat > /var/www/html/index.php <<'EOF'
<?php
$hostname = gethostname();
if (strpos($_SERVER['REQUEST_URI'], '/load') === 0) {
    ini_set('max_execution_time', '600');
    $seconds = isset($_GET['seconds']) ? (int)$_GET['seconds'] : 60;
    $endTime = microtime(true) + $seconds;
    $dummy = 0.0;
    while (microtime(true) < $endTime) { $dummy += sqrt(mt_rand(1, 1000)); }
    echo "<h2>Load finished</h2><p>Seconds: {$seconds}</p><p>Dummy: {$dummy}</p>";
    exit;
}
?>
<h1>Hello from <?php echo htmlspecialchars($hostname, ENT_QUOTES); ?></h1>
<p><a href="/load?seconds=60">Generate CPU load</a></p>
EOF
cat > /etc/nginx/conf.d/lab6.conf <<'EOF'
server { listen 80; root /var/www/html; index index.php; location / { try_files $uri $uri/ /index.php?$args; } location ~ \.php$ { include /etc/nginx/fastcgi_params; fastcgi_pass 127.0.0.1:9000; fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; } }
EOF
chown -R nginx:nginx /var/www/html
nginx -t
systemctl restart php-fpm nginx
```

### 2. Развёртывание инфраструктуры

```bash
terraform init
terraform apply
```

**Результат:** созданы VPC, подсети, Internet Gateway, Security Group, EC2-инстанс с nginx и PHP.

Проверка веб-сервера через `http://<public_ip>`: страница загружается, ссылка `Generate CPU load` создаёт нагрузку.

---

### 3. Создание AMI

1. EC2 → Instances → Выбран EC2 `lab6-web-ec2`.  
2. Actions → Image and templates → Create image → Название `project-web-server-ami`.  
3. После создания AMI статус `Pending` → `Available`.

**Примечание:** AMI — шаблон виртуальной машины с ОС, настройками и ПО. Snapshot — резервная копия EBS-диска.

---

### 4. Создание Launch Template

- EC2 → Launch Templates → Create → Название `project-launch-template`.  
- AMI: `project-web-server-ami`.  
- Instance type: `t3.micro`.  
- Security Group: `lab6-web-sg`.  
- Detailed CloudWatch monitoring и user data включены.

---

### 5. Создание Target Group

- EC2 → Target Groups → Create → Name: `project-target-group`.  
- Protocol: HTTP, Port: 80, Target type: Instances, VPC: lab6-vpc.  
- Health check: `/`, timeout 5s, interval 30s, healthy threshold 5, unhealthy threshold 2.

---

### 6. Создание Application Load Balancer

- EC2 → Load Balancers → Create → Application Load Balancer → `project-alb`.  
- Scheme: Internet-facing, IPv4.  
- Subnets: две публичные подсети.  
- Security Group: `lab6-web-sg`.  
- Listener: HTTP 80 → Forward to `project-target-group`.

---

### 7. Создание Auto Scaling Group

- EC2 → Auto Scaling Groups → Create → Name: `project-auto-scaling-group`.  
- Launch Template: `project-launch-template`.  
- Subnets: две приватные подсети.  
- Size: Desired 2, Min 2, Max 4.  
- Scaling Policy: Target Tracking → Average CPU Utilization 50%, Instance warm-up 60s.
- CloudWatch Metrics включены.

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
