# highload VK

Курсовая работа в рамках дисциплины "Проектирование высоконагруженных сервисов"<br> по теме: 
"Сервис по поиску свободных парковочных мест".

**Автор** - Ларин Александр WEB-31

## Содержание
  [1. Тема, функционал и аудитория](#раздел-1)
## 1. Тема, функционал и аудитория

### Тема
В качестве темы был выбран личный проект - сервис, который помогает водителям найти <br> ближайшее свободное парковочное место и сэкономить время.

### Ключевой функционал продукта
- Отображение парковок на карте
- Просмотр количества свободных мест на выбранной парковке
- Добавление/удаление избранных парковок

### Целевая аудитория
Проект взят, как теоретический, поэтому продуктовые метрики взяты у Яндекс Навигатора, исходя из <br> того, что многие водители и пассажиры пользуются именно ими.

- 24 млн. активных пользователей в месяц
- 59% аудитории - в возрасте от 25 до 44 лет
- 35% - женщины
- 65% - мужчины
- 66% с доходом выше среднего*

Также для статистики взяты данные Яндекс Go:
Количество активных пользователей в месяц - 43,5 млн.
Количество водителей - около 170 000.

## 2. Расчет нагрузки
- 24 млн. активных пользователей в месяц.
- дневной охват может варьироваться от 2,4 до 7,2 миллиона пользователей.
- Количество парковок всего - 344 150

### Рассмотрим основные типы данных:
- Информация о парковке занимает ~1 Кб
- Информация о пользователе занимает ~1Кб
- Общий объем, который занимает карта равняется 200Тб
- Объем карты для одного города равен примерно 200Мб
- При разделении карты города на секции, объем памяти условной секции равен 192Кб

### Общий объем памяти за месяц равен:
200Тб + 24 * 10^6 * 1Кб / 1024 / 1024 + 344 150  * 1Кб / 1024 / 1024 = 222.94 Тб

### Среднее количество действий пользователя:
| Действие | Количество действий(в среднем) |
|---------------------|---------------------|
| Авторизация | 2 раза в год |
| Просмотр мест на парковке | 6 раз в день |
| Просмотр избранного | 2 раза в день |
| Добавление в избранное | 1 раз в месяц |

В среднем пользователь просматривает карту 2 - 3 раза в день.<br>
В среднем пользователь при просмотре карты открывает 3 секции. <br>
Удаление парковки из избранного и регистрация рассматриваться не будут, т.к. это происходит крайне редко.

### RPS
DAU = 7.2 M <br>
RPS = Количесво действий на пользователя * DAU / 86 400 <br>
RPD = Количесво действий на пользователя * DAU <br>
| Действие | RPS | RPD |
|---------------------|---------------------|---------------------|
| Авторизация | 0.46 | 39452.05 |
| Просмотр секций карты | 100 | 8 640 000 |
| Просмотр мест на парковке | 500 | 43 200 000 |
| Просмотр избранного | 166.67 | 14 400  000 |
| Добавление в избранное | 2.78 | 240 000 |
RPS просмотра секций карты равен 100, исходя из максимума в источнике 3.

**RPS(общий)** = 769.91<br>
**RPS(пиковый)** = 1154.865<br>

### Сетевой трафик
| Действие | Трафик | Пиковый трафик |
|---------------------|---------------------|---------------------|
| Авторизация | 0.46 Кб/c | 0.92 Кб/c |
| Просмотр секций карты | 18.75 Мб/c | 37.5 Мб/c |
| Просмотр мест на парковке | 500 Кб/c| 1 000 Кб/c|
| Просмотр избранного | 166.67 Кб/c | 333.34 Кб/c |
| Добавление в избранное | 2.78 Кб/c | 5.56 Кб/c |


## 3. Глобальная балансировка

Целевой аудиторией сервиса являются пользователи России, поэтому необходимо проанализировать плотность населения, как на рисунке ниже, и оптимально расположить ЦОДы:
![image](https://github.com/AlexnadrLarin/highload/assets/47739783/a15de64e-d8fc-4ef6-b59a-b36e5baefc07)

Большая часть пользователей находиться в Европейской части России, поэтому оптимально расположить датацентры именно там:<br>
- Москва: Как столица и крупнейший город России, Москва является центром технологического развития и имеет крупные узлы связи.
- Санкт-Петербург: Второй по величине город в России, он также имеет развитую инфраструктуру и крупные центры данных.
- Екатеринбург: Расположенный в центре России, он может обеспечить хорошее покрытие для регионов Урала и Сибири.

Использую GeoDNS определяется физически ближайший к пользователю ЦОД. Далее с помощью BGP Anycast найти для каждого района свой ЦОД, чтоб улучшить балансировку.

## 4. Локальная балансировка

Для картографического сервиса предлагаю использовать следующие технологии:<br><br>
**1. Kubernetes**<br>
Kubernetes обеспечивает масштабируемость, отказоустойчивость и управление ресурсами, что важно для сервисов, <br> которые работают с картами, особенно при обработке больших объемов запросов.

Причины выбора Kubernetes:<br>
**- Масштабируемость:** Kubernetes позволяет легко масштабировать сервисы горизонтально и вертикально в зависимости от нагрузки. Это полезно для данного сервиса, который может иметь пиковые нагрузки в определенные периоды времени.

**- Управление ресурсами:** Kubernetes позволяет управлять ресурсами контейнеров, что позволяет оптимизировать использование вычислительных ресурсов и обеспечить стабильную производительность сервиса.

**- Отказоустойчивость:** Kubernetes автоматически управляет перезапуском контейнеров в случае их сбоев.

**- Управление конфигурацией и развертыванием:** Kubernetes предоставляет инструменты для управления конфигурацией и автоматизации развертывания, что упрощает процесс управления и обновлениями сервиса.

**2. Nginx на уровне L7:** <br> 
Nginx будет использоваться для балансировки нагрузки между контейнерами Kubernetes.

Причины выбора Nginx:<br>
**- Высокая производительность:** Nginx известен своей высокой производительностью и эффективностью в обработке большого количества одновременных соединений и запросов. 

**- Оптимизация под медленные клиентские соединения:** Nginx имеет встроенные механизмы для оптимизации работы с медленными клиентами. Например, он может настроить таймауты соединения, управлять буферизацией ответов и настроить параллельные обработчики соединений для улучшения производительности.

Также можно использовать Weighted Least Connections для того, чтобы распределять запросы на сервисы.

**3.SSL termination:**: <br>
Для шифрования и быстрого автоматического получения SSL-сертификатов будем использовать Let`s Encrypt. Также, чтобы сэкономить время для установки соединения, можно кешировать сессию - использовать Session Cache.

### Источники:
1. [Статистика Яндекс Go](https://inclient.ru/yandex-stats/#go)<br>
2. [Статистика Яндекс Карт](https://www.retail.ru/rbc/pressreleases/prodvizhenie-biznesa-v-puti-geomediynaya-reklama-kak-instrument-marketinga-v-2023/)
3. [Информация о тайлах карты](https://yandex.ru/dev/tiles/doc/ru/)






