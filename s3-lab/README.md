 # Лабораторная работа №4: Облачное хранилище данных Amazon S3

---

## Цель работы
Познакомиться с сервисом Amazon S3 и освоить основные операции:
- Создание публичного и приватного бакетов;
- Загрузка и организация объектов;
- Работа с AWS CLI (копирование, перемещение, синхронизация);
- Настройка версионирования и шифрования;
- Использование S3 Static Website Hosting;
- Применение Lifecycle-правил для автоматизации хранения данных.

---

## Шаг 1. Подготовка локальной структуры
Создана структура каталогов для работы с S3:
```
s3-lab/
├── public/
│ ├── avatars/
│ │ ├── user1.jpg
│ │ └── user2.jpg
│ └── content/
│ └── logo.png
├── private/
│ └── logs/
│ └── activity.csv
└── README.md
```

**Примечание:** В S3 "папки" — это префиксы ключей, а не реальные директории.  

**Скриншот локальной структуры:**  
<img width="634" height="757" alt="{91E083EF-33E7-400F-AEF4-49325AB65691}" src="https://github.com/user-attachments/assets/8c74c934-fa0e-4b63-ab2b-f7a2988807b2" />

**Вопросы и ответы:**

- **Что такое S3?**  
  S3 — объектное хранилище, где файлы (объекты) хранятся в бакетах с уникальными ключами.  

- **В чём отличие объекта от обычного файла?**  
  В S3 объект имеет ключ, метаданные и данные; "папки" — это просто префиксы ключей.

---

## Шаг 2. Создание бакетов

### 2.1 Публичный бакет

Создан бакет `cc-lab4-pub-01` в регионе `eu-central-1`.  

**Настройки:**
- ACLs enabled;
- Block all public access — снят.

**Скриншот консоли AWS:**  
<img width="1554" height="514" alt="{20A5857A-1A84-4CBB-A4DD-BE268CA2ED84}" src="https://github.com/user-attachments/assets/f3e76847-c8b0-4115-9856-3baca65d5395" />

### 2.2 Приватный бакет

Создан бакет `cc-lab4-priv-01` в регионе `eu-central-1`.  

**Настройки:**
- Block all public access — включен.

**Скриншот консоли AWS:**  
<img width="1902" height="740" alt="{C4636193-0E59-48FD-BEA4-E14CF2963CCA}" src="https://github.com/user-attachments/assets/0cd21c13-3cd8-40e6-9d4b-e9d733dcce6e" />

**Вопросы и ответы:**

- **Что означает опция “Block all public access”?**  
  Она запрещает делать объекты публичными, предотвращая случайный открытый доступ.  

- **Чем отличается ACLs enabled от Object Ownership Enforced?**  
  - ACLs: управление доступом к объектам на уровне ACL.  
  - Object Ownership Enforced: управление доступом централизованное через политики, ACL не используются.

---

## Шаг 3. Загрузка объектов через AWS CLI

### 3.1 Публичный бакет

# Загрузка user1.jpg
```
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user1.jpg" s3://cc-lab4-pub-01/avatars/user1.jpg
```
<img width="1658" height="933" alt="{2AE3FA28-43D3-42CB-9753-C9E036DC0501}" src="https://github.com/user-attachments/assets/69297ae9-d0c8-49c5-b038-1cde0f6c11b4" />

# Загрузка user2.jpg
```
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user2.jpg" s3://cc-lab4-pub-01/avatars/user2.jpg
```
<img width="1574" height="693" alt="{E75E9D7C-0C74-4795-897C-6C865CC826C9}" src="https://github.com/user-attachments/assets/3c2b729e-e1f3-4abe-8aa5-d3b256add2ed" />
# Загрузка logo.png
```
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\content\logo.png" s3://cc-lab4-pub-01/content/logo.png
```
<img width="1599" height="665" alt="{EC2750A3-0A72-4DBB-999C-6A3FC29870F2}" src="https://github.com/user-attachments/assets/f137b88b-6887-4a71-928d-914b3e4bf6e6" />


