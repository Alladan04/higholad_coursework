# Проектирование высоконагруженной системы на примере eBay
## Содержание
- [Проектирование высоконагруженной системы на примере eBay](#проектирование-высоконагруженной-системы-на-примере-ebay)
  - [Содержание](#содержание)
  - [1. Тема и целевая аудитория](#1-тема-и-целевая-аудитория)
    - [Основной функционал сервиса](#основной-функционал-сервиса)
    - [Целевая аудитория](#целевая-аудитория)
    - [Основные продуктовые решения:](#основные-продуктовые-решения)
  - [2. Расчет нагрузки](#2-расчет-нагрузки)
    - [Продуктовые метрики](#продуктовые-метрики)
    - [Средний размер хранилища пользователя](#средний-размер-хранилища-пользователя)
    - [Запросы пользователей в день и RPS](#запросы-пользователей-в-день-и-rps)
    - [Сетевой трафик](#сетевой-трафик)
  - [3. Глобальная балансировка нагрузки](#3-глобальная-балансировка-нагрузки)
    - [Обоснования расположения ЦОДов](#обоснования-расположения-цодов)
    - [Расчет распределения запросов по ЦОДам](#расчет-распределения-запросов-по-цодам)
    - [DNS-балансировка](#dns-балансировка)
  - [4. Локальная балансировка нагрузки](#4-локальная-балансировка-нагрузки)
    - [L7 балансировка](#l7-балансировка)
    - [k8s](#k8s)
    - [Отказоустойчивость](#отказоустойчивость)
    - [SSL termination:](#ssl-termination)
  - [5. Логическая схема базы данных](#5-логическая-схема-базы-данных)
    - [ER диаграмма](#er-диаграмма)
    - [Расчет размера данных](#расчет-размера-данных)
  - [5. Физическая схема данных](#5-физическая-схема-данных)
    - [Запросы](#запросы)
    - [Выбор СУБД и обеспечение доступности:](#выбор-субд-и-обеспечение-доступности)
    - [Библиотеки](#библиотеки)
  - [6. Алгоритмы](#6-алгоритмы)
    - [Ранжирование объявлений](#ранжирование-объявлений)
  - [7. Технологии](#7-технологии)
  - [8. Обеспечение надеждности](#8-обеспечение-надеждности)
  - [9. Схема проекта](#9-схема-проекта)
  - [10. Расчет ресурсов](#10-расчет-ресурсов)
  - [Использованные ресурсы:](#использованные-ресурсы)
## 1. Тема и целевая аудитория
**eBay** - американская компания, предоставляющая услуги в областях 
интернет-аукционов и интернет-магазинов. Основан в 1995 году как первая в мире площадка 
для проведения интернет аукционов. С тех пор сервис активно развивается и на данный момент является
одним из крупнейших маркетплейсов.
### Основной функционал сервиса
* Список объявлений
* Поиск объявлений
* Работа с объявлением (создание, редактирование, удаление)
* Просмотр объявления
* Модерация объявлений
* Профиль пользователя
* Оценить продавца
* Рейтинг продавца
* Статитстика продаж
* Купить товар сразу
* Участвовать в аукционе (сделать ставку)
### Целевая аудитория

По данным с [hypestat](https://hypestat.com/info/ebay.com) и [отчетов ebay.com](https://investors.ebayinc.com/overview/default.aspx):
* 632+ млн сессий в месяц
* 18 млн активных продавцов (активными считаются продавцы, которые опубликовали хотя бы 1 объявление за предыдущие 12 месяцев)
* 132 млн активных покупателей (активными считаются покупатели, которые совершили хотя бы 1 покупку в предыдущие 12 месяцев)
* 135 млн активных пользоввателей суммарно
* 20+ млн сессий в день
* 146+ млн страниц просматривается посетителями ежедневно
* Основная часть покупателей сервиса (≈81%) находится в США
* 2+ млрд активных объявлений

### Основные продуктовые решения:
* Главная модель совершения сделок - аукцион.
* Альтернативная модель совершения сделок в виде функции "Купить сейчас".
* Рекомендательная система на основе действий, совершенных пользователем ранее.
## 2. Расчет нагрузки
### Продуктовые метрики
| Метрика                  | Значение              |
|--------------------------|-----------------------|
| Месячная аудитория MAU** | 45 млн. пользователей |
| Дневная аудитория DAU*   | 18 млн. пользователей |
*Из предположения, что количество уникальных пользователей в день на 10% меньше, чем количество сессий\
** Вычислено исходя из соотношения MAU/DAU аналогичных ресурсов

### Средний размер хранилища пользователя
| Данные                               | Размер ед.      | Кол-во на 1 пользователя | Суммарный объем |
|--------------------------------------|-----------------|--------------------------|-----------------|
| Профиль пользователя (Аватар + ПД)   | 70Кб + 2Кб      | 1                        | 9,05 Tб         |
| Oбъявление (Фотографии + информация) | 1200Кб * 5 + 2Кб| 111 (на 1 продавца)      | 11 Пб          |
| Отзывы                               | 1Кб             | 200  (на 1 продавца)     | 3,35 Тб         |
| Статистические данные по покупателям | 300Байт         | 10000  ( на 1 покупателя)| 360 Тб          |

### Запросы пользователей в день и RPS

Данных по действиям пользователей в день в открытом доступе нет.\
Всего поользователями просматривается порядка 146млн страниц в день => каждый пользователь просматривает примерно 146млн/18млн ≈ 8 страниц в день.
RPS = среднее_кол-во_действий_в_день * DAU / 86400

| Действие                                                | Среднее количество в день на пользователя | RPS  |
|---------------------------------------------------------|:-----------------------------------------:|------|
| Просмотр списка объявлений                              |                    16                     | 3330 |
| Поиск                                                   |                    27                     | 5620 |
| Работа с объявлением (создание/редактирование/удаление) |                     9                     | 1870 |
| Просмотр объявления                                     |                    14                     | 2910 |
| Регистрация/авторизация                                 |                     1                     | 620  |
| Профиль пользователя                                    |                     4                     | 830  |
| Оценка продавца                                         |                     2                     | 410  |
| Просмотр статистики продаж                              |                     2                     | 410  |
| Совершение покупки/ставки                               |                     3                     | 620  |
| Модерация                                               |                     _                     | 1000 |

**Пояснение:**
В [исследовании архитектуры ebay](https://www.softwaresecretweapons.com/jspwiki/resources/presentations/Sun_eBay6-2_forWeb.pdf) указывается,
что 65% от всего трафика ресурса приходится на просмотр объявлений => 5 из 8 просмотренных страниц за сессию - это 
просмотр объявлений (будем исходить из того, что это включает поиск, просмотр списка объявлений и просмотр страницы одного объявления).\
По данным [статьи ](https://o2k.ru/blog/srednyaya-konversiya) можно сделать вывод, что в среднем конверсия в покупку составляет порядка 3%.\
Запросы на модерацию происходят со стороны сотрудников компании, поэтому не могут быть посчитаны аналогично остальным запросам. 
Но можно предположить, что они происходят 1-2 раза на каждое опубликованное объявление.

Получаем средний RPS : 17530. Заложим на пиковый RPS х2 от среднего.\
Получим **пиковую нагрузку в 35060 RPS**.

### Сетевой трафик

Нагрузку на сеть можно расчитать по формуле: нагрузка[Гбит/c] = средний_трафик_на_действие * RPS * 8 / 1024 / 1024

| Действие                                                | Средний трафик на одно действие | Нагрузка на сеть |
|---------------------------------------------------------|:-------------------------------:|------------------|
| Просмотр списка объявлений/ Поиск                       |             520 Кб             | 35 Гбит/с       |
| Работа с объявлением (создание/редактирование/удаление) |             144 Кб             | 2,04 Гбит/с      |
| Просмотр объявления                                     |             870 Кб              | 19,3 Гбит/с      |
| Профиль пользователя                                    |             500 КБ              | 3,2 Гбит/с       |
| Регистрация/авторизация                                 |             100 Кб              | 0,4 Гбит/с       |
| Оценка продавца                                         |             100 Кб              | 0,3 Гбит/с       |
| Просмотр статистики продаж                              |             320 Кб              | 1 Гбит/с         |
| Совершение покупки/ставки                               |             250 Кб              | 1,1 Гбит/с       |
| Модерация                                               |             250 Кб              | 1,9 Гбит/с       |

 
Суммарная нагрузка на сеть: 65 Гбит/с \
Пусть пиковая нагрузка будет в 2 раз больше, чем средняя (также как для RPS). **Получаем пиковую нагрузку: 130 Гбит/с**

## 3. Глобальная балансировка нагрузки

### Обоснования расположения ЦОДов

В пункте [1. Тема и целевая аудитория](#1-тема-и-целевая-аудитория) обозначили, что более 80% траффика на сервис приходит с территории США.
Однако не смотря на это сервис обслуживает пользователей по всему миру: Европа, Азия, Южная Америка и даже Австралия.

Болшую часть ЦОДов имеет смысл установить на территории США, для этого можно выбрать следующие города:
- Питтсбург, штат Пенсильвания
- Портленд, штат Орегон
- Денвер, штат Колорадо
- Финикс, штат Аризона 

### Расчет распределения запросов по ЦОДам

Будем ориентироваться на карту плотности населения США и географическое положение наших ЦОД: ![map](map.png) 

| ЦОД       | Примерная область покрытия            | Приблизительный % пользователей | DAU (млн.) | RPS  | Пиковые значения RPS |
|-----------|---------------------------------------|---------------------------------|------------|------|----------------------|
| Питтсбург | Восточная часть США, Европа           | 40                              | 7,2        | 7010 | 14020                |
| Портленд  | Западная часть США, Китай             | 25                              | 4,5        | 4380 | 8760                 |
| Финикс    | Южная часть США, страны Южной Америки | 15                              | 2,7        | 2630 | 5260                 |
| Денвер    | Северная и центральная части США      | 20                              | 3,6        | 3510 | 7020                 |

### DNS-балансировка

Для балансировки нагрузки по регионам будем использовать Latency-based DNS, который будет отправлять запросы клиентов на те
ЦОДы, где минимальный RTT.

## 4. Локальная балансировка нагрузки
### L7 балансировка
В качестве L7 балансировщика выберем envoy, с помощью него будем распределять запросы между сервисами, поднятыми в k8s.
Одним из преимуществ envoy также является возможность динамического изменения конфигурации.

### k8s
Система оркестрации k8s умеет самостоятельно распределять нагрузку между подами, а также позволяет выполнять автоматическую регуляцию ресурсов в зависимости от нагрузки.

### Отказоустойчивость
Для обеспечения отказоустойчивости системы используем keepalived - реализацию протокола VRRP. 
Таким образом, если один из наших активных серверов будет сбоить - его адрес будет делегирован другому серверу.

В связке с такими возомжностями envoy, как ретраи, мониторинг и circuit breaker получим достаточно высокую отказоустойчивость нашей системы.

### SSL termination:
Для того чтобы снять нагрузку с серверов по расшифровке SSL, это будет делать L7 балансировщик.
## 5. Логическая схема базы данных
### ER диаграмма
   ![ER.svg](ER.svg)
### Расчет размера данных
| Тип       | Размер (в байтах) |
|-----------|-------------------|
| BIGINT    | 8                 |
| CHAR      | 1                 |
| INT       | 4                 |
| TIMESTAMP | 8                 |

**Listing**
```TeX
  LisingID(8) + OwnerID (8) + CategoryID (8) + Title (200) + Price (8) + Description (1000) + SellModel (4) + ImageSource (1000) + Status (4) + PublicationDate (8) = 2248 байта
``` 
**Profile**
```TeX
   ProfileID (8) + Username (100) + PasswordHash (200) + ImageSource (200) + Phone (50) + Email (50) + Rating (4) + RegistrationDate (8) = 620 байт
``` 
**Auction**
```TeX
   AuctionID (8) + ListingID (8) + LastBidID (8) + CurrentPrice (8) + Step (8) + Status (4) + Deadline (8)=  52 байта
```

**Bid**
```TeX
   BidID (8) + AuctionID (8) + BuyerID (8) + CreatedAt (8) =  32 байта
```

**Order**
```TeX
  OrderID (8) + ListingID (8) + BuyerID (8) + SellerID (8) + AddressID (8) + Status (4) + CreatedAt (8) + DeliveryModel (4) = 56 байта
```
**Address**
```TeX
  AddressID (8) + ProfileID (8) + Country (50) + City (50) + Street (100) + House (50) + Flat (50) = 316 байт
```
**Category**
```TeX
  CategoryID (8) + ParentID (8) + Title (200) = 216 байт
```

**Comment**
```TeX
  CommentID (8) + CommentatorID (8) + ListingID (8) + CreatedAt (8) + Mark (4) + Text (500) = 536 байт
  
```
**UserAction**
```TeX
  ActionID (8) + ListingID (8) + UserID (8) + Type (4) + Time (8) + Features (256)= 292 байт
  
```

| Таблица        | Размер записи (байт) | Количество записей | Итого    |
|----------------|----------------------|--------------------|----------|
| **Listing**    | 2248                 | 2 млрд             | 4,8 Тб   |
| **Profile**    | 620                  | 200 млн            | 115 Гб   |
| **Auction**    | 52                   | 1,6 млрд*          | 77,5 Гб  |
| **Bid**        | 32                   | 16 млрд***         | 476,8 Гб |
| **Order**      | 56                   | 4 млрд             | 208,6 Гб |
| **Address**    | 316                  | 200 млн            | 58,8 Гб  |
| **Category**   | 216                  | 1000               | 0,2 Мб   |
| **Comment**    | 536                  | 1 млрд             | 449 Гб   |
| **UserAction** | 292                  | 100 млрд****       | 26,6 Тб  |

** В сервисе 2млрд активных объявлений, при этом считаем, что основная модель продаж - аукцион. Предположим, что 80% объявлений выставлены на аукцион, а 20% объявлений прдаются по фиксированной стоимости.\
***Из предположения, что на один аукцион в среднем приходится 10 ставок\
****Считаем, что храним данные о всех действиях пользователей за год

| Таблица        | Чтение (QPS) | Запись (QPS) |
|----------------|--------------|--------------|
| **Listing**    | 11860        | 3490         |
| **Profile**    | 4360*        | 2000         |
| **Auction**    | 11860**      | 2490         |
| **Bid**        | 11860        | 5500         |
| **Order**      | 2000         | 1500         |
| **Address**    | 4000         | 100          |
| **Category**   | 8950         | -            |
| **Comment**    | 3710         | 500          |
| **UserAction** | 1000         | 30000        |

*Считаем, что при просмотре страницы объявления - также делается запрос на чтение в таблицу поользователей для получения и отображения информации о продавце\
**На всех страницах, где получаются объявления необходимо также отображать информацию о состоянии аукциона.\
***На всех страницах с объявлениями показывается информация о количестве ставок.\

## 5. Физическая схема данных
### Запросы

| Таблица | Запросы                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Listing | ```SELECT ListingID, OwnerID,   CategoryID,   Title,   Price,   Description, SellModel , ImageSource  ,Status ,PublicationDate FROM Listing WHERE ListingID=...``` - получение объявления по идентификатору<br> ```SELECT ListingID, OwnerID,   CategoryID,   Title,   Price,   Description, SellModel , ImageSource  ,Status ,PublicationDate FROM Listing WHERE ... LIMIT N OFFSET M``` - получение списка объявлений с фильтрацией (по CategoryID,OwnerID, Price  и т.д.)<br>```INSERT INTO Listing  (ListingID, OwnerID,   CategoryID,   Title,   Price,   Description, SellModel , ImageSource  ,Status ,PublicationDate) VALUES (...)```- создание объявления <br>  ```UPDATE Listing SET ... WHERE ListingID= ...```- редактирование объявления |
| Profile | ```SELECT ProfileID, Username, PasswordHash, ImageSource , Phone, Email, Rating, RegistrationDate FROM Profile  WHERE ProfileID=...``` -  получение данных о пользоавтеле по идентификатору<br>```INSERT INTO Profile  (ProfileID, Username, PasswordHash, ImageSource , Phone, Email, Rating, RegistrationDate ) VALUES (...)```- создание профиля пользователя<br> ```UPDATE Profile  SET ... WHERE ProfileID=...```- редактирование профиля пользователя<br>                                                                                                                                                                                                                                                                                        |
| Auction | ```SELECT AuctionID , ListingID, LastBidID, CurrentPrice, Step, Status, Deadline FROM Auction  WHERE ListingID=...``` - получение информации об аукционе<br>```INSERT INTO Auction  (AuctionID , ListingID, LastBidID, CurrentPrice, Step, Status, Deadline) VALUES (...)```- создание профиля нового аукциона<br> ```UPDATE Auction SET ... WHERE AuctionID=...```- редактирование информации об аукционе<br>                                                                                                                                                                                                                                                                                                                                         |
| Bid     | ```SELECT BidID , AuctionID, BuyerID, CreatedAt FROM Bid  WHERE AuctionID=... LIMIT N OFFSET M``` - получение информации о ставках на аукционе<br>```SELECT BidID , AuctionID, BuyerID, CreatedAt FROM Bid  WHERE BuyerID=... LIMIT N OFFSET M``` - получение информации о ставках  пользователя<br>```INSERT INTO Bid  (BidID , AuctionID, BuyerID, CreatedAt ) VALUES (...)```- создание ставки<br>                                                                                                                                                                                                                                                                                                                                                  |
| Order   | ```SELECT OrderID , ListingID, BuyerID, SellerID , AddressID, Status, CreatedAt, DeliveryModel FROM Order  WHERE OrderID=...``` - получение информации о заказе по идентификатору<br>``````SELECT OrderID , ListingID, BuyerID, SellerID , AddressID, Status, CreatedAt, DeliveryModel FROM Order  WHERE ... LIMIT N OFFSET M``` - получение информации о заказе по BuyerID/SellerID и доп фильтрам (например, статусу или дате создания)<br>```                                                                                                                                                                                                                                                                                                       |

### Выбор СУБД и обеспечение доступности:
| СУБД          | Данные                   | Обеспечение доступности                                                                                                                             | Индексы                                                      |
|---------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Postgresql    | Profile                  | - MS-репликация<br> - Шардирование по хэшу ProfileID                                                                                                | Unique Email, Unique Phone                                   |
| Postgresql    | Address                  | - MS-репликация<br> - Шардирование по хэшу от  ProfileID                                                                                            | Индекс по ProfileID                                          |
| ElasticSearch | Listing                  | - Репликация средствами ElasticSearch <br> Шардирование по дате<br> - Архивные объявления из Elastic удаляются (так как по ним поиск не происходит) <br> Храним только данные необходимые для ранжирования| По Title, Description, CategoryID, Features, PublicationDate |
| MongoDB       | Listing                  | - MS-репликация <br> Заведем 2 версии БД - одна шардирована по ListingID, другая - по OwnerID , при этом вторая таблица служит только для поиска и содержит лишь номер шарда, из которого можно получить исокмые данные<br> - Архивные объявления хранятся отдельно  <br>    | OwnerID/ListingID <br>PublicationDate <br>                   |
| Postgresql    | Comment                  | MS-репликация <br> - Шардинг по хэшу от ProfileID                                                                                                   | Индекс по ProfileID                                          |
| Postgresql    | Auction                  | MS-репликация <br> - Шардтинг по хэшу от AuctionID <br> - Добавить к Listing поле AuctionID                                                         | Индекс по ListingID <br> Индекс по SellerID                  |
| Postgresql    | Bid                      | MS-репликация <br> - Шардинг по хэшу от AuctionID                                                                                                   | Индекс по AuctionID<br>Индекс по BuyerID, CreatedAt          |
| Postgresql    | Order                    | MS-репликация <br> - Храним только активные сделки, а завершенные переносим в CLickHouse и храним как исторические данные                           | Индекс по BuyerID<br> Индекс по SellerID <br>                |
| ClickHouse    | UserAction_Raw           | MS-репликация <br> - Партицирование по дате <br> - Шардинг по хэшу от ProfileID                                                                     | -                                                            |
| ClickHouse    | UserAction_AggrByListing | MS-репликация <br> - Партицирование по дате <br> - Шардинг по хэшу от ListingID                                                                     | -                                                            |
| CEPH          | FileStorage              | -                                                                                                                                                   | -                                                            |

[//]: # ()
[//]: # (Есть вариант денормализовать Profile и хранить у каждого пользователя список объявлений. Но он может быть очень большим. И тогда )

[//]: # (тоже нужно поддерживать консистеность)
#### Обеспечение консистентности между ELasticSearch и Mongo
Крон, который записывает изменения из MongoDB в ES "на лету". 
Например,использовать [Monstache](https://github.com/rwynn/monstache).

### Библиотеки
PostgreSQL: jackc/pgx

Redis: go-redis

ClickHouse: clickhouse/clickhouse-go

Elasticsearch: olivere/elastic

CEPH: go-ceph

Mongo: mongodb/mongo-go-driver
## 6. Алгоритмы
### Ранжирование объявлений
Запрос на поиск включает в себя две части: 
  - параметры для определения базовой выборки (введенная для поиска строка, фасетные фильтры)
  - параметры для ранжирования объявлений выборки которые собираются на основе пользовательских логов (история запросов, просмотров, покупок, адреса, рейтинг продавца и др.)

#### Этап 1. Подбор кандидатов
Средствами ES ищем все объявления-кандидаты на основе параметров поиска. При этом полнотекстовый поиск ведется не только на точное совпадение, но и близкие по смыслу.
На этом этапе также происходит определение релевантности на основе таких метрик, как **TF/IDF (Term Frequency-Inverse Document Frequency)**.
Возращаем топ-500 элементов.
#### Этап 2. Ранжирование по конверсии
Используем бинарный классификатор, обученный на логах просмотров и покупок пользователей.
Выборки образуются получением свойств рассматриваемого объявления и его рекомендуемых пар. Например, если пользователь 
просмотрел товар и приобрел его то он отмечается как позитивный класс, иначе - негативный.
Таким ообразом объявления, которые с большей вероятностью приведут к покупке будут оценены как более релевантные
#### Этап 3. Персонализация
Здесь используем модель  обученную на истории запросов/просмотров пользователя, которая может предсказать релевантность
объявления на основе поведения пользователя.

## 7. Технологии

| Технология           | Применение                          | Мотивация                                                                                                                             |
|----------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Go                   | Backend, основной язык сервисов     | Производительность, удобен для микросервисной архитектуры, низкий порог входа,  большое количество технологий                         |
| Python               | Backend, модель для рекомендаций    | Решения из коробки, быстрая и дешевая разработка                                                                                      |
| TypeScript           | Frontend                            | Строгая типизация, компонентный подход, быстрая разработка                                                                            |
| Envoy                | Proxy balancer                      | Поддерживает gRPC, разрабатывался для поддержания микросервисных архитектур, и является мощным инструментом для балансировки нагрузки |
| Kafka                | Брокер сообщений                    | Возможность снизить нагрузку на БД, более высокая пропускная способность                                                              |
| PostgreSQL           | Хранилище SQL, основная БД сервисов | Подходит для реляционного хранения данных большинства CRUD-сервисов, надежный, производительный                                       |                                                                                  |
| Elasticsearch        | Поисковый движок                    | Полнотекстовый поиск, распределенная система                                                                                          |
| ClickHouse           | Хранилище аналитических данных      | Эффективная работа с OLAP-нагрузкой, производительный                                                                                 |
| CEPH                 | Хранилище статики: фото, превью     | Не дорогой вариант, надежный, масштабируемый                                                                                          |
| Vault                | Хранилище секретов                  | Удобный, популярный                                                                                                                   |
| Grafana + Prometheus | Графики, мониторинг и алёрты        | Удобный, популярный, гибкий                                                                                                           |
| Kubernetes + Docker  | Deploy                              | Масштабирование, отказоустойчивость, оптимальная утилизация ресурсов                                                                  |

## 8. Обеспечение надеждности

| **Компонент**        | **Метод**                                                     | **Обоснование**                                                                                                                                                                                                                                                                                                          |
|-----------------------|---------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Envoy**            | Резервирование                                                | Необходимо иметь два Envoy в каждом ДЦ, каждый будет на одинаковом IP-адресе; второй Envoy будет подменять при отказе первого для обеспечения высокой доступности.(Использование Virtual IP через Keepalived,)                                                                                                           |
| **Kubernetes**       | Health-check, Load Balancing                                  | Необходим для балансировки нагрузки между подами и контроля жизни подов для их своевременной замены и перезапуска, что повышает отказоустойчивость приложения.  Использование как минимум 3 мастер-нод для обеспечения кворума (N+1 резервирование), автоматическое восстановление подов через ReplicaSet и StatefulSet. |
| **Redis**            | Репликация                                                    | Redis используется для авторизации пользователя; в случае сбоя одной БД, её подменяет реплика, что гарантирует доступность данных и минимальные задержки. Используем автоматический фэйловер.                                                                                                                                                                |
| **PostgreSQL**       | Репликация                                                    | Репликация критична для сохранения важных данных, чтобы предотвратить их потерю при отказе основного узла.Также необходимы регулярные бэкапы данных.    Используем patronic для реализации фэйлловера                                                                                                                                                                 |
| **MongoDB**          | Репликация                                                    | Для обеспечения доступности и сохранности данных об объявлениях  настройка реплика-сетов с минимум 3 нодами, где 1 нода выступает как резервная (N+1)                                                                                                                                                                    |
| **CEPH**             | Репликация                                                    | Репликация обеспечивает сохранность данных, таких как сгенерированные изображения пользователей, даже при сбое одного из узлов.CRUSH-алгоритм для распределения данных                                                                                                                                                                                         |
| **Backend services** | Репликация, Circuit Breaker, Retry Budget, Graceful Degradation | Репликация — несколько инстансов каждого сервиса для отказоустойчивости; Circuit Breaker — для уменьшения нагрузки на проблемный сервис; Retry Budget — для безопасных повторных запросов; Graceful Degradation — минимизирует влияние недоступности данных.                                                             |
| **Все компоненты**   | Метрики, логи, трейсы, алерты                                 | Для обеспечения Observability, которая позволяет быстро выявлять и устранять проблемы, обеспечивая стабильность всей системы.                                                                                                                                                                                            |
| **Физические серверы** | Резервирование (N+1)                                          | Использование резервных серверов, блоков питания и систем охлаждения для предотвращения отказов из-за аппаратных проблем.                                                                                                                                                                                                |
| **Датацентры**       | Географическое резервирование                                 | Размещение серверов и данных в нескольких датацентрах для защиты от локальных катастроф и повышения доступности.                                                                                                                                                                                                         |
| **Elasticsearch**    | Репликация, Snapshot и Backup                                 | Репликация индексов для повышения отказоустойчивости; регулярные снапшоты и бэкапы позволяют восстанавливать данные при сбоях.                                                                                                                                                                                           |
| **Kafka**            | Репликация, ISR (In-Sync Replicas)                            | Репликация топиков и синхронные реплики гарантируют сохранность и доступность данных даже при сбое брокера.                                                                                                                                                                                                              |

## 9. Схема проекта

![scheme](schema.svg)
*МБ юзать Change Data Capture? 
## 10. Расчет ресурсов

Для расчета ресурсов используются данные о целевой пиковой нагрузке (RPS), объеме хранимых данных, и пропускной способности сети. Основной метод – оценка удельного потребления ресурсов (CPU, RAM) на единицу нагрузки, с последующим масштабированием под пиковую нагрузку.

#### Таблица расчетов ресурсов

| **Сервис**         | **Целевая пиковая нагрузка (RPS)** | **CPU**  | **RAM** | **Net**   |
|--------------------|------------------------------------|----------|---------|-----------|
| **Listing**        | 15000                              | 150 ядер | 600 Гб  | 2 Гбит/с |
| **Search**         | 20000                              | 200 ядер | 800 Гб  | 3 Гбит/с |
| **Profile**        | 6400                               | 64 ядра  | 250 Гб  | 2 Гбит/с  |
| **Authentication** | 5000                               | 50 ядер  | 200 Гб  | 3 Гбит/с  |
| **Auction**        | 13000                              | 130 ядер | 500 Гб  | 15 Гбит/с |
| **Order**          | 3500                               | 35 ядер  | 120 Гб  | 4 Гбит/с  |
| **User Rating**    | 4000                               | 40 ядер  | 150 Гб  | 5 Гбит/с  |
| **Stats**          | 10000                              | 100 ядер | 500 Гб  | 10 Гбит/с |
| **PostgreSQL**      | 48,360                             | 150            | 600 ГБ  | 6              |
| **MongoDB**         | 15,350                             | 100            | 800 ГБ  |  3              |
| **Elasticsearch**   |  11,860                             | 120            | 700 ГБ  |  4              |
| **ClickHouse**      | 31,000                             | 200            | 1.2 ТБ  | 30 ТБ         | 6              |

---
### Обновленная таблица серверов с расчетом амортизации на 5 лет

| **Назначение**       | **Хостинг** | **Конфигурация**                           | **Количество** | **Стоимость за единицу (покупка)** | **Итоговая стоимость** | **Амортизация в месяц** |
|----------------------|-------------|--------------------------------------------|----------------|------------------------------------|------------------------|-------------------------|
| **Kubernetes Nodes** | Собственный | 2xAMD EPYC 7313 / 256GB RAM / 4xNVMe 4TB   | 50             | $15,000                            | $750,000               | $12,500                 |
| **ElasticSearch**    | Собственный | 2xAMD EPYC 7313 / 256GB RAM / 4xNVMe 4TB   | 20             | $15,000                            | $300,000               | $5,000                  |
| **Redis**            | Собственный | 1xAMD EPYC 7313 / 128GB RAM / 2xNVMe 2TB   | 10             | $8,000                             | $80,000                | $1,333                  |
| **PostgreSQL**       | Собственный | 2xAMD EPYC 7313 / 256GB RAM / 4xNVMe 4TB   | 15             | $15,000                            | $225,000               | $3,750                  |
| **ClickHouse**       | Собственный | 2xAMD EPYC 7313 / 256GB RAM / 4xNVMe 4TB   | 15             | $15,000                            | $225,000               | $3,750                  |
| **CEPH Storage**     | Собственный | 1xIntel Xeon 4314 / 128GB RAM / 8xHDD 16TB | 30             | $10,000                            | $300,000               | $5,000                  |

---
## Использованные ресурсы:
1. https://hypestat.com/info/ebay.com
2. https://www.ebayinc.com/company/
3. https://investors.ebayinc.com/overview/default.aspx
4. [eBay Creates Technology Architecture for the Future By David S. Marshak](https://www.softwaresecretweapons.com/jspwiki/resources/presentations/Sun_eBay6-2_forWeb.pdf) 
5. [The architecture of eBay search](https://ceur-ws.org/Vol-2311/paper_14.pdf)
6. [eBay’s Architectural Principles Architectural Strategies, Patterns, and Forces for Scaling a Large eCommerce Site, Randy Shoup, 2008](http://jaoo.dk/dl/qcon-london-2008/slides/RandyShoup_eBaysArchitecturalPrinciples.pdf)
