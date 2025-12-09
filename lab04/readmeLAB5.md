# Лабораторная работа №5. Облачные базы данных. Amazon RDS

# Введение  
В данной лабораторной работе мы изучаем облачные сервисы Amazon Web Services (AWS), предназначенные для работы с реляционными и нереляционными базами данных. Основная цель — научиться развертывать Amazon RDS, создавать Read Replicas, подключаться к базе данных через EC2 и выполнять CRUD‑операции.

## Цель работы
Цель данной лабораторной работы — изучить сервис Amazon RDS и освоить следующие навыки:
Развертывать и конфигурировать экземпляры реляционных баз данных в среде AWS.
Разбираться в механизме Read Replicas и понимать, как они повышают производительность и устойчивость системы.
Подключаться к Amazon RDS с виртуального сервера EC2 и выполнять основные CRUD-операции с данными.

---

## Шаг 1. Подготовка инфраструктуры (VPC, Subnets, Security Groups)

### Действия:

1. Создал VPC `project-vpc` с IPv4 CIDR `10.0.0.0/16`.
2. Создал две публичные и две приватные подсети:
   - `public-a` (10.0.1.0/24) – зона доступности eu-central-1a  
   - `private-a` (10.0.11.0/24) – зона доступности eu-central-1a  
   - `public-b` (10.0.2.0/24) – зона доступности eu-central-1b  
   - `private-b` (10.0.12.0/24) – зона доступности eu-central-1b  
3. Создал Security Group `web-security-group` для приложения:
   - Входящий: HTTP (порт 80) от любого источника  
   - Входящий: SSH (порт 22) от моего IP  
4. Создал Security Group `db-mysql-security-group` для базы данных:
   - Входящий: MySQL (порт 3306) от `web-security-group`  
5. Добавил правило исходящего трафика в `web-security-group` к `db-mysql-security-group` для порта 3306.

**Контрольный вопрос:**  
*Почему необходимо создавать отдельные группы безопасности для приложения и базы данных?*  
**Ответ:** Это обеспечивает изоляцию и безопасность: приложение имеет доступ к базе данных только через определенные правила, а база данных не открыта в интернет, что снижает риск несанкционированного доступа.

<img width="1064" height="491" alt="{C377FD87-F079-428A-ABD8-0C18C1D59144}" src="https://github.com/user-attachments/assets/7b6a0b7a-5186-4ed9-91ae-d151b9c7c5e1" />
<img width="1830" height="205" alt="{083A3C43-A74E-4A37-B7EE-44B3DC155A32}" src="https://github.com/user-attachments/assets/edbfb12d-c2f1-4335-a0db-fdb1c3585c73" />

---

## Шаг 2. Развертывание Amazon RDS

### Действия:

1. Создал Subnet Group `project-rds-subnet-group` и добавил две приватные подсети.
2. Создал базу данных MySQL 8.0.42 через Standard Create:
- **Creation method:** Standard Create  
- **Engine:** MySQL  
- **Version:** 8.0.42  
- **Template:** Free Tier  
- **DB instance identifier:** project-rds-mysql-prod  
- **Master username:** admin  
- **Class:** db.t3.micro  
- **Storage:** gp3, 20 GB, autoscaling до 100 GB  
- **Public access:** No  
- **VPC:** project-vpc  
- **DB subnet group:** project-rds-subnet-group  
- **Security group:** db-mysql-security-group  
- **Initial DB:** project_db  

После создания копируем **Endpoint**.

**Контрольный вопрос:**  
*Что такое Subnet Group и зачем она нужна?*  
**Ответ:** Subnet Group – это группа подсетей в VPC, где размещаются экземпляры RDS. Она необходима, чтобы база данных могла корректно работать в разных зонах доступности для отказоустойчивости.

