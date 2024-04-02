# Итоговое задание по "Программирование на Go"
## Запуск
Сначала необходимо скачать исходный код проекта и перейти в скачанную папку.
```
git clone https://github.com/alien-agent/users-balance-microservice

cd users-balance-microservice
```
Для запуска API сервера используется следующая команда:
```
docker-compose up
```
Для запуска тестов:
```
docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit
```
По умолчанию сервер будет доступен по адресу http://localhost:8080/.

## Описание API

Детальное описание каждого endpoint'а с примерами открывается по клику:

- [Получить баланс пользователя](https://github.com/korol787/users-balance-microservice/blob/master/docs/balance.md)
  :`POST /v1/deposits/balance`
- [Изменить баланс пользователя](https://github.com/korol787/users-balance-microservice/blob/master/docs/update.md)
  :`POST /v1/deposits/update`
- [Перевести деньги между двумя пользователями](https://github.com/korol787/users-balance-microservice/blob/master/docs/transfer.md)
  :`POST /v1/deposits/transfer`
- [Получить историю операций пользователя](https://github.com/korol787/users-balance-microservice/blob/master/docs/history.md)
  :`POST /v1/deposits/history`

Также есть небольшая коллекция запросов для запуска в Postman, которая находится в файле [postman_examples.json](https://github.com/korol787/users-balance-microservice/blob/master/postman_examples.json).
Для получения ожидаемых ответов сервера рекомендуется отправлять запросы в исходном порядке.

## Детали реализации

Микросервис написан на Go, СУБД - PostgreSQL. Все запросы/ответы принимаются/отдаются в формате JSON.

#### Как обеспечивается устойчивость данных о балансе пользователя?
1. На уровне *SQL Schema* установлен запрет на сохранение отрицательных балансов - даже в случае наличия бага в коде метод
   получит ошибку базы данных и сообщит о ней внешнему сервису.
2. Методы API, подразумевающие неоднократную запись в базу данных, вносят все изменения в БД в одной транзакции. В
случае если в процессе работы происходит ошибка, все изменения откатываются и API сообщает внешнему сервису об ошибке. Например,
при переводе денег между пользователями со счета отправителя сняли нужную сумму, но при начислении ее получателю произошла ошибка:
`TransactionHandler` гарантирует, что произойдет откат баланса отправителя, после чего API вернет ошибку.
3. Для представления баланса пользователя в API и БД используются 64-битные числа, что достаточно для представления любой
   практической суммы денег.

#### Конфигурация
Некоторые параметры сервиса можно настраивать с помощью файлов конфигурации. Доступны следующие параметры:
 - `server_port` - порт, на котором API сервер будет принимать запросы
 - `rates_expiration` - срок актуальности (частота обновления) курсов обмена валют
 - `dsn` - строка с настройками подключения к БД PostgreSQL

По умолчанию используется файл конфигурации `dev.yml`, а при запуске внутри Docker - `local.yml`. Также возможна 
конфигурация с помощью переменных среды - их приоритет выше, чем у файлов конфигурации. Соответствующие переменные среды
должны иметь префикс `APP_`, например: `APP_DSN`.

#### Реализация интеграции с валютым API
Конвертация валют происходит с использованием курса обмена валют с [бесплатного API](https://api.exchangerate.host/latest).
Если API обменных курсов недоступен, сервис API продолжает работать в штатном режиме, выдавая ошибку только при запросе 
баланса в отличной от рубля валюте. Курсы валют кэшируются на 10 минут (время можно настроить в файле конфигурации - 
параметр `rates_expiration`). 

#### Реализация транзакций
История изменений баланса пользователя представлена набором транзакций. Каждая транзакция имеет 4 основных поля - ID 
отправителя, ID получателя, сумма транзакции, дата и время транзакции. 

С помощью транзакций реализованы 3 типа операций с балансом пользователя:
1. **Пополнение баланса** - ID отправителя отсутствует (равно Nil UUID).
2. **Оплата услуги** - ID получателя отсутствует (равно Nil UUID).
3. **Денежный перевод** - ID отправителя и получателя указаны и не равны Nil UUID.

ID отправителя и получателя являются "позиционными" - транзакция всегда отражает **списание** денег со счета отправителя 
и **зачисление** на счет получателя, поэтому и сумма транзакции всегда положительна.

## Структура проекта

```
.
├── cmd                  main applications of the project
│   └── server           the API server application
├── config               configuration files for different environments
├── docs                 API endpoints documentation
├── internal             private application and library code
│   ├── config           configuration library
│   ├── deposit          deposit-related features
│   ├── entity           database models
│   ├── errors           error types and handling
│   ├── rates            exchange rates service
│   ├── requests         storing and validating requests' data
│   ├── test             helpers for testing purpose
│   └── transaction      transaction-related features
├── pkg                  public library code
│   ├── accesslog        access log middleware
│   ├── dbcontext        db transaction helpers
│   ├── log              structured and context-aware logger
└── testdata             test data scripts
```

Этот проект основан на [Go RESTful API Starter Kit](https://github.com/qiangxue/go-rest-api).