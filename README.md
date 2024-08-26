# Olympics
In this repository, I will be answering various questions related to the dataset using SQL (PostgreSQL). Dataset is available on Kaggle 
[link](https://www.kaggle.com/datasets/heesoo37/120-years-of-olympic-history-athletes-and-results), questions are taken from techTFQ 
[link](https://techtfq.com/blog/practice-writing-sql-queries-using-real-dataset#google_vignette=).

Skills : Subquery, common table expressions (CTE), joins, aggregats functions, window functions

```sql
-- creating tables

CREATE TABLE IF NOT EXISTS OLYMPICS_HISTORY
(
    id          INT,
    name        VARCHAR,
    sex         VARCHAR,
    age         VARCHAR,
    height      VARCHAR,
    weight      VARCHAR,
    team        VARCHAR,
    noc         VARCHAR,
    games       VARCHAR,
    year        INT,
    season      VARCHAR,
    city        VARCHAR,
    sport       VARCHAR,
    event       VARCHAR,
    medal       VARCHAR
);

CREATE TABLE IF NOT EXISTS OLYMPICS_HISTORY_NOC_REGIONS
(
    noc         VARCHAR,
    region      VARCHAR,
    notes       VARCHAR
);

-- checking data

select * from OLYMPICS_HISTORY;
select * from OLYMPICS_HISTORY_NOC_REGIONS;

-- 1. How many olympics games have been held?

select count(distinct games) as no_of_games
from olympics_history;

-- 2. List down all Olympics games held so far.

select distinct games
from olympics_history
order by 1;

-- 3. Mention the total no of nations who participated in each olympics game?

select games, count(distinct region) as countries
from olympics_history as oh join olympics_history_noc_regions as ohnc on ohnc.noc = oh.noc
group by 1
order by 1;

-- 4. Which year saw the highest and lowest no of countries participating in olympics?

with cte as (
	select games, rank() over(order by count(distinct region)) as ranking
	from olympics_history as oh join olympics_history_noc_regions as ohnc on ohnc.noc = oh.noc
	group by 1
	)
select distinct first_value(games) over() || ' - ' || first_value(ranking) over() as lowest, 
	   		    last_value(games) over() || ' - ' || last_value(ranking) over() as highest
from cte;

-- 5. Which nation has participated in all of the olympic games?

with cte as (
	select region, count(distinct games) as count_participated
	from olympics_history as oh join olympics_history_noc_regions as ohnc on ohnc.noc = oh.noc
	group by 1
	)
select region
from cte
where count_participated = (select count(distinct games) from olympics_history);

-- 6. Identify the sport which was played in all summer olympics.

select sport
from olympics_history
where season = 'Summer'
group by 1
having count(distinct games) = (select count(distinct games) from olympics_history where season = 'Summer');

-- 7. Which Sports were just played only once and in which year in the olympics?

select distinct t.*, year
from (select sport
	  from olympics_history
	  group by 1
	  having count(distinct games) = 1) as t join olympics_history as oh on t.sport = oh.sport;

-- 8. Fetch the total no of sports played in each olympic games.

select games, count(distinct sport) as no_of_sports
from olympics_history as oh
group by 1
order by 1;

-- 9. Fetch details of the oldest athletes to win a gold medal.

with cte as (
	select id, name, sex, cast((case when age = 'NA' then '0' else age end) as int) as age2, 
	       height, weight, team, games, sport, event
	from olympics_history
	where medal = 'Gold'
	)
select *
from cte
where age2 = (select max(age2) from cte);

-- 10. Find the Ratio of male and female athletes participated in all olympic games.

select round(1.00*count(case when sex = 'M' then sex end)/count(case when sex = 'F' then sex end), 2) || ' : ' || '1'
	   as male_female_ratio
from olympics_history;

-- 11. Fetch the top 5 athletes who have won the most gold medals.

select *
from (select name, team, count(medal) as golds, dense_rank() over(order by count(medal) desc) as ranking
	  from olympics_history
	  where medal = 'Gold'
	  group by 1, 2) as t
where ranking <= 5;
	
-- 12. Fetch the top 5 athletes who have won the most medals (gold/silver/bronze).

with cte as (
	select name, team, count(medal) as medals, dense_rank() over(order by count(medal) desc) as ranking
	from olympics_history
	where medal <> 'NA'
	group by 1, 2
	)
select name, team, medals
from cte
where ranking <= 5;

-- 13.Fetch the top 5 most successful countries in olympics. Success is defined by no of medals won.

select region, medals 
from (
	select region, count(medal) as medals, dense_rank() over(order by count(medal) desc) as ranking
	from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
	where medal <> 'NA'
	group by 1) as t
where ranking <= 5;

-- 14. List down total gold, silver and broze medals won by each country.

select region, count(case when medal = 'Gold' then medal end) as golds,
	   count(case when medal = 'Silver' then medal end) as silvers,
	   count(case when medal = 'Bronze' then medal end) as bronzes
from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
where medal <> 'NA'
group by 1
order by 2 desc, 3 desc, 4 desc;

-- 15. List down total gold, silver and broze medals won by each country corresponding to each olympic games.

select games, region, count(case when medal = 'Gold' then medal end) as golds,
	   count(case when medal = 'Silver' then medal end) as silvers,
	   count(case when medal = 'Bronze' then medal end) as bronzes
from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
where medal <> 'NA'
group by 1, 2
order by 1, 2;

-- 16. Identify which country won the most gold, most silver and most bronze medals in each olympic games.

with cte as (
	select games, region, count(case when medal = 'Gold' then medal end) as golds,
		   count(case when medal = 'Silver' then medal end) as silvers,
		   count(case when medal = 'Bronze' then medal end) as bronzes
	from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
	where medal <> 'NA'
	group by 1, 2
	order by 1
	),
max_medals as (
	select games, max(golds) as max_gold, max(silvers) as max_silver, max(bronzes) as max_bronze
	from cte
	group by 1
	),
m_gold as (
	select cte.games, region || ' - ' || max_gold as Golds
	from cte join max_medals as mm on cte.games = mm.games and golds = max_gold
	),
m_silver as (
	select cte.games, region || ' - ' || max_silver as Silvers
	from cte join max_medals as mm on cte.games = mm.games and silvers = max_silver
	),
m_bronze as (
	select cte.games, region || ' - ' || max_bronze as Bronze
	from cte join max_medals as mm on cte.games = mm.games and bronzes = max_bronze
	)
select m_gold.*, silvers, bronze
from m_gold, m_silver, m_bronze
where m_gold. games = m_silver.games and m_silver.games = m_bronze.games;

-- 18. Which countries have never won gold medal but have won silver/bronze medals?

select *
from (
	select region, count(case when medal = 'Gold' then medal end) as golds,
		   count(case when medal = 'Silver' then medal end) as silvers,
		   count(case when medal = 'Bronze' then medal end) as bronzes
	from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
	where medal <> 'NA'
	group by 1) as t
where golds = 0 and (bronzes > 0 or silvers > 0);

-- 19. In which Sport/event, India has won highest medals.

with cte as (
	select region, sport, count(medal) as medals_won
	from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
	where region = 'India' and medal <> 'NA'
	group by 1, 2
	)
select region, sport, medals_won
from cte
where medals_won = (select max(medals_won) from cte);

-- 20. Break down all olympic games where india won medal for Hockey and how many medals in each olympic games.

select region, sport, games, count(medal) as medals_won
from olympics_history as oh join olympics_history_noc_regions as ohnc on oh.noc = ohnc.noc
where region = 'India' and sport = 'Hockey' and medal <> 'NA'
group by 1,2,3
```
