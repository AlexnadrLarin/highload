# highload VK

Курсовая работа в рамках дисциплины "Проектирование высоконагруженных сервисов"<br> по теме: 
"Сервис по поиску свободных парковочных мест".

**Автор** - Ларин Александр WEB-31

## Содержание
  [1. Тема, функционал и аудитория](#1-тема-функционал-и-аудитория)<br>
  [2. Расчет нагрузки](#2-расчет-нагрузки)<br>
  [3. Глобальная балансировка](#3-глобальная-балансировка)<br>
  [4. Локальная балансировка](#4-локальная-балансировка)<br>
  [5. Логическая схема БД](#5-логическая-схема-бд)<br>
  [6. Физическая схема БД](#6-физическая-схема-бд)<br>
  [7. Алгоритм](#7-алгоритм)<br>
  [8. Технологии](#8-технологии)<br>
  [9. Схема проекта](#9-схема-проекта)<br>
  [10. Расчёт ресурсов](#10-расчёт-ресурсов)<br>


## 1. Тема, функционал и аудитория

### Тема
В качестве темы был выбран личный проект - сервис, который помогает водителям найти <br> ближайшее свободное парковочное место и сэкономить время.

### Ключевой функционал продукта
- Отображение парковок на карте;
- Просмотр количества свободных мест на выбранной парковке;
- Добавление/удаление избранных парковок.

### Целевая аудитория
Проект взят, как теоретический, поэтому продуктовые метрики взяты у сервиса Яндекс Навигатор, исходя из <br> того, что многие водители и пассажиры пользуются именно ими.
- 24 млн. активных пользователей в месяц;
- 59% аудитории - в возрасте от 25 до 44 лет;
- 35% - женщины;
- 65% - мужчины;
- 66% с доходом выше среднего.


## 2. Расчет нагрузки
- 24 млн. активных пользователей в месяц;
- дневной охват может варьироваться от 2,4 до 7,2 миллиона пользователей;
- Количество парковок всего - 344 150.

### Рассмотрим основные типы данных:
- Информация о парковке занимает 140 бит;
- Размер фото равен примерно 400Кб(таких фото будет около 3-4 на парковке), итоге 1600Кб;
- Информация о пользователе занимает 116 бит;
- Информация об одной парковке, добавленной в избранное пользователя, занимает 24 бита, но т.к. <br> максимум в избранное можно добавить 10 парковок, будем считать объем избранного для одного пользователя равным 240 бит;
- Информация о сессии занимает 74 бита;

### Среднее количество действий пользователя:

При тестировании первой версии продукта было установлено, что пользователь открывает приложение 2-3 раза в день.
По наблюдениям выявим количество действий пользователя(примерно).

| Действие                     | Количество действий(в среднем) |
|------------------------------|--------------------------------|
| Авторизация                  | 2 раза в день                  |
| Просмотр фотографии парковки | 17 раз в день                  |
| Просмотр мест на парковке    | 15 раз в день                  |
| Просмотр избранного          | 2 раза в день                  |
| Добавление в избранное       | 1 раз в месяц                  |

Удаление парковки из избранного и регистрация рассматриваться не будут, т.к. это происходит крайне редко.

### RPS
DAU = 7.2 M <br>
RPS = Количество действий на пользователя * DAU / 86 400 <br>
RPD = Количество действий на пользователя * DAU <br>

| Действие | RPS    | RPD     |
|----------|--------|---------|
| Авторизация | 166.67 | 14 400 000 |
| Просмотр фотографии парковки | 1416   | 480 000 |
| Просмотр мест на парковке | 1250   | 108 000 000 |
| Просмотр избранного | 166.67 | 14 400 000 |
| Добавление в избранное | 2.78   | 240 000 |

**RPS(общий)** = 3002<br>
**RPS(пиковый)** = 4503<br>

### Сетевой трафик
| Действие | Трафик     | Пиковый трафик |
|----------|------------|----------------|
| Авторизация | 2.84 Кб/c  | 4.26 Кб/c      |
| Просмотр фотографии парковки            | 2.16 Гб/c  | 3.24 Гб/c      |
| Просмотр мест на парковке | 21.36 Кб/c | 32.04 Кб/c     |
| Просмотр избранного | 4.88 Кб/c  | 7.32 Кб/c      |
| Добавление в избранное | 66.72 б/c  | 100 б/c        |

### Прирост аудитории
За три года MAU Яндекс Карт удалось увеличить в 2,5 раза [[4](https://mindbox.ru/journal/experts/it-brand/)]

((140 + 116 + 74 + 240) / 8 / 1024 + 1600) * 14.4 * 10^6 / 1024^2 = 21.458 Тб
## 3. Глобальная балансировка

Целевой аудиторией сервиса являются пользователи России, поэтому необходимо проанализировать плотность населения, как на рисунке ниже, и оптимально расположить ЦОДы:

Большая часть пользователей находиться в Европейской части России, поэтому оптимально расположить датацентры именно там:<br>
- Москва: Как столица и крупнейший город России, Москва является центром технологического развития и имеет крупные узлы связи;
- Санкт-Петербург: Второй по величине город в России, он также имеет развитую инфраструктуру и крупные центры данных;
- Екатеринбург: Расположенный в центре России, он может обеспечить хорошее покрытие для регионов Урала и Сибири.

Использую GeoDNS определяется физически ближайший к пользователю ЦОД. Далее с помощью BGP Anycast найти для каждого района свой ЦОД, чтоб улучшить балансировку.

## 4. Локальная балансировка

Для картографического сервиса предлагаю использовать следующие технологии:<br>
### 1. Kubernetes
Kubernetes обеспечивает масштабируемость, отказоустойчивость и управление ресурсами, что важно для сервисов, <br> которые работают с картами, особенно при обработке больших объемов запросов.

Причины выбора Kubernetes:<br>
**- Масштабируемость:** Kubernetes позволяет легко масштабировать сервисы горизонтально и вертикально в зависимости от нагрузки. Это полезно для данного сервиса, который может иметь пиковые нагрузки в определенные периоды времени;

**- Управление ресурсами:** Kubernetes позволяет управлять ресурсами контейнеров, что позволяет оптимизировать использование вычислительных ресурсов и обеспечить стабильную производительность сервиса;

**- Отказоустойчивость:** Kubernetes автоматически управляет перезапуском контейнеров в случае их сбоев;

**- Управление конфигурацией и развертыванием:** Kubernetes предоставляет инструменты для управления конфигурацией и автоматизации развертывания, что упрощает процесс управления и обновлениями сервиса.

### 2. Nginx на уровне L7
Nginx будет использоваться для балансировки нагрузки между контейнерами Kubernetes.

Причины выбора Nginx:<br>
**- Высокая производительность:** Nginx известен своей высокой производительностью и эффективностью в обработке большого количества одновременных соединений и запросов;

**- Оптимизация под медленные клиентские соединения:** Nginx имеет встроенные механизмы для оптимизации работы с медленными клиентами. Например, он может настроить таймауты соединения, управлять буферизацией ответов и настроить параллельные обработчики соединений для улучшения производительности.

Также можно использовать Weighted Least Connections для того, чтобы распределять запросы на сервисы.

### 3. SSL termination
Для шифрования и быстрого автоматического получения SSL-сертификатов будем использовать Let`s Encrypt. Также, чтобы сэкономить время для установки соединения, можно кешировать сессию - использовать Session Cache.


## 5. Логическая схема БД
На рисунке ниже представлена схема базы данных:
![image](https://github.com/AlexnadrLarin/highload/assets/47739783/6f58d162-4b83-441d-a979-44e351809e75)

### Описание таблиц <br>
- Таблица users содержит информацию о пользователях (PK - id пользователя, email, хэш-пароль);
- Таблица sessions содержит информацию о сессиях пользователя (PK - id сессии, дата создания, FK - id пользователя);
- Таблица parking_lots содержит информацию о парковках (PK - id парковки, адрес, координаты, количество свободных мест, количество занятых мест);
- Таблица views содержит информацию о парковках (PK - id парковки, id камеры, FK - id парковки);
- Таблица rows содержит информацию о парковках (PK - id ряда, координаты ряда, вместительность ряда, количество свободных мест, дата обновления, FK - id вида);
- Таблица favourites содержит информацию об избранных парковках пользователя (PK - id избранной парковки, FK - id пользователя, id парковки).
- Таблица parking_lots_photos содержит фотографии парковочных мест.

### Размер данных <br>
Количество зарегистрированных пользователей возьмем с запасом в 5 раз, т.е. в 2 раза больше прироста:
- **Users:** <br>
16(UUID) + 50(email) + 50(password) = 116 бит<br>
116 бит * 24млн * 5 = 1.6205 Гб
<br><br>
- **Sessions:** <br>
50(session_id) + 16(UUID) +  8(date) = 74 бит<br>
74 бит * 24млн * 5 = 1.033 Гб
<br><br>
- **Favourites:** <br>
4(id) + 16(UUID) +  4(parking_lot_id) = 24 бит<br> 
24 бит * 24млн * 5 * 10 (максимальное количество парковок в избранном) = 3.353 Гб
<br><br>
Также посчитаем размер данных для аватарок: <br>
1600 Кб * 24 * 10^6 * 5 / 1024^3 = 178.81 Тб <br><br>
Количество парковок возьмем из п.2: <br>
- **Parking_lots:** <br>
4(id) + 32(coordinates) + 50(city) + 50(street) + 4(house) = 140 бит<br>
140 бит * 344 150 = 45.95 Мб
<br><br>
- **Views:** <br>
4(id) + 4(parking_lot_id) + 4(camera) = 12 бит <br>
12 бит * 344 150 * 10(максимальное количество видов) = 39.385 Мб
<br><br>
- **Rows:** <br>
4(id) + 4(view_id) + 32(coordinates) + 4(capacity) + 4(free_places) + 8(date) = 56 бит<br>
56 бит * 344150 * 10 (максимальное количество видов) * 2(оптимальное кол-во рядов для распознавания) = 367.592 Мб
<br><br>
Рассчитаем RPS на чтение/запись для каждой таблицы:
- Таблица Users:<br>
Мы обращаемся к этой таблице при смене аватарки, просмотре избранного, добавление в избранное, следовательно RPS на чтение равен 175.
RPS на запись возьмём 5 (примерно)
- Таблица Sessions:<br>
Мы обращаемся к этой таблице при авторизации, поэтому возьмем RPS на чтение из пункта 2.
RPS на запись возьмём 5 (примерно)
- Таблица Favourites:<br>
Мы обращаемся к этой таблице при просмотре избранного, добавление в избранное, следовательно RPS на чтение равен 169,45.
RPS на запись возьмём 2.78, исходя из пункта 2.
- Таблица Parking_lots:<br>
Мы обращаемся к этой таблице при просмотре избранного, добавление в избранное, при просмотре мест на парковке, при просмотре фотографий парковки, следовательно RPS на чтение равен 169,45.
RPS на запись возьмём 5 (примерно)
- Таблица Views:<br>
Мы обращаемся к этой таблице при просмотре избранного, добавление в избранное, при просмотре мест на парковке, следовательно RPS на чтение равен 169,45.
RPS на запись возьмём 5 (примерно)
- Таблица Rows:<br>
Мы обращаемся к этой таблице при просмотре избранного, добавление в избранное, при просмотре мест на парковке, следовательно RPS на чтение равен 169,45.
Запись производится регулярно, раз в 30 секунд, следовательно RPS:<br>
344 150(парковок) * 10(видов максимум) * 1/30 = 17200


| Таблица       | Чтение  | Запись |
|---------------|---------|--------|
| Users         | 175     | 5      |
| Sessions      | 166.67  | 5      |
| Favourites    | 169,45  | 2.78   |
| Parking_lots  | 2835,45 | 5      |
| Views         | 1419,45 | 5      |
| Rows          | 1419,45 | 17 200 |


## 6. Физическая схема БД
### Выбор хранилища данных
- Таблица **sessions**: Можно использовать Redis(user_id), т.к. он обладает хорошей отказоустойчивостью и производительностью, а также позволяет установить TTL и удалять ненужные данные через какое-то время;
- Таблица **users**: Для этой таблицы лучше всего использовать реляционную базу данных, такую как PostgreSQL. Они обеспечивают эффективное хранение и операции с данными пользователей, такие как аутентификация и управление доступом;
- Таблица **parking_lots, rows, views**: Для информации о парковках лучше всего использовать географическую базу данных, такую как PostGIS (расширение PostgreSQL) с поддержкой геоиндексов. Это позволит эффективно хранить и выполнять пространственные запросы к парковкам, например, поиск ближайших парковок к определенной точке на карте. Для повышения производительности будем использовать Redis для хранения популярных избранных парковок; 
- Таблица **favourites**: Для хранения информации об избранных парковках пользователя также можно использовать реляционную базу данных, поддерживающую быстрый доступ к данным, такой как PostgreSQL. 
- Таблица **parking_lots_photos**: Для хранения фотографий будем использовать Amazon S3.

### Индексы
 - Таблица **users**: Индекс по email;
 - Таблица **sessions**: Индекс по id пользователя;
 - Таблица **parking_lots**: Индекс по адресу парковки;
 - Таблица **favourites**: Индекс по id парковки;.
 - Таблица **views**: Индекс по id парковки;.

### Денормализация
- Объединим таблицы **views** и **parking_lots** в таблицу parking_lots_views.<br>

### Физическая схема
![image](https://github.com/AlexnadrLarin/highload/assets/47739783/e3dcd56b-0e71-4cc8-8d7e-73a3925574d5)


## 7. Алгоритм
В сервисе используется алгоритм для распознавания парковочных мест. Он работает по следующему принципу:<br>
1. Берутся данные с камеры в виде кадров.
2. Нейросеть распознает машины, определяет их местоположение на кадре.
3. С помощью этих данных создается матрица счетчиков по размеру кадра, то есть каждому пикселю присваивается свой счётчик.
4. Если область машины попала на нужный пиксель, то счетчик этого пикселя инкрементируется. Чем больше величина счетчика, тем более вероятно, что этот пиксель входит в область парковочного места.
5. Этот алгоритм некоторое время работает, чтобы более точно распознать парковочные места. После того, как матрица счётчиков сформирована, по ней создается матрица булевых значений, которая нужна для определения занятости парковочного места.
   

## 8. Технологии
| Технология | Область применения               | Мотивация                                                                                          |
|------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| Typescript | Frontend                         | Большая популярность, поддержка яндекс карт                                                        |
| Golang     | Backend                          | Асинхронность, строгая типизация                                                                   |
| Nginx      | Балансировщик                    | Удобный, легковесный и надежный балансировщик                                                      |
| Redis      | Кеширование сессий и избранного  | Простой в обслуживании, лекговесный, есть TTL                                                      |
| PostgreSQL | DataBase                         | Популярный и надежный сервис, большой функционал, реляционная база данных                          |
| PostGis    | DataBase                         | Возможность работы с координатами                                                                  |
| Amazon S3  | Хранение фотографий пользователя | Удобный, простой в использовании, гибкая конфигурация                                              |
| Kafka      | Брокер сообщений                 | Возможность снизить нагрузку на БД, более высокая пропускная способность                           |
| Kubernetes | DevOps                           | Предоставляется автоматическая масштабируемость, отказоустойчивость и удобное управление ресурсами |


## 9. Схема проекта
![image](https://github.com/AlexnadrLarin/highload/assets/47739783/21d774fd-a5c3-4bc7-b937-f5e4ac7d1695)

| Сервис      | Область применения                                           |
|-------------|--------------------------------------------------------------|
| Auth        | Авторизация, регистрация пользователя, управление сессиями   |
| Users       | Взаимодействие с данными о зарегистрированных пользователях  |
| Favorites   | Взаимодействие с данными об избранных парковках пользователя |
| Parking_lot | Взаимодействие с данными о парковках                         |
| Analyzer    | Распознавание мест и запись в БД                             |


## 10. Надежность
| Технология          | Метод обеспечения надежности | Мотивация                                                                                       |
|---------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| nginx               | Резервирование               | Резервные сервера необходимы, чтобы при отказе одного сервера, запросы шли на другой сервер     |
| PostgreSQL, PostGIS | Репликация                   | Необходима репликация для того, чтобы при отказе не потерять данные о пользователях и парковках |
| S3                  | Репликация                   | Необходима репликация для того, чтобы при отказе не потерять аватарки                           |
| API Gateway         | Репликация                   | Это узкая часть системы, поэтому необходима репликация                                          |


Kubernetes позволяет равномерно распределять нагрузку между контейнерами, чтобы предотвратить перегрузку и обеспечить стабильность работы. В случае падения контейнеров, автоматически сработает перезапуск, также Kubernetes обеспечивает практически мгновенное горизонтальное масштабирование. 

Для БД - использование реплицирования обеспечивают отказоустойчивость при многочисленных транзакциях на чтение / запись. Добавим сервис-очередь, к примеру Kafka, который снизит нагрузку. Также необходимы резервное копирование и снапшоты для возможности восстановления.

Для ДЦ - расставить ЦОДы в нескольких городах и использовать несколько провайдеров.

Также необходимы:
- Сбор и хранение логов с возможностью трассировки для отладки и анализа инцидентов;
- Регулярное статистическое профилирование для выявления проблем производительности;
- Разделение системы на независимые сервисы, чтобы сбои разных систем не влияли друг на друга;
- Graceful shutdown, чтобы не терять данные;
- Graceful degradation, чтобы система продолжала работать даже при некоторых сбоях.


## 10. Расчёт ресурсов

Исходя из п.5 построим таблицу для микросервисов:
1/5 запросов кешируется.

| Технология | RPS |
|------------|-----|
| 1 ядро CPU | 50  |
| 1 Гб RAM   | 500 |

| Сервис      | RPS   | Трафик    | CPU | RAM  |
|-------------|-------|-----------|-----|------|
| Auth        | 250   | 4 Кб/c    | 2   | 1    |
| Users       | 254   | 4.05 Кб/c | 2   | 1    |
| Favorites   | 8000  | 7.32 Кб/c | 80  | 16   |
| Parking_lot | 8300  | 3.3 Гб/c  | 83  | 17   |
| Analyzer    | 25800 | 800 Мб/c  | 250 | 52   |

В сумме:<br>
RPS = 42604<br>
Трафик = 4.08 Гб/c

Для nginx:
8 ядер CPU выдерживает 38,900 HTTPS CPS, значит нам необходимо взять 2 таких сервера.

### Конфигурация серверов
| Сервис      | Конфигурация                         | Количество для 1 дц | 
|-------------|--------------------------------------|---------------------|
| nginx       | 8 core CPU; 4x8 GB RAM 2400 MHz DDR4 | 2                   |
| Auth        | 1 core CPU; 1x4 GB RAM 2400 MHz DDR4 | 1                   |
| Users       | 1 core CPU; 1x4 GB RAM 2400 MHz DDR4 | 1                   |
| Favorites   | 2 core CPU; 1x8 GB RAM 2400 MHz DDR4 | 2                   |
| Parking_lot | 2 core CPU; 1x8 GB RAM 2400 MHz DDR4 | 2                   |
| Analyzer    | 8 core CPU; 4x8 GB RAM 2400 MHz DDR4 | 2                   |


### Источники:
1. [Статистика Яндекс Go](https://inclient.ru/yandex-stats/#go)
2. [Статистика Яндекс Карт](https://www.retail.ru/rbc/pressreleases/prodvizhenie-biznesa-v-puti-geomediynaya-reklama-kak-instrument-marketinga-v-2023/)
3. [Информация о тайлах карты](https://yandex.ru/dev/tiles/doc/ru/)
4. [Информация о приросте аудитории Яндекс Карт](https://mindbox.ru/journal/experts/it-brand/)