<img width="1222" height="555" alt="{4C15F101-8340-4396-AD7D-0DA923E755D4}" src="https://github.com/user-attachments/assets/f195a39a-7a38-4e08-a9bc-0e48813a46fb" />
<img width="1400" height="681" alt="{3A31D5A5-BCB7-4F98-BBAF-8EB2AE7DACC6}" src="https://github.com/user-attachments/assets/0d2da6ef-7d00-4008-b218-a2f8cf1c234c" />
<img width="1637" height="572" alt="{56E2AA8C-100A-49BB-8502-F48D989FD1B5}" src="https://github.com/user-attachments/assets/ca709081-1c97-4524-8396-8b7fe1049abb" />
<img width="1088" height="354" alt="{7CCEB4D9-C3E4-4243-958E-4B2BBB14F4AB}" src="https://github.com/user-attachments/assets/c866deaf-d7f6-4a92-bfcd-f0fae957ec2d" />
<img width="1122" height="122" alt="{F555A073-3E38-4CB6-B815-0A5CE89BF4EE}" src="https://github.com/user-attachments/assets/0af13f58-05c8-475e-89f5-3a11f18b34d1" />

---

## Шаг 3. Создание виртуальной машины EC2

### Действия:

1. Создал EC2 в публичной подсети VPC.
2. Назначил Security Group `web-security-group`.
3. Установил MySQL клиент на EC2:

## Устанавливаем MySQL-клиент:
```bash
#!/bin/bash
dnf update -y
dnf install -y mariadb105
```

**Контрольный вопрос:**  
*Зачем нужен MySQL клиент на EC2?*  
**Ответ:** Для подключения к RDS и выполнения SQL-запросов напрямую с виртуальной машины без использования внешних инструментов.

<img width="631" height="144" alt="{9932534D-32C7-41F6-B847-FA1025790F0F}" src="https://github.com/user-attachments/assets/53d5c6ca-7249-4f5c-a3b1-366d9516b27a" />
<img width="435" height="253" alt="{D6456525-546C-4F61-AC39-99A867C00912}" src="https://github.com/user-attachments/assets/3317a051-c5bd-4120-af40-ee7351b9ab1c" />

---

## Шаг 4. Подключение к базе данных и выполнение базовых операций

### Действия:

1. Подключился к EC2 по SSH.
2. Подключился к базе данных RDS:

<img width="697" height="202" alt="{8341736B-18FF-441E-B5EC-E2C2379402F8}" src="https://github.com/user-attachments/assets/cd6d13ae-8ce3-4b69-86a1-124cd8f10f02" />


3. Выбрал базу данных:

```sql
USE project_db;
```

4. Создал таблицы `categories` и `todos` с отношением один-ко-многим:

```sql
CREATE TABLE categories (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL
);

CREATE TABLE todos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255),
  status VARCHAR(50),
  category_id INT,
  FOREIGN KEY (category_id) REFERENCES categories(id)
);
```
<img width="655" height="498" alt="{C699D0ED-6386-41F8-8027-D5C20D74F097}" src="https://github.com/user-attachments/assets/3f7db8b0-586d-4611-b4cb-6e94853f61e8" />

5. Вставил данные (по 3 записи в каждую таблицу) и выполнил JOIN-запросы.

**Контрольный вопрос:**  
*Зачем создается отношение один-ко-многим между таблицами?*  
**Ответ:** Чтобы структурировать данные, позволяя одной категории иметь несколько связанных задач, что облегчает анализ и выборку данных.

---

## Шаг 5. Создание Read Replica

### Действия:

1. В консоли AWS выбрал базу `project-rds-mysql-prod` и создал Read Replica:
- Identifier: project-rds-mysql-read-replica
- Class: db.t3.micro
- Public access: No
- Security group: db-mysql-security-group
После запуска — копируем endpoint реплики.

2. Дождался статуса `Available`.
3. Подключился к Read Replica и выполнил SELECT-запросы для проверки данных.

**Контрольный вопрос:**  
*Что происходит при выполнении запроса на запись на Read Replica?*  
**Ответ:** Запись не выполняется, так как Read Replica только для чтения; она синхронизируется с основной базой.

4. Добавил новую запись в основной базе и проверил, что она появилась на Read Replica.

