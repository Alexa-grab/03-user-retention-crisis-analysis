#### Учебный проект

от Karpov.Courses [Karpov.Courses](https://karpov.courses/simulator)

# Анализ двух ключевых событий приложения.

Проект анализа ключевых событий приложения в Apache Superset.

## Вводные данные

Социальная сеть с двумя сервисами:

 - **Лента новостей — просмотр и лайки постов**
 - **Мессенджер — обмен сообщениями**

Все события хранятся в ClickHouse:

 - **feed_actions — действия в ленте (просмотры, лайки)**
 - **message_actions — отправка сообщений**

Произошедшие события:

  - **Маркетинговая компания для привлечения новых пользовталей**
  - **Резкое падение аудитории**

## Задачи

   - Проанализировать характер Retention пользователей, привлечённых рекламной кампанией.
   - Что стало с рекламными пользователями в дальнейшем, как часто они продолжают пользоваться приложением?
   - Выяснить, какие пользователи не смогли воспользоваться лентой. Что их объединяет?

## Инструменты

- **База данных:** ClickHouse
- **BI-инструменты:** Superset, Redash

**Структура таблицы БД**

**1. Feed_actions**

   - **post_id:** id поста;
   - **user_id:** id пользователя;
   - **action:** совершенное действие view/like;
   - **time:** время совершенного действия;
   - **gender:** пол пользователя;
   - **city:** город пользователя;
   - **country:** страна пользователя;
   - **os:** тип операционной системы;
   - **source:** источник входа(органический/рекламный).

## 1. Анализ аудитории: общие данные
![Анализ аудитории:общие данные](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D0%BE%D0%B1%D1%89%D0%B81.png)

- #### DAU
  
  **Выводы по графику**
   - Маркетинговая компания была проведена 15.08.25
   - Резкое падение аудитории произошло 24.08.25


- #### Динамика удержания пользователей (Retention) по неделям

  **Выводы по графику** 
    График удержания демонстрирует позитивную динамику: в течение первых двух месяцев наблюдается уверенный рост лояльности пользователей, после чего метрика стабилизировалась, выйдя на устойчивое плато.  
    Это указывает на то, что продукт или маркетинговая кампания успешно сформировали стабильное ядро лояльной аудитории.
    
  **Код SQL**    
```
-- Анализ еженедельного Retention: новых, вернувшихся и ушедших пользователей

-- Анализ УШЕДШИХ пользователей (Churn)

Select this_week,
previous_week,
-uniq(user_id) as count_users,-- Отрицательное значение для визуализации оттока
status 
From

(SELECT user_id ,
groupUniqArray(toMonday(toDate(time))) as weeks_visited, -- Массив всех недель посещения для каждого пользователя
arrayJoin(weeks_visited) as previous_week, -- Разворачиваем массив в строки (каждая неделя как отдельная запись)
addWeeks(previous_week,+1) as this_week, -- Следующая неделя после посещения
'gone' as status -- Статус "ушел"
From simulator_20250720.feed_actions 
Group by user_id
)
where NOT has(weeks_visited, this_week)  -- Фильтр: пользователь НЕ был активен на следующей неделе
  AND this_week <= toMonday(today())     -- Только завершенные недели (исключаем текущую)
Group by this_week,previous_week,status

UNION ALL -- Объединяем с результатом анализа новых и вернувшихся пользователей

-- Анализ НОВЫХ и ВЕРНУВШИХСЯ пользователей

Select this_week,
previous_week,
toInt64(uniq(user_id)) as count_users, -- Положительное значение для прироста
status 
From

(SELECT user_id ,
groupUniqArray(toMonday(toDate(time))) as weeks_visited,
arrayJoin(weeks_visited) AS this_week,   -- Текущая неделя активности
addWeeks(this_week,-1) as previous_week, -- Предыдущая неделя
IF  -- Проверяем был ли пользователь активен на предыдущей неделе
(
has(weeks_visited,addWeeks(this_week,-1)),
'retained', -- Если ДА - "вернувшийся"
'new')      -- Если НЕТ - "новый"
as status
From simulator_20250720.feed_actions 
Group by user_id
)

Group by this_week,previous_week,status
```
## 2. Анализ маркетинговой компании 

![Аудиторные данные](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D1%80%D0%B5%D0%BA%D0%BB%D0%B0%D0%BC%D0%BD%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B0%D0%BD%D0%B8%D1%8F%201.png)

- #### DAU за период с 12.08.25 по 19.09.25

**Выводы по графику**  
  Маркетинговая кампания вызвала явный всплеск активности (DAU), достигнув пика 15 августа.

- #### Пользователи привлеченные рекламной компанией

**Выводы по графику**

Маркетинговая кампания дала значительный прирост: в день пика мы зафиксировали 8370 пользователей с рекламного трафика — на 38% больше, чем неделей ранее.  
Из них 2592 были новыми пользователями.

- #### Ретеншен 1го дня и последущая активность новых пользователей

![heatmap](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D1%80%D0%B5%D0%BA%D0%BB%D0%B0%D0%BC%D0%BD%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B0%D0%BD%D0%B8%D1%8F%203.png)
![Ретеншен 1го дня](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D1%80%D0%B5%D0%BA%D0%BB%D0%B0%D0%BC%D0%BD%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B0%D0%BD%D0%B8%D1%8F%207.png)

**Выводы по графикам** 

Динамика удержания когорты от 15.08 тревожная: начавшись с крайне низкого показателя в 3.97%, Retention продолжил снижаться и стабилизировался на отметке всего 1.31%.


**SQL запросы к графикам**

 - **heatmap retantion**
```
-- heatmap retantion
Select
toString(date) as date,
toString(start_date) as start_date,
count(user_id) as active_users ,
source
From

(SELECT
user_id,
min(toDate(time)) as start_date,
source 
From simulator_20250820.feed_actions
WHERE source ='ads'
group by user_id,source
having start_date between '2025-08-12' and '2025-08-19'
) t1

join

(select
 distinct user_id,
toDate(time) as date 
From simulator_20250820.feed_actions ) t2

using user_id

group by date,start_date,source

UNION ALL

Select
toString(date) as date,
toString(start_date) as start_date,
count(user_id) as active_users ,
source

From
(SELECT
user_id,
min(toDate(time)) as start_date,
source 
From simulator_20250820.feed_actions
WHERE source ='organic'
group by user_id,source
having start_date >= '2025-08-15'
) t1

join

(select
 distinct user_id,
 toDate(time) as date 
From simulator_20250820.feed_actions ) t2

using user_id

group by date,start_date,source
```

 - **Retantion 1го дня и дальнейшее поведение когорты**
   
```
SELECT 
    date,
    count(user_id) as num_users,
    source,
    (num_users / first_day_users) as percentage_of_first_day
FROM 
(
    SELECT 
        t2.date,
        t2.source,
        t1.start_day,
        t1.user_id,
        -- Получаем количество пользователей в первый день для каждой группы
        countIf(t1.user_id, t2.date = t1.start_day) OVER (PARTITION BY t2.source) as first_day_users
    FROM 
    (
        SELECT 
            user_id,
            min(toDate(time)) as start_day 
        FROM simulator_20250820.feed_actions 
        GROUP BY user_id 
        HAVING min(toDate(time)) = '2025-08-15'
    ) t1
    JOIN 
    (
        SELECT DISTINCT 
            user_id, 
            toDate(time) as date,
            source 
        FROM simulator_20250820.feed_actions 
    ) t2 USING (user_id)
)
GROUP BY date, source, start_day, first_day_users
ORDER BY date, source
```
 #### Активность привлеченной аудитории

![Активность привлеченной аудитории](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D1%80%D0%B5%D0%BA%D0%BB%D0%B0%D0%BC%D0%BD%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B0%D0%BD%D0%B8%D1%8F%206.png)

**2.5. Лайки и просмотры**
 - Выводы
 - 
**2.6. CTR**
- Выводы
- 
#### Аудитория**

![Аудитория](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D1%80%D0%B5%D0%BA%D0%BB%D0%B0%D0%BC%D0%BD%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B0%D0%BD%D0%B8%D1%8F%205.png)

**2.7. Пол/возраст/операционная система**

- ВЫВОДЫ
```
SELECT 
    toDate(time) AS day,
    user_id,
    CASE
        WHEN gender = 1 THEN 'Woman'
        WHEN gender = 0 THEN 'Man'
        ELSE 'Unknown'
    END AS gender_text,
    CASE 
        WHEN age <= 17 THEN 'До 17' 
        WHEN age > 17 AND age <= 24 THEN '18-24 лет' 
        WHEN age > 24 AND age <= 34 THEN '25-34 лет' 
        WHEN age > 34 AND age <= 44 THEN '35-44 лет' 
        WHEN age > 44 AND age <= 54 THEN '45-54 лет' 
        WHEN age > 54 THEN '55+' 
        ELSE 'Не указан'
    END AS age_group,
    action,
    source,
    city,
    os
FROM simulator_20250820.feed_actions 
Where  user_id IN (Select user_id from simulator_20250820.feed_actions 
GROUP by user_id 
having min(toDate(time))='2025-08-15')
```

**2.8. Топ - 5 самых популярных городов**

- SQL запрос

```
SELECT 
    city,
    count(distinct user_id) as num_users_city,
    source
FROM simulator_20250820.feed_actions 
WHERE toDate(time) = '2025-08-15' and source='ads'
AND user_id IN (
    SELECT user_id 
    FROM simulator_20250820.feed_actions 
    GROUP BY user_id 
    HAVING min(toDate(time)) = '2025-08-15'
)
GROUP BY city,source
ORDER BY num_users_city DESC
limit 5
UNION All

SELECT 
    city,
    count(distinct user_id) as num_users_city,
    source
FROM simulator_20250820.feed_actions 
WHERE toDate(time) = '2025-08-15' and source='organic'
AND user_id IN (
    SELECT user_id 
    FROM simulator_20250820.feed_actions 
    GROUP BY user_id 
    HAVING min(toDate(time)) = '2025-08-15'
)
GROUP BY city,source
ORDER BY num_users_city DESC
limit 5
```
### Результаты анализа маркетинговой компании от 15/08/25

 По итогу было привлечено 2592 человека (это 3.3% от числа всех пользователей, по состоянию на 15.08).
 Основная аудитория – это женщины 18-24 лет из Москвы и СПБ.
 Основные наблюдения:
 - Retention
  •	Retention  1 дня составил 3,97%.
  •	Через 4 недели, охват привлеченных пользователей упал до  1% . 
- Активность
  •	Лайки и просмотры упали после первого дня использования и был небольшой всплеск активности 24.08.25 (когда произошел сбой системы).
  •	График CTR имеет пилообразную форму, аномалий не замечено
  Можно сделать вывод, что маркетинговая кампания была успешной, однако существуют сложности с удержанием привлечённых пользователей.

  ## 3. Падение аудитории от 24.08.25
  
 ![DAU](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D0%BF%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%205.png)

 **3.1 Анализ метрики DAU за период с 19/08/25 по 26/08/25**
 - ВЫВОД

**3.2. Основные события за 24.08.25**

![Основыне события](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D0%BF%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%206.png)

 - СTR с интервалом 30 мин;
 - Все события с интервалом 30 мин;
 - Лайки и просмотры с интервалом 30 мин.

- ВЫВОДЫ

  **3.3. Пользователи,которые не смогли воспользоваться приложением 24.08.25**
  
  **География активностей**
  
  ![города](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D0%BF%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%207%20%D0%B3%D0%BE%D1%80%D0%BE%D0%B4%D0%B0.png)

-ВЫВОД
-SQL код

```
Select date,city,count_users from 
(
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-19' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-20' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-21' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-22' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-23' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-24' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-25' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
union all
SELECT toDate(time) as date,city,Count(user_id) as count_users From simulator_20250820.feed_actions 
WHERE toDate(time)='2025-08-26' 
Group by toDate(time),city
order by count_users desc,date 
limit 5
)
order by date
  ```
**3.4.Активность по источнику входа и операционной системе**

![ос и источник входа](https://github.com/Alexa-grab/desktop-tutorial/blob/main/%D0%BF%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%209%20.png)

- ВЫВОДЫ
### Результаты анализа падения аудитории 24.08.25  
  
  Основные наблюдения:
  
  - 24.08.25 зашло на 2122 пользователя меньше , по сравнению с предыдущим днем.  
    Это не типичная ситуация(на прошлой неделе было равномерное распределение)
  - Отток произошел по обоим каналам равномерно (органическому и рекламному)
  - CTR начал падать с 03.00 AM и восстановился только к 10.30 PM
  - В 4.00 AM меньше всего активности аудитории, но это типично для этого времени.
  - Отсутствуют пользователи из топ-5 самых активных городов:  
    Москвы,Санкт-Петербурга,Новосибирска и Екатеринбурга.
  - Проблемы со стороны os :
         - Вход с os andriod упал на 502К событий.
         - Вход  с os IOS упал на 273К событий.

Скорее всего, произошёл масштабный технический сбой на стороне вашего сервиса, который длился с примерно 03:00 AM до 22:30 PM.  
Еще один момент, есть взаимосвязь между людьми пришедшими 15.08.25 по маркетинговой компании и днем массового сбоя системы 24.08.25.  
Эти пользователи проявили наибольшую активность именно в этот день.  
Возможно, это связано с тем, что столкнувшись с ошибкой входа, они начали проявлять  большую активность чем другие пользователи
и пытались войти по рекламной ссылке.


 
