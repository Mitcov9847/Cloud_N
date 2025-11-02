 # Лабораторная работа №4: Облачное хранилище данных Amazon S3

**Студент:** Евгений Митков  
**Дата выполнения:** 02.11.2025  

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

markdown
Копировать код

**Примечание:** В S3 "папки" — это префиксы ключей, а не реальные директории.  

**Скриншот локальной структуры:**  
![Структура папок](./screenshots/step1_structure.png)

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
![Публичный бакет](./screenshots/step2_pub_bucket.png)

### 2.2 Приватный бакет

Создан бакет `cc-lab4-priv-01` в регионе `eu-central-1`.  

**Настройки:**
- Block all public access — включен.

**Скриншот консоли AWS:**  
![Приватный бакет](./screenshots/step2_priv_bucket.png)

**Вопросы и ответы:**

- **Что означает опция “Block all public access”?**  
  Она запрещает делать объекты публичными, предотвращая случайный открытый доступ.  

- **Чем отличается ACLs enabled от Object Ownership Enforced?**  
  - ACLs: управление доступом к объектам на уровне ACL.  
  - Object Ownership Enforced: управление доступом централизованное через политики, ACL не используются.

---

## Шаг 3. Загрузка объектов через AWS CLI

### 3.1 Публичный бакет

```powershell
# Загрузка user1.jpg
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user1.jpg" s3://cc-lab4-pub-01/avatars/user1.jpg

# Загрузка user2.jpg
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user2.jpg" s3://cc-lab4-pub-01/avatars/user2.jpg

# Загрузка logo.png
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\content\logo.png" s3://cc-lab4-pub-01/content/logo.png
Скриншот успешной загрузки:

3.2 Приватный бакет
powershell
Копировать код
# Загрузка лог-файла
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\private\logs\activity.csv" s3://cc-lab4-priv-01/logs/activity.csv
Скриншот загрузки:

Вопросы и ответы:

В чём разница между aws s3 cp, mv и sync?

cp — копирование файлов;

mv — перемещение (копирование + удаление исходника);

sync — синхронизация директорий, копирует только новые или изменённые файлы.

Что делает флаг --acl public-read?
Делает объект публичным, доступным для всех по URL.

Шаг 4. Проверка доступа к объектам
Публичные объекты доступны по ссылкам:

user1.jpg

user2.jpg

logo.png

Приватный объект activity.csv недоступен публично.

Скриншот проверки:

Вопросы и ответы:

Чем отличается публичный объект от приватного?
Публичный доступен всем через URL, приватный — только владельцу или через IAM-политику.

Шаг 5. Версионирование объектов
powershell
Копировать код
aws s3api put-bucket-versioning --bucket cc-lab4-pub-01 --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket cc-lab4-priv-01 --versioning-configuration Status=Enabled
Проверка:

powershell
Копировать код
aws s3api get-bucket-versioning --bucket cc-lab4-pub-01
aws s3api get-bucket-versioning --bucket cc-lab4-priv-01
Скриншот вкладки Versions:

Вопросы и ответы:

Что такое версионирование в S3?
Позволяет хранить все версии объекта, включая старые, чтобы можно было восстановить изменения.

Что произойдёт, если выключить версионирование?
Старые версии останутся, новые будут записываться без версий.

Шаг 6. Lifecycle-правила
Файл lifecycle.json:

json
Копировать код
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
Применение:

powershell
Копировать код
aws s3api put-bucket-lifecycle-configuration --bucket cc-lab4-priv-01 --lifecycle-configuration file://C:\Users\jenia\Desktop\s3-lab\lifecycle.json
Скриншот Lifecycle правила:

Вопросы и ответы:

Что такое Storage Class в S3?
Класс хранения определяет стоимость и скорость доступа: Standard, Standard-IA, Glacier, Deep Archive.

Зачем нужны Lifecycle-правила?
Для автоматизации перехода объектов в дешёвое или архивное хранение и удаления старых файлов.

Шаг 7. Статический веб-сайт на S3
powershell
Копировать код
# Создание бакета для веб-сайта
aws s3 mb s3://cc-lab4-web-01 --region eu-central-1

# Настройка статического сайта
aws s3 website s3://cc-lab4-web-01/ --index-document index.html

# Загрузка файлов сайта
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\content\logo.png" s3://cc-lab4-web-01/content/logo.png
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user1.jpg" s3://cc-lab4-web-01/avatars/user1.jpg
aws s3 cp "C:\Users\jenia\Desktop\s3-lab\public\avatars\user2.jpg" s3://cc-lab4-web-01/avatars/user2.jpg
URL сайта: http://cc-lab4-web-01.s3-website.eu-central-1.amazonaws.com/

Скриншоты сайта:


Вопросы и ответы:

Что такое S3 Static Website Hosting?
Позволяет размещать статические сайты (HTML, CSS, JS) без серверов.

Как сделать файлы публичными?
Через ACL (--acl public-read) или политику бакета.
