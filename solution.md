--Задание 1. Rolling retention
with a as (
select to_char(date_joined, 'YYYY-MM') as date, extract(days from u2.entry_at-u.date_joined) as diff, u2.user_id as id
from users u 
join userentry u2 
on u2.user_id = u.id 
)
-- разбиваем на даты, на когорты, вычисляем в процентном выражении количество активных пользователей с момента регистрации на платформе
select date,
count(distinct case 
	when diff>=0 then id end)*100.0/ count(distinct case 
		when diff>=0 then id end) as "Day 0",
    round(count(distinct case 
	    when diff>=1 then id end)*100.0/
    count (distinct case 
	    when diff>=0 then id end), 2) as " Day 1",
    round(count(distinct case 
	    when diff>=3 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 3",
    round(count(distinct case 
	    when diff>=7 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 7",
    round(count(distinct case 
	    when diff>=14 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 14",
    round(count(distinct case 
	    when diff>=30 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 30",
    round(count(distinct case 
	    when diff>=60 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 60",
    round(count(distinct case 
	    when diff>=90 then id end)*100.0/ 
count (distinct case 
	when diff>=0 then id end), 2) as "Day 90"
From a
group by date
-- Интерпретация результатов анализа:
Отличный старт (day_0): В день регистрации 80-90% пользователей возвращаются. Это значит, что первое впечатление от платформы хорошее и с энтузиазмом.
Большой отток на следующий день (day_1): Уже на второй день остается только в среднем 27-65%. Явная необходимость в улучшении процесса удержания заинтересованности пользователей
Стабилизация через неделю (7-14 дней): К 7-14 дню удержание держится в среднем на 10-38%. Это время, когда формируется привычка пользоваться платформой, лояльность и заинтересованность пользователей 
Долгосрочные результаты (30-90 дней): На 30-й день в среднем — 3-47%, на 90-й — 0-27%. Поскольку многие пользователи возвращаются в первые 7-14 дней, стоит ввести недедельную подписку для тех, кто тестирует платформу.Годовую подписку - для наиболее активных и лояльных в когортах 60-90

--Задание 2.Балансы пользователей
with a as (
    select
        user_id,
        -- Исходя из входящих данных, мы можем сделать вывод, что списания в transaction_type - type id: 1, 23-28.
        sum(case when type_id in (1, 23, 24, 25, 26, 27, 28) then -value end) write_offs,
        -- Исходя из входящих данных, мы можем сделать вывод, что начисления в transaction_type - все оставшиеся
        sum(case when type_id not in (1, 23, 24, 25, 26, 27, 28) then value end) accruals,
        sum(case when type_id in (1, 23, 24, 25, 26, 27, 28) then -value else value end) balance
    from "transaction" t
    group by user_id
)
-- Получаем средние значения
select
    round(avg(write_offs), 2) as write_off,
    round(avg(accruals), 2) as accruals,
    round(avg(balance), 2) as balance,
    percentile_cont(0.5) within group (order by balance) as median_balance
from a
--Интерпретация результатов анализа:
Пользователи редко пользуются платным функционалом: в среднем начисляется 306 коинов, а тратится только 70.При установке цены подписки необходимо на медианный баланс (62 коина), а не на средний, чтобы учесть типичного пользователя (большие выбросы в распределении).

--Задание 3.Активности пользователей
-Метрика 1: Сколько в среднем пользователь решает задач
with a1 as(  				  
	select 
		user_id, 
		problem_id as cnt
	from coderun
	union						 
	select 
		user_id, 
		problem_id as cnt   
	from codesubmit
),
a2 as (
	select count(*) as cnt     	
	from a1
	group by user_id
)
select round(avg(cnt), 2) as average_problems
from a2
--Вывод: Среднее количество решенных задач на пользователя 9.18.

--Метрика 2: Сколько в среднем пользователь предпринимается попыток для решения 1 задачи
with b as (
	select 
		user_id, 
		problem_id, 
		count(problem_id) as cnt1
	from codesubmit c
	group by user_id, problem_id
	union
	select 
		user_id, 
		problem_id, 
		count(problem_id) as cnt1
	from coderun c1
	group by user_id, problem_id
)
select round(avg(cnt1),2) as average_attempts
from b
--Вывод: Среднее количество попыток на 1 пользователя 5.75.

--Метрика 3: Сколько в среднем пользователь проходит тестов
with c as (
	select 
		user_id, 
		count(distinct test_id) as cnt3
	from teststart t 
	group by user_id
)
select round(avg(cnt3),2) as average_tests
from c
--Вывод: Среднее количество пройденных тестов 1.68 на 1 пользователя

--Метрика 4: Сколько в среднем пользователь делает попыток для прохождения 1 теста
with d as (
	select 
		user_id, 
		test_id, 
		count(test_id) as cnt4
	from teststart t 
	group by user_id, test_id
)
select round(avg(cnt4),2) as average_testattempts
from d
--Вывод: Среднее количество предпринятых попыток 1.26 на 1 тест

--Метрика 5: Какая доля от общего числа пользователей решала хотя бы одну задачу или начинала проходить хотя бы один тест
with e as (
	select distinct user_id
	from codesubmit
	union
	select distinct user_id
	from coderun
	union
	select distinct user_id
	from teststart
)
select round(count(*) *100.0 / (select count(*) from users),2) as regular_users
from e

WITH codecoins AS (
  SELECT 
    t.user_id,
    t.type_id,
    CASE 
      WHEN t.type_id = 23 THEN 'task'
      WHEN t.type_id = 27 THEN 'test'
      WHEN t.type_id = 24 THEN 'hint'
      WHEN t.type_id = 25 THEN 'solution'
      ELSE 'other'
    END AS purchase_type
  FROM transaction t
  WHERE t.type_id IN (23, 24, 25, 27, 28)
    AND t.user_id IS NOT NULL
),
coins_results AS (
  SELECT 
    (SELECT COUNT(DISTINCT user_id) FROM codecoins WHERE purchase_type = 'task') AS codecoins_tasks,
    (SELECT COUNT(DISTINCT user_id) FROM codecoins WHERE purchase_type = 'test') AS codecoins_tests,
    (SELECT COUNT(DISTINCT user_id) FROM codecoins WHERE purchase_type = 'hint') AS codecoins_hepls,
    (SELECT COUNT(DISTINCT user_id) FROM codecoins WHERE purchase_type = 'solution') AS codecoins_solutions,
    (SELECT COUNT(*) FROM codecoins WHERE purchase_type = 'task') AS total_tasks_purchased,
    (SELECT COUNT(*) FROM codecoins WHERE purchase_type = 'test') AS total_tests_purchased,
    (SELECT COUNT(*) FROM codecoins WHERE purchase_type = 'hint') AS total_helps_purchased,
    (SELECT COUNT(*) FROM codecoins WHERE purchase_type = 'solution') AS total_solutions_purchased,
    (SELECT COUNT(DISTINCT user_id) FROM codecoins) AS all_ways,
    (SELECT COUNT(DISTINCT user_id) FROM transaction WHERE user_id IS NOT NULL) AS users_with_any_transaction
)
SELECT 
  *,
  (SELECT COUNT(*) FROM codecoins) AS total_codecoin_transactions
FROM coins_results

--Интерпретация полученных результатов:
Акивность пользователей: 63% от общего числа пользователей решала хотя бы 1 задачу или начинала проходить хотя бы 1 тест
Как пользователи пользуются платформой:
В среднем решают ~10 задач, но только 1-2 теста — задачи явно популярнее.
3.17 попытки на задачу показывают: контент сложный, требует практики.
Что покупают чаще:
Тесты лидируют (676 юзеров, 989 покупок).
Задачи вторые (522 юзера, 1675 покупок).
Решения реже (151 юзер, 423 покупки).
Подсказки совсем мало (53 юзера, 118 покупок).
Что включить в подписку:
Бесплатно: Основные задачи (главный магнит для пользователей).
В подписке:
Премиум-тесты (самый ходовой платный контент).Решения задач (ускоряет обучение). Больше попыток на задачу (сейчас ~3). Подсказки (дополнение, хоть и не топ).Лимит для free: 2-3 попытки на задачу бесплатно, потом — подписка.