# 3.2 Приватный бакет
# Загрузка лог-файла
```
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\private\logs\activity.csv" s3://cc-lab4-priv-01/logs/activity.csv
```
Скриншот загрузки:

Вопросы и ответы:
В чём разница между aws s3 cp, mv и sync?
cp — копирование файлов;
mv — перемещение (копирование + удаление исходника);
sync — синхронизация директорий, копирует только новые или изменённые файлы.

Что делает флаг --acl public-read?
Делает объект публичным, доступным для всех по URL.

# Шаг 4. Проверка доступа к объектам
Публичные объекты доступны по ссылкам:

user1.jpg

user2.jpg

logo.png

Приватный объект activity.csv недоступен публично.

Скриншот проверки:

Вопросы и ответы:

Чем отличается публичный объект от приватного?
Публичный доступен всем через URL, приватный — только владельцу или через IAM-политику.

# Шаг 5. Версионирование объектов

```
aws s3api put-bucket-versioning --bucket cc-lab4-pub-01 --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket cc-lab4-priv-01 --versioning-configuration Status=Enabled
```
Проверка:

```
aws s3api get-bucket-versioning --bucket cc-lab4-pub-01
aws s3api get-bucket-versioning --bucket cc-lab4-priv-01
```
Скриншот вкладки Versions:

Вопросы и ответы:

Что такое версионирование в S3?
Позволяет хранить все версии объекта, включая старые, чтобы можно было восстановить изменения.

Что произойдёт, если выключить версионирование?
Старые версии останутся, новые будут записываться без версий.

# Шаг 6. Lifecycle-правила

Файл lifecycle.json:

```
{
  "Rules": [
    {
      "ID": "logs-archive",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 1825
      }
    }
  ]
}
```
Применение:

```
aws s3api put-bucket-lifecycle-configuration --bucket cc-lab4-priv-01 --lifecycle-configuration file://C:\Users\jenia\Desktop\s3-lab\lifecycle.json
```
Скриншот Lifecycle правила:
<img width="641" height="673" alt="{9D6034D2-E557-4F26-9005-335116ED0EFC}" src="https://github.com/user-attachments/assets/487751ce-830a-470a-9c5c-1bdfbd5450a6" />

Вопросы и ответы:
```
Что такое Storage Class в S3?
Класс хранения определяет стоимость и скорость доступа: Standard, Standard-IA, Glacier, Deep Archive.

Зачем нужны Lifecycle-правила?
Для автоматизации перехода объектов в дешёвое или архивное хранение и удаления старых файлов.
```
# Шаг 7. Статический веб-сайт на S3
```
# Создание бакета для веб-сайта
aws s3 mb s3://cc-lab4-web-01 --region eu-central-1
```
# Настройка статического сайта
```
aws s3 website s3://cc-lab4-web-01/ --index-document index.html
```
# Загрузка файлов сайта
```
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\content\logo.png" s3://cc-lab4-web-01/content/logo.png
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user1.jpg" s3://cc-lab4-web-01/avatars/user1.jpg
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user2.jpg" s3://cc-lab4-web-01/avatars/user2.jpg
```
URL сайта: http://cc-lab4-web-01.s3-website.eu-central-1.amazonaws.com/
Скриншоты сайта:
<img width="1604" height="932" alt="{1C5F9983-7D19-4F5E-89B5-4BC1BE4F2DCC}" src="https://github.com/user-attachments/assets/cb1e7f9c-718c-45e0-81f7-8d45acadc740" />

# Вопросы и ответы:
```
Что такое S3 Static Website Hosting?
Позволяет размещать статические сайты (HTML, CSS, JS) без серверов.

Как сделать файлы публичными?
Через ACL (--acl public-read) или политику бакета.
```





