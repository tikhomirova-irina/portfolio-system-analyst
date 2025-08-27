# 💾 SQL-запросы для анализа данных

Этот раздел содержит **обучающую последовательность SQL-запросов**, демонстрирующих работу с базой данных психологического центра.

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
INNER JOIN "user" u ON t.user_id = u.id
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
INNER JOIN "user" u ON b.client_user_id = u.id
INNER JOIN session s ON b.session_id = s.session_id
INNER JOIN "user" u2 ON s.therapist_user_id = u2.id
INNER JOIN service_type st ON s.service_type_id = st.service_type_id
INNER JOIN booking_status bs ON b.status_id = bs.status_id
WHERE bs.status IN ('booked', 'paid')
  AND s.start_time >= CURRENT_DATE
  AND s.start_time <= CURRENT_DATE + INTERVAL '3 months'
ORDER BY s.start_time ASC, u2.last_name, u.last_name
LIMIT 1000;
