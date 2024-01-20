# 3장. SQL의 조건 분기
## 8장 : UNION을 사용한 쓸데없이 긴 표현

union은 성능적인 측명에서 큰 단점을 가지고 있다. 내부적으로는 여러개의 select 구분을 실행하는 것으로 해석된다.

union시 table access full이 발생하여 읽어들이는 테이블의 크기에 따라 성능이 성현으로 증가하게 된다. 테이블의 크기가 커지게 되면 캐시 히트율도 낮아져 더욱 성능의 차이가 크게 나타난다.

```jsx
SELECT
  CASE
    WHEN score >= 90 THEN 'A'
    WHEN score >= 80 THEN 'B'
    WHEN score >= 70 THEN 'C'
    ELSE 'D'
  END AS grade
FROM students;
```

조건분기를 where 구로만 하는 사람들은 초보자다. 잘 하는 사람은 select 구 만으로 조건 분기를 한다.

sql의 성능이 좋은지 나쁜디는 반드시 실행 계획 레벨에서 판단해야한다.

union을 사용한 분기는 select 구문을 기본단위로 분기하고 있다.

반면 case 식을 사용한 분기는 문자 그래도 ‘식’을 바탕으로 하는 사고 이다. ⇒ SQL을 마스터 하는 열쇠중 하나이다.

⇒ 이런 경우 문제를 절차지향으로 해결한다면 어떤 if로 해결할 수 있을까 보다 이것을 sql의 case로는 어떻게 해결할 수 있지? 가 더 적합하다.

## 9장 집계와 조건분기

지역별 남녀 인구를 각 지역별로 여성과 남성의  인구를 구하여라

### 1. 집계 대상으로 조건분기

개선 전 :

```sql
SELECT prefecture, SUM(pop_mem) AS pop_men, SUM(pop_wom) AS pop_wom
FROM ( SELECT prefecture, pop AS pop_mem, null AS pop_wom
				FROM Population
				WHERE sex = '1'
				UNION 
			SELECT prefecture, pop AS pop_mem, null AS pop_wom
				FROM Population
				WHERE sex = '2') TMP 
GROUP BY prefecture;
```

개선 후 :

```sql
SELECT prefecture,
				sum(CASE WHEN sex ='1' THEN pop ELSE 0 END) AS pop_mem,
				sum(CASE WHEN sex ='2' THEN pop ELSE 0 END) AS pop_wom
FROM population
GROUP BY prefecture;
```

### 2. 집약 결과로 조건 분기

테이블에 emp_id team_id emp_name team이 있는 테이블이 있어 여기서 1개의 팀에 속한 사람은 속한 팀의 이름을 출력하고 팀이 2개 인 경우에는 2개를 겸무 라고 출력하고 3개 이상인 경우네는 3개 이상을 겸무 라고 출력하는 경우

개선 전 :

```sql
SELECT
  emp_name,MAX(team) AS team
	FROM Employees 
	GROUP BY emp_name
	HAVING COUNT(*) = 1 
UNION

SELECT
	  emp_name, '2개를 겸무' AS team
	FROM Employees 
	GROUP BY emp_name
	HAVING COUNT(*) = 2 
UNION
SELECT
	  emp_name,'3개 이상을 겸무' AS team
	FROM Employees 
	GROUP BY emp_name
	HAVING COUNT(*) >= 3 
```

개선 후 :

```sql
SELECT
  emp_id,
  emp_name,
  CASE
    WHEN COUNT(*) = 1 THEN team
    WHEN COUNT(*) = 2 THEN '2개를 겸무'
    WHEN COUNT(*) >= 3 '3개 이상을 겸무'
  END AS team_status
FROM
  employees
GROUP BY
  emp_name

```

## 10장 그래도 UNION을 필요한 이유

1. union을 사용할 수 밖에 없는 경우
    1. 머지 대상이 되는 select 구문들에서 사용하는 테이블이 다른 경우
2. UNION을 사용하는 것이 성능적으로 더 좋은경우

   테이블에 key, name, date_1 , flg_1, date_2 , flg_2, date_3 , flg_3 중에 flg가 T인 것들에 대한 날짜들만 출력하는 쿼리 작성하는법

    1. union

        ```sql
        SELECT
          key,
          name,
          date_1,flg_1,
          date_2,flg_2,
          date_3,flg_3
        FROM
          threeElements
        WHERE
          date_1 = '2013-11-01' AND flg_1 = 'T'
        UNION
        SELECT
          key,
          name,
          date_1,flg_1,
          date_2,flg_2,
          date_3,flg_3
        FROM
          threeElements
        WHERE
          date_2 = '2013-11-01' AND flg_2 = 'T'
        UNION
        SELECT
          key,
          name,
          date_1,flg_1,
          date_2,flg_2,
          date_3,flg_3
        FROM
          threeElements
        WHERE
          date_3 = '2013-11-01' AND flg_3 = 'T'
        ```

       a . 인덱스를 추가

        ```sql
        CREATE INDEX IDX_1 ON ThreeElements(date_1, flg_1);
        CREATE INDEX IDX_2 ON ThreeElements(date_2, flg_2);
        CREATE INDEX IDX_3 ON ThreeElements(date_3, flg_3);
        ```

    2. or ⇒ 인덱스 스캔이 사용되지 않는다

        ```sql
        SELECT
        	key, name,
          date_1, flg_1,
          date_2, flg_2,
          date_3, flg_3
        FROM threeElements
        WHERE (date_1 = '2013-11-01' AND flg_1 = 'T')
        	OR (date_2 = '2013-11-01' AND flg_2 = 'T') 
        	OR (date_3 = '2013-11-01' AND flg_3 = 'T')
        ```

    3. in

        ```sql
        SELECT
          key,
          name,
          date_1,
          flg_1,
          date_2,
          flg_2,
          date_3,
          flg_3
        FROM
          threeElements
        WHERE
          ('2013-11-01', 'T') 
        		IN ((date_1, flg_1),
        				(date_2, flg_2),
        				(date_3, flg_3));
        ```

    4. case

        ```sql
        SELECT
          key,
          name,
          date_1,flg_1,
          date_2,flg_2,
          date_3,flg_3
        FROM
          threeElements
        WHERE
          CASE
            WHEN date_1 = '2013-11-01' THEN flg_1 
            WHEN date_2 = '2013-11-01' THEN flg_2 
            WHEN date_3 = '2013-11-01' THEN flg_3 
            ELSE NULL END = 'T';
        ```


3회의 인덱스 스캔 ( union )  vs 1회의 테이블 풀스캔 (or ,in ,case)

하지만 테이블이 크고 where 조건으로 선택되는 레코드의 수가 충분히 작다면 UNION이 더 빠릅니다. 따라서 UNION을 사용하는 경우가 더 빠를 수도 있다.

## 11장 절차 지향형과 선언형

SQL 초보자는 절차지향 세계에서 살고 있다 + 구문

SQL 중급자 선언적인 세계 + 식