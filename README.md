# Проектирование высоконагруженной системы на примере eBay
## Содержание
1. [Тема и целевая аудитория](#1-тема-и-целевая-аудитория)
   1. [Основной функционал сервиса](#основной-функционал-сервиса)
   2. [Целевая аудитория](#целевая-аудитория)
   3. [Основные продуктовые решения](#основные-продуктовые-решения)
2. [Расчет нагрузки](#2-расчет-нагрузки)
3. [Использованные ресурсы](#использованные-ресурсы)
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
Метрика      | Значение
-------------| -------------
Месячная аудитория MAU        | 630 млн. пользователей
Дневная аудитория DAU         | 20 млн. пользователей

### Средний размер хранилища пользователя
| Данные                               | Размер ед.      | Кол-во на 1 пользователя | Суммарный объем |
|--------------------------------------|-----------------|--------------------------|-----------------|
| Профиль пользователя (Аватар + ПД)   | 70Кб + 2Кб      | 1                        | 9,05 Tб         |
| Oбъявление (Фотографии + информация) | 120Кб * 5 + 2Кб | 111 (на 1 продавца)      | 1,1 Пб          |
| Отзывы                               | 1Кб             | 2000  (на 1 продавца)    | 33,5 Тб         |
| Статистические данные по покупателям | 300Байт         | 1000  ( на 1 покупателя) | 36 Тб           |



## Использованные ресурсы:
1. https://hypestat.com/info/ebay.com
2. https://www.ebayinc.com/company/
3. https://investors.ebayinc.com/overview/default.aspx
