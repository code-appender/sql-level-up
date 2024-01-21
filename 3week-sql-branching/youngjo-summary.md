# 3장. SQL의 조건 분기

## UNION을 사용한 쓸데없이 긴 표현

---

### 초보자들이 자주 사용하는 UNION을 사용한 조건 분기
초보자들이 사용하는 UNION -> WHERE 구만 조금씩 다른 여러 개의 SELECT 구문의 결합 = 절차 지향형
But, 이러한 방법은 성능이 매우 좋지 않다!
외부적으로는 하나의 SQL 구문을 실행하는 것처럼 보임 -> 내부에서 여러번의 SELECT 구문 실행함 -> 테이블 접근 횟수 증가 -> I/O 비용 증가
UNION을 사용해야할 시점을 알고 사용하자. (아무때나 사용 X)

- UNION을 사용한 조건 분기 예시
```sql
SELECT item_name, year, price_tax_ex AS price
  FROM Items
 WHERE year <= 2001
UNION ALL
SELECT item_name, year, price_tax_in AS price
  FROM Items
 WHERE year >= 2002;
```
이렇게 사용할 경우 실행 계획에서 테이블 풀 스캔이 N번 일어남.

- SELECT 구를 사용한 조건 분기
```sql
SELECT item_name, year,
       CASE WHEN year <= 2001 THEN price_tax_ex
            WHEN year >= 2002 THEN price_tax_in END AS price
  FROM Items;
```
해당 방법으로 실행 계획을 확인하면 테이블 풀 스캔이 1회 발생함.

=> I/O 비용 N배 감소

구문을 기본 단위로 분기하는 방법 -> UNION -> 절차지향형
식을 기본 단위로 분기하는 방법 -> CASE -> 선언형

## 집계와 조건 분기

---
### 집계 대상으로 조건 분기
- UNION
```sql
SELECT prefecture, SUM(pop_men) AS pop_men, SUM(pop_wom) AS pop_wom
  FROM (SELECT prefecture, pop AS pop_men, null AS pop_wom
          FROM Population
         WHERE sex = '1'
         UNION 
        SELECT prefecture, NULL AS pop_men, pop AS pop_wom
          FROM Population
         WHERE sex = '2') TMP
 GROUP BY prefecture;
```
이러한 방식의 문제점 = WHERE 구에서 sex 필드로 분기한 후, UNION으로 머지하는 절차지향적 구성

- CASE
```sql
SELECT prefecture,
       SUM(CASE WHEN sex = '1' THEN pop ELSE 0 END) AS pop_men,
       SUM(CASE WHEN sex = '2' THEN pop ELSE 0 END) AS pop_wom
  FROM Population
 GROUP BY prefecture;
```
해당 방법 사용시 쿼리의 간결화 + I/O 비용 감소 = 성능 향상

### 집약 결과로 조건 분기
- UNION
```sql
SELECT emp_name,
       MAX(team) AS team
  FROM Employees
 GROUP BY emp_name
HAVING COUNT(*) = 1
UNION
SELECT emp_name,
       '2개를 겸무' AS team
FROM Employees
GROUP BY emp_name
HAVING COUNT(*) = 2
UNION
SELECT emp_name,
       '3개 이상을 겸무' AS team
FROM Employees
GROUP BY emp_name
HAVING COUNT(*) >= 3;
```
- CASE
```sql
SELECT emp_name,
       CASE WHEN COUNT(*) = 1 THEN MAX(team)
            WHEN COUNT(*) = 2 THEN '2개를 겸무'
            WHEN COUNT(*) >= 3 THEN '3개 이상을 겸무'
        END AS team
  FROM Employees
 GROUP BY emp_name;
```

## 그래도 UNION이 필요한 경우

---
UNION을 사용하는 것은 모든 것이 나쁘다? -> 때에 따라 다르다. 오히려 성능이 좋을 때도 있다.
### 1. UNION을 사용할 수 밖에 없는 경우
머지 대상이 되는 SELECT 구문ㄴ들에서 사용하는 테이블이 다른 경우

### 2. UNION을 사용하는 것이 성능적으로 더 좋은 경우
인덱스를 사용하여 압축을 잘 했을 경우 (n회의 테이블 풀 스캔 vs 1회의 인덱스 스캔)