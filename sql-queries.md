# 💾 SQL-запросы для анализа данных

Этот раздел содержит **последовательность SQL-запросов**, демонстрирующих работу с базой данных психологического центра.

Запросы упорядочены **от базовых операций к более сложным аналитическим конструкциям**, включая:
- выборку данных (`SELECT`)
- фильтрацию и сортировку (`WHERE`, `ORDER BY`)
- соединение таблиц (`JOIN`)
- агрегацию (`COUNT`, `AVG`)
- подзапросы и CTE

Каждый запрос сопровождается **бизнес-задачей**, чтобы показать, как SQL помогает принимать решения в реальной системе.

---

## 1. Список всех психотерапевтов с их методом работы

**Бизнес-задача:**  
Клиент на сайте хочет выбрать специалиста по терапевтическому подходу (например, когнитивно-поведенческий, гештальт). Этот запрос нужен для отображения списка терапевтов на странице "Психотерапевты".

```sql
SELECT 
    u.last_name, 
    u.first_name, 
    u.middle_name, 
    t.method
FROM therapist t
INNER JOIN "user" u ON t.user_id = u.user_id
ORDER BY t.method, u.last_name;
```

## 2. Все клиенты старше 25 лет

**Бизнес-задача:**  
SMM-менеджер или администратор планирует целевую маркетинговую кампанию для взрослой аудитории (например, «терапия для родителей» или «работа со стрессом у профессионалов»). Этот запрос позволяет выделить клиентов старше 25 лет для сегментации и персонализированных рассылок.

```sql
SELECT 
    u.last_name, 
    u.first_name, 
    u.middle_name, 
    u.date_of_birth
FROM "user" u
WHERE u.role = 'client'
  AND EXTRACT(YEAR FROM AGE(CURRENT_DATE, u.date_of_birth)) > 25
ORDER BY u.last_name, u.first_name;
```

## 3. Актуальные бронирования: клиент, терапевт, время, форма работы и статус

**Бизнес-задача:**  
Администратору нужно быстро получить обзор всех **актуальных будущих сессий** (забронированных и оплаченных) на ближайшие 3 месяца. Этот запрос используется для мониторинга загруженности центра, контроля расписания и подготовки отчётов. Вывод включает статус бронирования, чтобы администратор мог различать "забронировано" и "оплачено".

```sql
SELECT 
    u.last_name AS client_last_name,
    u.first_name AS client_first_name,
    u.middle_name AS client_middle_name,
    u2.last_name AS therapist_last_name,
    u2.first_name AS therapist_first_name,
    u2.middle_name AS therapist_middle_name,
    s.start_time,
    st.name AS service_type,
    bs.status AS booking_status
FROM booking b
INNER JOIN "user" u ON b.client_user_id = u.user_id
INNER JOIN session s ON b.session_id = s.session_id
INNER JOIN "user" u2 ON s.therapist_user_id = u2.user_id
INNER JOIN service_type st ON s.service_type_id = st.service_type_id
INNER JOIN booking_status bs ON b.status_id = bs.status_id
WHERE bs.status IN ('booked', 'paid')
  AND s.start_time >= CURRENT_DATE
  AND s.start_time <= CURRENT_DATE + INTERVAL '3 months'
ORDER BY s.start_time, u2.last_name, u.last_name
LIMIT 1000;
```

## 4. Количество проведённых сессий по каждому психотерапевту за последний год

**Бизнес-задача:**  
Администратор хочет оценить **активность психотерапевтов** за последний год. Это позволяет:
- выявить самых востребованных специалистов,
- оценить загруженность,
- принять решения о мотивации или расписании.

```sql
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name,
    COUNT(b.booking_id) AS sessions_count
FROM booking b
INNER JOIN booking_status bs ON b.status_id = bs.status_id
INNER JOIN session s ON b.session_id = s.session_id
INNER JOIN "user" u ON s.therapist_user_id = u.user_id
WHERE bs.status = 'completed'
  AND s.start_time >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY u.user_id, u.last_name, u.first_name, u.middle_name
ORDER BY sessions_count DESC;
```

## 5. Средняя оценка каждого психотерапевта по отзывам (только за последний год, 3+ отзывов)

**Бизнес-задача:**  
Администратор хочет оценить **актуальную репутацию терапевтов** — кто получает высокие оценки **в последнее время**. Этот запрос учитывает только отзывы за последний год и включает только тех терапевтов, у кого 3 и более отзывов. Это помогает выявить специалистов, которые сейчас особенно востребованы и высоко оцениваются клиентами.

```sql
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name,
    ROUND(AVG(r.rating), 2) AS avg_rating,
    COUNT(r.review_id) AS review_count
FROM "user" u 
INNER JOIN review r ON r.therapist_user_id = u.user_id
WHERE r.rating IS NOT NULL
  AND r.created_at >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY u.user_id, u.last_name, u.first_name, u.middle_name
HAVING COUNT(r.review_id) >= 3
ORDER BY avg_rating DESC, review_count DESC;
```

## 6. Рейтинг терапевтов по доходу от оплаченных сессий (последний месяц)