**Контрольный вопрос:**  
*Зачем нужны Read Replicas?*  
**Ответ:** Для разгрузки основной базы, повышения производительности запросов на чтение и обеспечения отказоустойчивости.
- масштабирования нагрузки на чтение;
- аналитики;
- резервного копирования;
  
---
# Подключение приложения к RDS (вариант на выбор)

## Создание простого CRUD‑приложения
Приложение должно:
- писать/обновлять в master;
- читать из read replica.

##  Использование PHP‑проекта из ЛР №4
Изменяем параметры подключения:

```php
$host = "project-rds-mysql-prod.xxxxxx.rds.amazonaws.com";
$user = "admin";
$pass = "********";
$db   = "project_db";
```
После этого приложение полностью работает на Amazon RDS.

## Шаг 6. Подключение приложения к базе данных

### Действия:

1. Развернул простое веб-приложение на EC2.
2. Настроил подключение к Master для операций записи и Read Replica для операций чтения.
3. Проверил CRUD-операции: создание, чтение, обновление и удаление записей.

<img width="570" height="282" alt="{D6EACDAA-E674-4564-A623-DDC094EDFB4C}" src="https://github.com/user-attachments/assets/fa38e906-1bd3-418f-8257-4fb7bac22905" />

**Контрольный вопрос:**  
*Почему нужно разделять операции чтения и записи между Master и Read Replica?*  
**Ответ:** Чтобы уменьшить нагрузку на основную базу и ускорить обработку запросов на чтение.


## Настройка единого подключения к Amazon RDS

В каталоге `src` создаётся файл `db.php`, который инкапсулирует подключение к базе данных Amazon RDS через PDO.

```php
<?php
$host     = "project-rds-mysql-prod.c7a2w4yiq4ee.eu-central-1.rds.amazonaws.com";
$dbname   = "project_db";
$username = "admin";
$password = "";

try {
    $pdo = new PDO(
        "mysql:host=$host;dbname=$dbname;charset=utf8",
        $username,
        $password,
        [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        ]
    );
} catch (PDOException $e) {
    die("Database connection failed: " . $e->getMessage());
}
```

<img width="955" height="110" alt="{C5C6F1C6-DE5C-4DD8-B171-EFEA3F6DAA69}" src="https://github.com/user-attachments/assets/74a2de33-e688-44ba-bb68-2ada0227459c" />
```
// На главной странице
require_once __DIR__ . '/../src/db.php';
```
<img width="337" height="42" alt="{F1197634-0025-4EBD-8A01-FAF9E82FBE81}" src="https://github.com/user-attachments/assets/e25e10a2-9e4b-4de6-a756-54472a7a8858" />
```
// На странице со всеми рецептами
require_once __DIR__ . '/../../src/db.php';
```
<img width="374" height="38" alt="{6EB7480F-D7AB-434F-B454-74FEC727FB7B}" src="https://github.com/user-attachments/assets/b7fe83ac-595c-43be-89fc-2b06e8d09826" />
```
// В обработчиках формы
require_once __DIR__ . '/../../../src/db.php';
```
<img width="355" height="41" alt="{A23493CD-2CEA-4198-8DC6-D9F4E083CB4E}" src="https://github.com/user-attachments/assets/ca2b3a2e-7295-46c1-92ab-cc1d980e32be" />
Такое решение упрощает и гарантирует приложение использует одно и то же подключение к Amazon RDS.

---

## Реализация операции Create: сохранение нового рецепта в Amazon RDS

Обработчик public/handlers/save_recipe.php выполняет сохранение рецепта, проходя следующие этапы:
Инициализация сессии и подключение вспомогательных функций вместе с файлом db.php.
Получение данных из формы и их очистка (получаются title, category, ingredients, description и массив steps).
Проверка корректности заполнения полей; при наличии ошибок данные и сообщения об ошибках сохраняются в $_SESSION, после чего происходит возврат пользователя на форму.

