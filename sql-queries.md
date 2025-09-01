# 💾 SQL-запросы для анализа данных

Этот раздел содержит **последовательность SQL-запросов**, демонстрирующих работу с базой данных психологического центра.

Запросы упорядочены **от базовых операций к более сложным аналитическим конструкциям**, включая:
- выборку данных (`SELECT`)
- фильтрацию и сортировку (`WHERE`, `ORDER BY`)
- соединение таблиц (`JOIN`)
- агрегацию (`COUNT`, `AVG`)
- подзапросы и CTE
- оконные функции

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
## 6. Категоризация клиентов по активности (новый, активный, постоянный)

**Бизнес-задача:**  
Администратор хочет понять, какие клиенты являются новыми, активными или постоянными, чтобы:
- мотивировать "спящих" клиентов (рассылка, скидка),
- удержать постоянных,
- адаптировать коммуникацию под сегмент.

Этот запрос использует CASE для категоризации клиентов по количеству проведённых сессий, что демонстрирует умение переводить бизнес-правила в SQL.

```sql
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name,
    COUNT(b.booking_id) AS total_sessions,
    CASE 
        WHEN COUNT(b.booking_id) = 1 THEN 'новый'
        WHEN COUNT(b.booking_id) BETWEEN 2 AND 4 THEN 'активный'
        WHEN COUNT(b.booking_id) >= 5 THEN 'постоянный'
        ELSE 'нет активности'
    END AS client_segment
FROM "user" u
LEFT JOIN booking b ON u.user_id = b.client_user_id
WHERE u.role = 'client'
GROUP BY u.user_id, u.last_name, u.first_name, u.middle_name
ORDER BY total_sessions DESC;
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

## 8. Динамика количества отзывов по месяцам (за 6 полных месяцев)

**Бизнес-задача:**  
Администратор хочет проанализировать динамику популярности терапевтов за **последние 6 полных месяцев**, исключая текущий месяц, чтобы избежать искажений из-за неполных данных. Этот запрос помогает выявить специалистов, чья популярность растёт, и принять решения по маркетингу и мотивации.

```sql
WITH review_month AS (
    SELECT 
        r.therapist_user_id,
        DATE_TRUNC('month', r.created_at)::DATE AS review_month,
        COUNT(r.review_id) AS review_count
    FROM review r
    WHERE r.created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '6 months')
      AND r.created_at < DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY r.therapist_user_id, DATE_TRUNC('month', r.created_at)
),
review_month_growth AS (
    SELECT 
        rm.therapist_user_id,
        rm.review_month,
        rm.review_count,
        LAG(rm.review_count) OVER (
            PARTITION BY rm.therapist_user_id 
            ORDER BY rm.review_month
        ) AS review_count_prev
    FROM review_month rm
)
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name,
    rmg.review_month,
    rmg.review_count,
    rmg.review_count_prev,
    rmg.review_count - COALESCE(rmg.review_count_prev, 0) AS review_growth
FROM review_month_growth rmg
INNER JOIN "user" u ON rmg.therapist_user_id = u.user_id
ORDER BY 
    review_growth DESC
    rmg.review_month DESC;
```
## 9. Терапевты, у которых нет ни одного отзыва

**Бизнес-задача:**  
Администратор хочет выявить терапевтов, которые работают, но не получают обратной связи. Это помогает инициировать процессы сбора фидбека.

```sql
SELECT 
    u.last_name,
    u.first_name,
    u.middle_name
FROM "user" u
INNER JOIN therapist t ON u.user_id = t.user_id
LEFT JOIN review r ON t.user_id = r.therapist_user_id
WHERE r.review_id IS NULL
ORDER BY u.last_name, u.first_name;