**Бизнес-задача:**  
SMM-менеджер и администратор хотят выявить самых востребованных терапевтов **по доходу и по количеству сессий**. Этот запрос использует оконную функцию `ROW_NUMBER()` для ранжирования и помогает:
- формировать маркетинговые кампании ("ТОП-3 терапевта месяца"),
- анализировать финансовую эффективность,
- принимать решения по мотивации.

```sql
SELECT 
    u.last_name,
    u.first_name,
    COUNT(b.booking_id) AS paid_sessions_count,
    SUM(p.amount) AS total_revenue,
    ROW_NUMBER() OVER (ORDER BY SUM(p.amount) DESC) AS rank_num
FROM booking b
INNER JOIN booking_status bs ON b.status_id = bs.status_id
INNER JOIN session s ON b.session_id = s.session_id
INNER JOIN payment p ON b.booking_id = p.booking_id
INNER JOIN "user" u ON s.therapist_user_id = u.user_id
WHERE bs.status = 'paid'
  AND s.start_time >= CURRENT_DATE - INTERVAL '1 month'
GROUP BY u.user_id, u.last_name, u.first_name
ORDER BY total_revenue DESC;
```

## 7. Доля выручки терапевта в рамках его формы работы (проведённые сессии, последние 3 месяца)

**Бизнес-задача:**  
Администратор хочет оценить вклад каждого терапевта в доход своей ниши (индивидуальная, семейная, групповая форма работы). Этот запрос рассчитывает долю выручки терапевта от общего дохода по форме, что помогает:
- выявить ключевых специалистов,
- оценить баланс нагрузки,
- принять решения по мотивации и маркетингу.

```sql
SELECT 
    u.last_name,
    u.first_name,
    st.name AS service_type,
    COUNT(b.booking_id) AS completed_sessions_count,
    SUM(p.amount) AS total_revenue,
    ROUND(
        100.0 * SUM(p.amount) / SUM(SUM(p.amount)) OVER (PARTITION BY st.service_type_id),
        2
    ) AS share_revenue_percent
FROM booking b
INNER JOIN booking_status bs ON b.status_id = bs.status_id
INNER JOIN session s ON b.session_id = s.session_id
INNER JOIN payment p ON b.booking_id = p.booking_id
INNER JOIN "user" u ON s.therapist_user_id = u.user_id
INNER JOIN service_type st ON s.service_type_id = st.service_type_id
WHERE bs.status = 'completed'
  AND s.start_time >= CURRENT_DATE - INTERVAL '3 months'
GROUP BY u.user_id, u.last_name, u.first_name, st.service_type_id, st.name
ORDER BY st.name, share_revenue_percent DESC;
```

## 8. Рост количества отзывов у терапевта по месяцам

**Бизнес-задача:**  
Администратор хочет выявить терапевтов, чья популярность растёт — кто получает больше отзывов по сравнению с предыдущим месяцем. Это помогает в маркетинге, мотивации и анализе качества сервиса.

**Примечание:** Ниже приведены два эквивалентных подхода к решению задачи — с использованием CTE и с прямым применением оконной функции. Оба возвращают одинаковый результат, но различаются по стилю и читаемости.

---

### 🔹 Вариант 1: с CTE 

```sql
WITH monthly_reviews AS (
    SELECT 
        r.therapist_user_id,
        DATE_TRUNC('month', r.created_at) AS review_month,
        COUNT(r.review_id) AS review_count
    FROM review r
    WHERE r.created_at >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY r.therapist_user_id, DATE_TRUNC('month', r.created_at)
),
review_with_growth AS (
    SELECT 
        mr.therapist_user_id,
        mr.review_month,
        mr.review_count,
        LAG(mr.review_count) OVER (
            PARTITION BY mr.therapist_user_id 
            ORDER BY mr.review_month
        ) AS prev_month_count
    FROM monthly_reviews mr
)
SELECT 
    u.last_name,
    u.first_name,
    rwg.review_month,
    rwg.review_count,
    rwg.prev_month_count,
    (rwg.review_count - COALESCE(rwg.prev_month_count, 0)) AS growth
FROM review_with_growth rwg
INNER JOIN "user" u ON rwg.therapist_user_id = u.user_id
WHERE rwg.review_count > COALESCE(rwg.prev_month_count, 0)
ORDER BY rwg.review_month DESC, growth DESC;
```

### 🔹 Вариант 2: Только оконные функции

```sql
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name,
    DATE_TRUNC('month', r.created_at) AS review_month,
    COUNT(r.review_id) AS review_count,
    LAG(COUNT(r.review_id)) OVER (
        PARTITION BY u.user_id 
        ORDER BY DATE_TRUNC('month', r.created_at)
    ) AS prev_month_count,
    COUNT(r.review_id) - 
    LAG(COUNT(r.review_id)) OVER (
        PARTITION BY u.user_id 
        ORDER BY DATE_TRUNC('month', r.created_at)
    ) AS growth
FROM review r
INNER JOIN "user" u ON r.therapist_user_id = u.user_id
WHERE r.created_at >= CURRENT_DATE - INTERVAL '6 months'
GROUP BY u.user_id, u.last_name, u.first_name, u.middle_name, DATE_TRUNC('month', r.created_at)
HAVING COUNT(r.review_id) > 0
ORDER BY u.last_name, review_month DESC;
```