Далее выполняется транзакция PDO с двумя запросами — в таблицу рецептов и в таблицу шагов:
```
<?php
try {
    // Начало транзакции
    $pdo->beginTransaction();

    // Сохраняем основной рецепт
    $stmt = $pdo->prepare("
        INSERT INTO recipes (title, category, description)
        VALUES (:title, :category, :description)
    ");
    $stmt->execute([
        ':title'       => $title,
        ':category'    => $category,
        ':description' => $fullDescription,
    ]);

    // Получаем ID только что созданного рецепта
    $recipeId = $pdo->lastInsertId();

    // Кодируем шаги в JSON
    $stepsJson = json_encode($steps, JSON_UNESCAPED_UNICODE);

    if ($stepsJson === false) {
        // Проверка на ошибки кодирования JSON
        throw new Exception('Ошибка кодирования шагов рецепта в JSON: ' . json_last_error_msg());
    }

    // Сохраняем шаги приготовления
    $stmt2 = $pdo->prepare("
        INSERT INTO recipe_steps (recipe_id, steps_json)
        VALUES (:recipe_id, :steps_json)
    ");
    $stmt2->execute([
        ':recipe_id' => $recipeId,
        ':steps_json'=> $stepsJson,
    ]);

    // Фиксируем транзакцию
    $pdo->commit();

    $_SESSION['success'] = 'Рецепт успешно сохранён в Amazon RDS.';
    header('Location: ../index.php');
    exit();

} catch (Exception $e) {
    // Откат транзакции при любой ошибке
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }

    $_SESSION['errors'] = [
        'db' => 'Ошибка сохранения: ' . $e->getMessage(),
    ];
    $_SESSION['old'] = $_POST;

    header('Location: ../recipe/create.php');
    exit();
}

```
Если обработчик вызывается не POST-запросом, выполняется безопасный возврат на главную страницу.
Таким образом реализована операция Create для базы Amazon RDS.

---

# Заключение

В ходе работы выполнено:
- создание изолированной облачной инфраструктуры AWS;
- развертывание Amazon RDS MySQL в приватных подсетях;
- подключение через EC2;
- создание таблиц и выполнение SQL‑операций;
- создание Read Replica и проверка работы репликации;
- подключение приложения к AWS RDS.
- 
## Вывод

В ходе лабораторной работы я познакомился с Amazon RDS, создал базу данных MySQL, настроил безопасность через VPC и Security Groups, развернул EC2, подключился к базе, создал таблицы и наполнил их данными, создал Read Replica и проверил синхронизацию. Также настроил веб-приложение для работы с основной базой и репликой, выполнил CRUD-операции и убедился в правильности архитектуры.

## Библиография

1. **Amazon RDS Documentation** — официальная документация по работе с сервисом Amazon RDS, созданию и управлению экземплярами реляционных баз данных, настройке Subnet Groups, параметров безопасности и резервного копирования.  
   [https://docs.aws.amazon.com/rds/](https://docs.aws.amazon.com/rds/)

2. **Amazon EC2 Documentation** — руководство по созданию и настройке виртуальных машин EC2, безопасности через Security Groups, подключению к базам данных и работе с клиентами MySQL.  
   [https://docs.aws.amazon.com/ec2/](https://docs.aws.amazon.com/ec2/)

3. **Amazon VPC Documentation** — справочник по созданию виртуальных частных облаков (VPC), подсетей, маршрутов, Internet Gateway и настройке сетевой безопасности для приложений и баз данных.  
   [https://docs.aws.amazon.com/vpc/](https://docs.aws.amazon.com/vpc/)

4. **MySQL Documentation** — официальная документация по MySQL, включая создание таблиц, связи один-ко-многим, выполнение CRUD-запросов и работу с JOIN-операциями.  
   [https://dev.mysql.com/doc/](https://dev.mysql.com/doc/)

5. **Amazon Read Replicas Documentation** — руководство по созданию и использованию Read Replicas в Amazon RDS для повышения производительности и отказоустойчивости баз данных.  
   [https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)



