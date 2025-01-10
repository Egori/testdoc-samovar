# Тестовое задание

- [Часть 1: Серверная часть на Go (Backend)](#часть-1-серверная-часть-на-go-backend)
  - [1. Разработка API](#1-разработка-api)
  - [2. Оптимизация производительности](#2-оптимизация-производительности)
- [Часть 2: Работа с PostgreSQL](#часть-2-работа-с-postgresql)
  - [3. Проектирование базы данных](#3-проектирование-базы-данных)
  - [4. Оптимизация запросов](#4-оптимизация-запросов)
- [Часть 3: Контейнеризация с Docker](#часть-3-контейнеризация-с-docker)
  - [5. Создание контейнера для приложения](#5-создание-контейнера-для-приложения)
  - [6. Автоматизация CI/CD](#6-автоматизация-cicd)
- [Часть 4: Frontend на React](#часть-4-frontend-на-react)
  - [7. Разработка UI](#7-разработка-ui)
  - [8. Оптимизация интерфейса](#8-оптимизация-интерфейса)
- [Часть 5: Интеграция с 1С](#часть-5-интеграция-с-1с)
  - [9. Интеграция с 1С через API](#9-интеграция-с-1с-через-api)
  - [10. Доработка конфигурации 1С](#10-доработка-конфигурации-1с)
- [Часть 6: Операционная система и инфраструктура](#часть-6-операционная-система-и-инфраструктура)
  - [11. Настройка Nginx](#11-настройка-nginx)
- [Часть 7: Оценка общей архитектуры](#часть-7-оценка-общей-архитектуры)
  - [13. Проектирование архитектуры](#13-проектирование-архитектуры)


## Часть 1: Серверная часть на Go (Backend)  
### 1. Разработка API    

Ссылка на репозиторий с приложением:  
https://github.com/Egori/userlist-api-go-test

Приложение доступно по адресу: http://193.43.79.184:8080/

Примеры запросов:  
```bash

 curl http://193.43.79.184:8080/api/users

 curl -X POST http://193.43.79.184:8080/users \    
-H "Content-Type: application/json" \
-d '{"name": Jack Smith, "email": "jack.sm@example.com"}'

 curl -X PUT http://193.43.79.184:8080/users/1 \    
-H "Content-Type: application/json" \
-d '{"name": User Updated, "email": "user.upd@example.com"}'

 curl -X DELETE http://193.43.79.184:8080/users/1

```

Приложение написано на Go с использованием фреймворка Echo.   
Использовано ORM Gorm, для миграции задействовано Gorm AutoMigrate.   
Написаны тесты для ендпоинтов.

### 2. Оптимизация производительности

 Оптимизация производительности API для работы с большим объемом данных требует всестороннего подхода, включающего оптимизацию базы данных, кода и инфраструктуры.
####  Оптимизация базы данных
##### Индексация
Необходимо создать индексы в базе данных на колонки, используемые в фильтрации и сортировке, такие как ID, name, Email. Например:
```sql
CREATE INDEX idx_name ON users (name);  
CREATE INDEX idx_email ON users (email);  
CREATE INDEX idx_id ON users (id);  
```
Если используется Gorm,как в нашем случае, индексы прописываются в структуре модели:
```go
type User struct {
	gorm.Model
	Name  string `json:"name" gorm:"size:100;not null;index"`
	Email string `json:"email" gorm:"size:255;not null;unique"`
}
```

При этом индексы будут созданы автоматически в базе данных при создании таблицы.
```go
    db.AutoMigrate(&User{})
```

##### Шардирование
При огромных объемах данных (>10 млн записей) можно рассмотреть горизонтальное шардирование, например, по возрасту или географическому региону.

#### Оптимизация приложения
##### Пагинация
Вместо возврата всех записей в ответе, потребуется реализовать пагинацию. Например:
```go
func (r *userRepository) GetPaginated(limit, offset int) ([]models.User, error) {
    var users []models.User
    err := r.db.Limit(limit).Offset(offset).Find(&users).Error
    return users, err
}
```
##### Кэширование
Кэширование часто запрашиваемых данных в Redis или Memcached поможет ускорить доступ к данным и уменьшить нагрузку на базу данных.

##### Batch Updates
выполнение операций с большим объемом данных производится партиями:
for _, user := range users {
    db.Model(&user).Updates(map[string]interface{}{"age": 30})
}

#####  Сжатие данных
Cжатие HTTP-ответов через middleware, например, gzip:
```go
import	"github.com/labstack/echo/v4/middleware"

	e := echo.New()

	// Добавляем middleware для сжатия
	e.Use(middleware.CompressWithConfig(middleware.CompressConfig{
		Level: middleware.DefaultCompressionLevel, // Уровень сжатия
	}))
  ```
  
  #### Инфраструктура
##### Вертикальное масштабирование
Обеспечить достаточное количество ресурсов ссервера (RAM, CPU).
##### Горизонтальное масштабирование и балансировка нагрузки баз данных.
Создать реплики базы данных для распределения нагрузки на чтение.
Балансировка нагрузки с репликами может быть реализована:  
 - В приложении (самостоятельное распределение запросов).
 - Через прокси (ProxySQL, Pgpool-II).
 - На уровне драйвера базы данных.
##### Балансировка нагрузки
Использовать Load Balancer (например, настроив Nginx) для распределения запросов между несколькими экземплярами API.

## Часть 2: Работа с PostgreSQL  
### 3. Проектирование базы данных

#### Таблица `users` (Пользователи)
Хранит информацию о пользователях.  

| Поле           | Тип данных          | Описание                                |
|----------------|---------------------|----------------------------------------|
| `id`          | SERIAL PRIMARY KEY  | Уникальный идентификатор пользователя  |
| `name`        | VARCHAR(255)        | Имя пользователя                       |
| `email`       | VARCHAR(255) UNIQUE | Email пользователя                     |
| `created_at`  | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | Дата регистрации |

#### Таблица `products` (Товары)
Хранит информацию о товарах.  

| Поле           | Тип данных          | Описание                                |
|----------------|---------------------|----------------------------------------|
| `id`          | SERIAL PRIMARY KEY  | Уникальный идентификатор товара        |
| `name`        | VARCHAR(255)        | Название товара                        |
| `price`       | DECIMAL(10, 2)      | Цена товара                            |
| `stock`       | INTEGER             | Количество на складе                   |
| `created_at`  | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | Дата добавления товара |

#### Таблица `orders` (Заказы)
Хранит информацию о заказах пользователей.  

| Поле           | Тип данных          | Описание                                |
|----------------|---------------------|----------------------------------------|
| `id`          | SERIAL PRIMARY KEY  | Уникальный идентификатор заказа        |
| `user_id`     | INTEGER             | Внешний ключ на `users.id`             |
| `total`       | DECIMAL(10, 2)      | Общая сумма заказа                     |
| `created_at`  | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | Дата создания заказа |
| FOREIGN KEY   |                     | (`user_id`) REFERENCES `users`(`id`) ON DELETE CASCADE |

#### Таблица `order_items` (Элементы заказа)
Связывает заказы с товарами.  

| Поле           | Тип данных          | Описание                                |
|----------------|---------------------|----------------------------------------|
| `id`          | SERIAL PRIMARY KEY  | Уникальный идентификатор записи        |
| `order_id`    | INTEGER             | Внешний ключ на `orders.id`            |
| `product_id`  | INTEGER             | Внешний ключ на `products.id`          |
| `quantity`    | INTEGER             | Количество товара                      |
| `price`       | DECIMAL(10, 2)      | Цена за единицу товара на момент заказа|
| FOREIGN KEY   |                     | (`order_id`) REFERENCES `orders`(`id`) ON DELETE CASCADE |
| FOREIGN KEY   |                     | (`product_id`) REFERENCES `products`(`id`) |

---

### Обеспечение целостности данных

1. **Первичные ключи (`PRIMARY KEY`)**  
   Гарантируют уникальность записей в каждой таблице.

2. **Внешние ключи (`FOREIGN KEY`)**  
   - В таблице `orders` поле `user_id` связано с таблицей `users`.  
   - В таблице `order_items` поля `order_id` и `product_id` связаны с таблицами `orders` и `products` соответственно.  
   - Механизм `ON DELETE CASCADE` удаляет связанные записи автоматически, если пользователь или заказ удаляются.

3. **Ограничения уникальности (`UNIQUE`)**  
   Поле `email` в таблице `users` уникально, чтобы избежать дублирования.

4. **Проверка (`CHECK`)**  
   - Проверки на положительное количество товаров и цен:  
     ```sql
     CHECK (price >= 0), CHECK (quantity > 0), CHECK (stock >= 0)
     ```

5. **Индексы**  
   - Индексы на поля, которые часто используются в фильтрах или JOIN (`user_id`, `product_id`, `product_name`, `order_id`), для ускорения запросов.  

   Пример:
   ```sql
   CREATE INDEX idx_orders_user_id ON orders (user_id);
   CREATE INDEX idx_order_items_product_id ON order_items (product_id);
   ```

6. **Нормализация**  
   - Таблицы спроектированы согласно нормальным формам. Данные о пользователях, товарах и заказах разделены для предотвращения избыточности.


### 4. Оптимизация запросов    
Для оптимизации производительности запроса можно использовать предварительное агрегирование данных в подзапросе.

```sql
WITH max_order_dates AS (
    SELECT 
        user_id,
        MAX(created_at) AS max_date
    FROM orders
    GROUP BY user_id
)
SELECT 
    u.id AS user_id,
    u.name AS user_name,
    u.email,
    o.id AS order_id,
    o.total AS order_total,
    o.created_at AS order_date
FROM users u
LEFT JOIN max_order_dates md
    ON u.id = md.user_id
LEFT JOIN orders o
    ON md.user_id = o.user_id
    AND md.max_date = o.created_at
ORDER BY u.id;
```

В подзапросе заранее агрегируются максимальные даты заказов для каждого пользователя, что уменьшает количество вычислений при выполнении основного запроса. Это ускоряет `LEFT JOIN`, так как фильтрация происходит на меньшем объеме данных.

 **Индексация**  
   - На таблице `orders`:
     ```sql
     CREATE INDEX idx_orders_user_id_created_at ON orders (user_id, created_at DESC);
     ```
     Этот индекс ускоряет как группировку, так и фильтрацию по максимальной дате.
   - На таблице `users`:
     ```sql
     CREATE INDEX idx_users_id ON users (id);
     ```
     Обычно создается автоматически как PRIMARY KEY.

---


## Часть 3: Контейнеризация с Docker  
### 5. Создание контейнера для приложения   
  
Для того, чтобы использовать минимальные зависимости, можно использовать базовый образ alpine. 
При этом нужно собрать приложение вне контейнера.

   **Dockerfile**
   ```Dockerfile
    
      FROM alpine:latest

      # Устанавливаем сертификаты
      RUN apk add --no-cache ca-certificates

      # Устанавливаем рабочую директорию
      WORKDIR /app

      # Копируем скомпилированный бинарный файл
      COPY userlist-api .

      RUN chmod +x userlist-api

      # Копируем переменные окружения
      COPY .env .env

      # Открываем порт
      EXPOSE 8080

      # Запуск приложения
      CMD ["/app/userlist-api"]

```

Файл `docker-compose.yml` для работы с базой данных:
```yaml
version: "3.9"
services:
  postgres:
    image: postgres:17
    container_name: userlist_test_pg
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5433:${POSTGRES_PORT}"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  app:
    env_file:
      - .env
    build:
      context: .
      dockerfile: Dockerfile
    container_name: userlist_test_app
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=${POSTGRES_PORT}
    ports:
      - "${HTTP_PORT}:${HTTP_PORT}"
    depends_on:
      - postgres

networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: "172.28.0.0/16"

volumes:
  postgres_data:

```

Создаются два контейнера, приложение и база данных. Приложение зависит от базы данных, поэтому оно запускается после базы данных.

### 6. Автоматизация CI/CD    
Конфигурация для CI/CD GitHub Actions создана в репозитории, ```.github/workflows/ci-cd-test.yml```.
Выполняются тесты, линт, сборка и деплой приложения.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.23.2

      - name: Install dependencies
        run: go mod tidy

      - name: Run golangci-lint
        run: |
          curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v1.54.2
          ./bin/golangci-lint run
  
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.23.2

      - name: Install dependencies
        run: go mod tidy

      - name: Run tests
        run: go test ./... -v

  build-and-deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.23.2

      - name: Install dependencies
        run: go mod tidy
      
      - name: Build the application
        run:  CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o userlist-api ./cmd/app

      - name: Add SSH key to known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.TEST_SERVER }} >> ~/.ssh/known_hosts
          chmod 600 ~/.ssh/known_hosts

      - name: Create directory on the server
        env:
          TEST_SERVER: ${{ secrets.TEST_SERVER }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ssh_key
          chmod 600 ssh_key
          # Создаём директорию на сервере
          ssh -i ssh_key test@${{ secrets.TEST_SERVER }} "mkdir -p /home/test/userlist-api"

      - name: Copy files to the server
        env:
          TEST_SERVER: ${{ secrets.TEST_SERVER }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ssh_key
          chmod 600 ssh_key
          cd $GITHUB_WORKSPACE  # Переход в корневую директорию репозитория
          # Копируем файлы на сервер
          scp -i ssh_key userlist-api test@${{ secrets.TEST_SERVER }}:/home/test/userlist-api
          scp -i ssh_key .env test@${{ secrets.TEST_SERVER }}:/home/test/userlist-api
          scp -i ssh_key Dockerfile test@${{ secrets.TEST_SERVER }}:/home/test/userlist-api
          scp -i ssh_key docker-compose.yml test@${{ secrets.TEST_SERVER }}:/home/test/userlist-api

      - name: Deploy application
        env:
          TEST_SERVER: ${{ secrets.TEST_SERVER }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          ssh -i ssh_key test@${{ secrets.TEST_SERVER }} << 'EOF'
            cd /home/test/userlist-api
            docker-compose down || true
            docker-compose up -d --build
          EOF

```

Приложение успешно собирается и деплоится на сервер:  
![Screen_1](img/Screen_1.jpg)  
![Screen_2](img/Screen_2.jpg)  
![Screen_5](img/Screen_5.jpg)
![Screen_3](img/Screen_3.jpg)  

---

## Часть 4: Frontend на React  
### 7. Разработка UI    
   Создано приложение на React, которое отображает список пользователей из API, получая данные через AJAX-запросы. 
   https://github.com/Egori/userlist-react
   Реализована пагинация и фильтрация по имени пользователя.  
   Приложение доступно по адресу: http://193.43.79.184:3000/

### 8. Оптимизация интерфейса
8. Оптимизация интерфейса    

Для работы с наборами данных более 100 000 пользователей:  
 **Виртуализация**  
   В проекте используется `react-window` для рендеринга только видимых элементов, что снижает нагрузку на DOM и память.  
 **Lazy loading или пагинация**  
   Рекомендуется реализовать Lazy loading или пагинацию при получении данных с API, чтобы уменьшить время начальной загрузки.  
 **Дебаунсинг фильтрации**  
   Для повышения производительности поиска можно использовать задержку (debounce) обработки ввода пользователя, чтобы избежать частых вызовов API или тяжелых операций.

## Часть 5: Интеграция с 1С  
## 9. Интеграция с 1С через API    

 **Для выгрузки данных из 1С в Go**

Можно реализовать обмен по REST API.

На стороне go-приложения создать необходимые ендпоинты, которые будут принимать запросы от 1С в виде JSON.

На стороне 1С создать обработку, которая будет отправлять данные в Go-приложение.

   **Пример кода**  
   Простой код отправки данных выглядит так:
   ```1C

//здесь выполняем запрос, в котором есть данные о нашых товарах (код товара (sku),
//Наименование товара(name), цена(price) и остаток(qty))
Результат = Запрос.Выполнить();

    ВыборкаДетальныеЗаписи = Результат.Выбрать();

    //Создали запись ЗаписьJSON
    ЗаписьJSON = Новый ЗаписьJSON;
    //Задаем параметры без переноса строк, можно и с переносом, как кому нравится
    тПараметрыJSON = Новый ПараметрыЗаписиJSON(ПереносСтрокJSON.Нет, " ", Истина);
    ЗаписьJSON.УстановитьСтроку(тПараметрыJSON);


    МассивДанныхJSON = Новый Массив;
    СтруктураДанныхJSON = Новый Структура;

    //Выбираем данные из запроса и записываем в массив "МассивДанныхJSON"

    Пока ВыборкаДетальныеЗаписи.Следующий() Цикл
       // Каждая запись товара у нас отдельная структура...
              тДанные = Новый Структура;
                тДанные.Вставить("sku", ВыборкаДетальныеЗаписи.sku);
                тДанные.Вставить("name", ВыборкаДетальныеЗаписи.name);
                тДанные.Вставить("price", ВыборкаДетальныеЗаписи.price);
                тДанные.Вставить("qty", ВыборкаДетальныеЗаписи.qty);
       //Добавляем структуру с информацией о товаре в наш массив "МассивДанныхJSON"
               МассивДанныхJSON.Добавить(тДанные);

    КонецЦикла;
    // вставляем наш массив в ещеодну структуру
    СтруктураДанныхJSON.Вставить("test", МассивДанныхJSON);

    ЗаписатьJSON(ЗаписьJSON, СтруктураДанныхJSON);
    //Здесь нам платформа переделала нашу сложную структуру в строку данных в формате JSON
    СтрокаJS = ЗаписьJSON.Закрыть();
    //В этот файл для примера наш сайт сформирует ответ после отправки на него данных методом POST
    ФайлОтвета = КаталогВременныхФайлов()+ "\answer.txt";

    //здесь надо указать путь к сайту
    HTTPСоединение = Новый HTTPСоединение("mysite.com/products");
    //создаем запрос данных методом POST
    запросPOST = Новый HTTPЗапрос("POST");

//это обязательный заголовок тела запроса
    запросPOST.Заголовки.Вставить("Content-type", "application/x-www-form-urlencoded");
//Здесь задаем текст нашей отформатированной строки + задаем формат сроки
    запросPOST.УстановитьТелоИзСтроки("mData="+СтрокаJS,"windows-1251",ИспользованиеByteOrderMark.НеИспользовать);

//Отправляем запрос для обработки на сервер
    Попытка
        HTTPСоединение.ОтправитьДляОбработки(запросPOST, ФайлОтвета);

    Исключение
        #Если клиент Тогда
           Сообщить(ОписаниеОшибки());
        #КонецЕсли
    КонецПопытки;

   ``` 

   **Для отправки данных из Go-приложения на 1C:**
   Если 1С расположена на сервере, то на стороне 1С создаются HTTP-сервисы,которые будут принимать запросы от Go-приложения.
При этом нужно на сервере с 1С настроить веб-сервер Apache.
   Если 1С расположена на локальном компьютере, то можно насторить регламентное задание на получение свежих данных. 
Например, для получения данных о новых заказах, можно создать триггер в 1С, который будет выполняться каждые 5 минут.
 При этом на стороне Go-API-сервера нужно реализовать фильтрацию данных по дате, а из 1с получать только свежие данные, используя этот фильтр.

 Стандартный механизм обмена в 1С - создание так называемого web-сервиса, в котором реализован обмен через протокол SOAP. Данные отправляются и принимаются в формате WSDL. В этом случае на стороне 1С настройка производится без написания кода, но на стороне Go-приложения возможны сложности с разбором этого формата.

### 10. Доработка конфигурации 1С    
 Доработка конфигурации 1С на прием запросов заключается в создании HTTP-сервисов, которые будут принимать запросы от Go-приложения. Код по обработке запросов будет находиться в модулях этих сервисах и в общем модуле.
 
 Для отдачи данных возможно понадобится создать регламентное задание в 1С, других изменеий конфигурации не потребуется, - всю логику работы можно вынести во внешнюю обработку.

 ## Часть 6: Операционная система и инфраструктура  
### 11. Настройка Nginx    

Настройка **Nginx** для работы с высоконагруженным Go-приложением включает несколько аспектов: балансировку нагрузки, обеспечение отказоустойчивости и кэширование. Рассмотрим все эти аспекты.

---

#### **Базовая конфигурация Nginx для Go-приложения**

##### Настройка обратного прокси
Nginx будет принимать HTTP-запросы и проксировать их на приложение Go. 

Пример конфигурации:
```nginx
http {
    upstream go_app {
        server 127.0.0.1:8080; # Приложение Go работает на порту 8080
        server 127.0.0.1:8081; # Вторая копия приложения
    }

    server {
        listen 80;
        server_name yourdomain.com;

        location / {
            proxy_pass http://go_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

---

#### **Обеспечение отказоустойчивости**

##### Проверка состояния бэкендов (health check)
Nginx поддерживает проверки состояния серверов для исключения недоступных экземпляров:
```nginx
http {
    upstream go_app {
        server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    }
}
```

- `max_fails=3`: Если сервер не отвечает три раза подряд, он исключается.
- `fail_timeout=30s`: Сервер будет исключён на 30 секунд.


#### **Балансировка нагрузки**

##### Алгоритмы балансировки
По умолчанию используется **Round Robin**, но можно настроить другие методы:
- **Least Connections**: Направляет запросы серверу с наименьшим количеством активных подключений:
  ```nginx
  upstream go_app {
      least_conn;
      server 127.0.0.1:8080;
      server 127.0.0.1:8081;
  }
  ```

- **Weight-based**: Серверы с большим весом получают больше запросов:
  ```nginx
  upstream go_app {
      server 127.0.0.1:8080 weight=2; # Приоритетный сервер
      server 127.0.0.1:8081 weight=1;
  }
  ```

#### **Настройка кэширования**

Кэширование может значительно снизить нагрузку на приложение, особенно для статических данных или редко изменяющихся API-ответов.

##### Кэширование статических файлов
```nginx
server {
    location /static/ {
        root /var/www/go_app;
        expires max;
        add_header Cache-Control "public, immutable";
    }
}
```

##### Кэширование API-ответов
Для динамических данных, если их не нужно часто обновлять:
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=go_app_cache:10m max_size=1g inactive=60m use_temp_path=off;

server {
    location /api/ {
        proxy_cache go_app_cache;
        proxy_cache_valid 200 302 60m; # Кэшировать успешные ответы на 60 минут
        proxy_cache_valid 404 1m;     # Кэшировать ошибки 404 на 1 минуту
        proxy_pass http://go_app;
    }
}
```

#### Очистка кэша
Кэш можно очищать вручную, либо настроить механизм автоматической очистки по TTL.

---

#### **Настройки производительности**

##### Количества воркеров и соединений
```nginx
worker_processes auto;
worker_connections 1024;

events {
    use epoll; # Для Linux
    worker_connections 1024;
}
```

##### Настройка keep-alive соединений
```nginx
http {
    upstream go_app {
        server 127.0.0.1:8080;
        server 127.0.0.1:8081;
        keepalive 64; # Сохранять до 64 соединений
    }

    server {
        location / {
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_pass http://go_app;
        }
    }
}
```

##### Сжатие ответов
```nginx
http {
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;
}
```


#### **Безопасность**

##### Ограничение числа запросов (Rate Limiting)
```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

    server {
        location /api/ {
            limit_req zone=api_limit burst=10 nodelay;
            proxy_pass http://go_app;
        }
    }
}
```

##### HTTPS
Добавление SSL-сертификата:
```nginx
server {
    listen 443 ssl;
    server_name app.domain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://go_app;
    }
}
```

##### **Мониторинг и логирование**

##### Логирование запросов
```nginx
http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$request_time"';

    access_log /var/log/nginx/access.log main;
}
```

##### Статистика Nginx
Модуль статистики для мониторинга:
```nginx
server {
    location /status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```
### 12. Микросервисная архитектура    
Взаимодействие между микросервисами можно организовать различными способами, и выбор подхода зависит от требований приложения, инфраструктуры и командных предпочтений.

##### 1. Прямое взаимодействие через HTTP/REST API
Это самый простой способ взаимодействия между микросервисами. Каждый сервис предоставляет RESTful API, а другие сервисы обращаются напрямую по этим URL-адресам.

Преимущества:

 - Простота реализации.
 - Легкость тестирования и мониторинга каждого сервиса отдельно.
 
Недостатки:

 - Прямая зависимость сервисов друг от друга может привести к проблемам с отказоустойчивостью при сбоях одного из них.
 - Сложность масштабирования и управления конфигурацией.

 ##### 2. Использование промежуточного слоя (API Gateway)
Для улучшения управляемости и снижения зависимости между сервисами часто используют API Gateway. Это центральная точка входа для всех внешних запросов, которая маршрутизирует их к соответствующим микросервисам.

Преимущества:

 - Упрощение клиентских приложений за счет единого интерфейса.
 - Возможность централизованного управления аутентификацией, авторизацией, логированием и мониторингом.
 - Изоляция внутренних микросервисов от внешнего мира.

Недостатки:

 - Дополнительный уровень сложности.
 - Возможные проблемы с производительностью при большом количестве запросов.

 ##### 3. Обмен сообщениями через брокер сообщений (Pub/Sub)
Этот подход предполагает асинхронное взаимодействие между микросервисами через очередь сообщений. Сервисы публикуют сообщения в общую очередь, а другие подписываются на эти сообщения и обрабатывают их.

Преимущества:

 - Асинхронность позволяет повысить устойчивость системы.
 - Уменьшение прямой зависимости между сервисами.
 - Масштабируемость и надежность за счет использования очередей.

Недостатки:

 - Увеличенная сложность архитектуры.
 - Необходимость тщательного проектирования и обработки ошибок.

##### Оркестрация контейнеров
Рассмотрим два популярных инструмента для оркестрации контейнеров: Docker Swarm и Kubernetes.

**Docker Swarm**
Docker Swarm — это встроенный механизм оркестрации контейнеров в Docker. Он позволяет создавать кластеры из нескольких хостов и управлять контейнерами как единым целым.

Преимущества:

 - Простота настройки и использования.
 - Интегрирован с Docker CLI.
 - Хорошая поддержка сетевых возможностей, таких как overlay-сети.

Недостатки:

 - Ограниченные возможности по сравнению с Kubernetes.
 - Меньшая гибкость в управлении состоянием приложений.

 **Kubernetes**
Kubernetes — это более мощный инструмент для оркестрации контейнеров, который предлагает множество функций для управления приложениями в кластере.

Преимущества:

 - Высокая гибкость и расширяемость.
 - Поддержка автоматического масштабирования, самовосстановления и балансировки нагрузки.
 - Богатый набор инструментов.
Недостатки:

 - Более сложная настройка и управление.
 - Требует больше ресурсов для работы.

В целом, Docker Swarm подойдет для небольших проектов, а Kubernetes — для крупных и сложных систем.

## Часть 7: Оценка общей архитектуры  
### 13. Проектирование архитектуры    
  **Принципы проектирования**
 -Принцип разделения ответственности (Separation of Concerns):
--Разделение логики приложения на независимые модули/компоненты позволит легко поддерживать и расширять функциональность.
-Масштабируемость (Scalability):
--Архитектура должна позволять горизонтальное масштабирование, чтобы справляться с увеличением нагрузки без необходимости изменения основной структуры приложения.
-Отказоустойчивость (Fault Tolerance):
--Система должна уметь обрабатывать сбои отдельных компонентов без остановки всей работы приложения.
-Обратная совместимость (Backward Compatibility):
--При внесении изменений в архитектуру важно сохранять возможность взаимодействия с существующими системами, такими как 1С.
-Безопасность (Security):
--Внедрение современных стандартов защиты данных, таких как шифрование, аутентификация и авторизация.
-Производительность (Performance):
--Оптимизация запросов к базе данных, кэширование и использование асинхронных операций для повышения скорости обработки данных.

Компоненты и сервисы
1. Веб-сервер (GoLang)  
Основной компонент приложения будет написан на GoLang. Этот язык хорошо подходит для создания высокопроизводительных серверных приложений благодаря своей эффективности и поддержке многопоточности через горутины (goroutines).

2. API Gateway / Reverse Proxy  
Для управления входящими запросами и маршрутизации их между различными сервисами рекомендуется использовать API Gateway (например, Nginx). Это также поможет балансировать нагрузку и защищать внутренние сервисы от внешних угроз.

3. Сервис интеграции с 1С  
Отдельный микросервис, который будет отвечать за взаимодействие с 1С. Он может реализовывать RESTful API для обмена данными с другими частями системы. Для интеграции с 1С часто используют протокол HTTPS или SOAP.

4. База данных  
Для хранения больших объемов данных подойдет реляционная база данных, такая как PostgreSQL или MySQL. Если требуется работа с неструктурированными данными, можно рассмотреть NoSQL базы данных, такие как MongoDB или Cassandra.

5. Кэширование  
Для ускорения доступа к данным и снижения нагрузки на базу данных необходимо внедрить механизм кэширования. Redis – отличный выбор для этого, так как он поддерживает высокую производительность и позволяет хранить данные в оперативной памяти.

6. Асинхронная обработка задач  
Для выполнения длительных операций, таких как генерация отчетов или обработка большого количества данных, стоит использовать очередь сообщений (например, RabbitMQ или Kafka). Это позволит выполнять задачи асинхронно и не блокировать основную работу приложения.

7. Логирование и мониторинг  
Централизованная систему логирования (например, ELK Stack) и мониторинга (Prometheus + Grafana). Это поможет отслеживать состояние системы, выявлять проблемы и анализировать производительность.

8. CI/CD Pipeline  
Автоматизированный процесс сборки, тестирования и развертывания приложения с помощью инструментов непрерывной интеграции и доставки (CI/CD), таких как Jenkins, GitLab CI или CircleCI. Это ускорит разработку и уменьшит вероятность ошибок при выпуске новых версий.

9. Безопасность  
Механизмы аутентификации и авторизации пользователей, используя стандарты OAuth2 или JWT-токены. Все внешние соединения должны быть защищены SSL/TLS, а доступ к конфиденциальным данным ограничен.
