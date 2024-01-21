# SQL Branching

## UNION을 사용한 쓸데없이 긴 표현

1. UNION을 사용한 조건 분기
    - WHERE 구만 조금씩 다른 여러 개의 SELECT 구문을 합친 실행
    - 성능적인 측면에서 큰 단점
        - 외부적으로는 하나의 SQL 구문을 실행하는 것처럼 보이지만 내부적으로는 여러 개의 SELECT 구문을 실행하는 실행 계획
        - I/O 비용 크게 증가
2. UNION을 사용한 조건 분기 예제
    - 상품 테이블

        | item_id | year | item_name | price_tax_ex | price_tax_in |
        | --- | --- | --- | --- | --- |
        | 100 | 2000 | 머그컵 | 500 | 525 |
        | 100 | 2001 | 머그컵 | 520 | 546 |
        | 100 | 2002 | 머그컵 | 600 | 630 |
        | 100 | 2003 | 머그컵 | 600 | 630 |
        | 101 | 2000 | 티스푼 | 500 | 525 |
        | 101 | 2001 | 티스푼 | 500 | 525 |
        | 101 | 2002 | 티스푼 | 500 | 525 |
        | 101 | 2003 | 티스푼 | 500 | 525 |
        - 2001년까지는 세금이 포함되지 않은 가격 표시, 2002년부터는 세금이 포함된 가격 표시
    - UNION을 사용한 조건 분기
        
        ```sql
        SELECT item_name, year, price_tax_ex AS price
        FROM items
        WHERE year <= 2001
        
        UNION ALL
        
        SELECT item_name, year, price_tax_in AS price
        FROM items
        WHERE year >= 2002;
        ```
        
        - items 테이블 2회 접근
        - 거의 똑같은 두 개의 쿼리를 두 번이나 실행
        - 데이터를 캐시에 둔다고 해도 크기가 커진다면 캐시 히트율 감소
3. WHERE 구에서 조건 분기를 하는 사람은 초보자
    - SELECT 구문에서 CASE 식을 사용한 조건 분기

        ```sql
        SELECT item_name, year,
        CASE WHEN year <= 2001 THEN price_tax_ex 
        		 WHEN year >= 2002 THEM price_tax_in END AS price 
        FROM items;
        ```

    - items 테이블 2회 접근
4. SELECT 구를 사용한 조건 분기 실행 계획
    - SQL 구문의 성능이 좋은지 나쁜지는 반드시 실행 계획 레벨에서 판단
    - WHERE 조건 분기 → 절차지향형 발상

## 집계와 조건 분기

1. 집계 대상으로 조건 분기 예제
    - 인구 테이블

        | prefecture | sex | pop |
        | --- | --- | --- |
        | 성남 | 1 | 60 |
        | 성남 | 2 | 40 |
        | 수원 | 1 | 90 |
        | 수원 | 2 | 100 |
        | 광명 | 1 | 100 |
        | 광명 | 2 | 50 |
        | 일산 | 1 | 100 |
        | 일산 | 2 | 100 |
        | 용인 | 1 | 20 |
        | 용인 | 2 | 200 |
    - UNION을 사용한 방법
        
        ```sql
        SELECT prefecture, SUM(pop_men) AS pop_men, SUM(pop_wom) AS pop_wom
        FROM (SELECT prefecture, pop AS pop_men, null AS pop_wom
        			FROM Population
        			WHERE sex = '1'
        			UNION
        			SELECT prefecture, pop AS pop_wom, null AS pop_men
        			FROM Popultaion
        			WHERE sex = '2') TMP
        GROUP BY prefecture;
        ```
        
        - Population 테이블 스캔 2회 실행
    - CASE를 사용한 방법
        
        ```sql
        SELECT prefecture,
        SUM(CASE WHEN sex = '1' THEN pop ELSE 0 END) as pop_men,
        SUM(CASE WHEN sex = '2' THEN pop ELSE 0 END) as pop_wom,
        FROM Popultaion
        GROUP BY prefecture;
        ```
        
        - Population 테이블 풀 스캔 1회 실행
2. 집약 결과로 조건 분기 예제
    - 직원 테이블

        | emd_id | team_id | emp_name | team |
        | --- | --- | --- | --- |
        | 201 | 1 | Joe | 상품기획 |
        | 201 | 2 | Joe | 개발 |
        | 201 | 3 | Joe | 영업 |
        | 202 | 2 | Jim | 개발 |
        | 203 | 3 | Carl | 영업 |
        | 204 | 1 | Bree | 상품기획 |
        | 204 | 2 | Bree | 개발 |
        | 204 | 3 | Bree | 영업 |
        | 204 | 4 | Bree | 관리 |
        | 205 | 1 | Kim | 상품기획 |
        | 205 | 2 | Kim | 개발 |
        - 소속된 팀이 1개라면 해당 직원은 팀의 이름 그대로 출력
        - 소속된 팀이 2개라면 해당 직원은 ‘2개를 겸무’ 출력
        - 소속된 팀이 3개라면 해당 직원은 ‘3개 이상을 겸무’ 출력
    - UNION을 사용한 방법
        
        ```sql
        SELECT emp_name, MAX(team) AS team
        FROM Employees
        GROUP BY emp_name
        HAVING COUNT(*) = 1
        
        UNION
        
        SELECT emp_name, '2개를 겸무' AS team
        FROM Employees
        GROUP BY emp_name
        HAVING COUNT(*) = 2
        
        UNION 
        
        SELECT emp_name, '3개 이상을 겸무' AS team
        FROM Employees
        GROU BY emp_name
        HAVING COUNT(*) >= 3;
        ```
        
        - Employees 테이블 스캔 3회 실행
    - CASE를 사용한 방법
        
        ```sql
        SELECT emp_name,
        CASE WHEN COUNT(*) == 1 THEN MAX(COUNT),
        		 WHEN COUNT(*) == 2 THEN '2개를 겸무',
        		 WHEN COUNT(*) >= 3 THEN '3개 이상을 겸무'
        FROM Employees 
        GROUP BY emp_name;
        ```
        
        - Employees 테이블 풀 스캔 1회 실행
        - GROUP BY의 HASH 연산 감소
        - 집약 결과 (COUNT(*)) 는 스칼라이므로 CASE 식의 입력으로 사용

## 그래도 UNION이 필요한 경우

1. UNION을 사용할 수밖에 없는 경우
    - SELECT 구문에서 사용하는 테이블이 다른 경우
    - 여러 개의 테이블에서 검색한 결과를 머지하는 경우
2. UNION을 사용하는 것이 성능적으로 더 좋은 경우
    - ThreeElements 테이블

        | key | name | date_1 | flg_1 | date_2 | flg_2 | date_3 | flg_3 |
        | --- | --- | --- | --- | --- | --- | --- | --- |
        | 1 | a | 2013-11-01 | T |  |  |  |  |
        | 2 | b |  |  | 2013-11-01 | T |  |  |
        | 3 | c |  |  | 2013-11-01 | F |  |  |
        | 4 | d |  |  | 2013-12-30 | T |  |  |
        | 5 | e |  |  |  |  | 2013-11-01 | T |
        | 6 | f |  |  |  |  | 2013-12-01 | F |
        - 2013-11-01을 가지고 있고 대칭되는 플래그 필드의 값이 T인 레코드 선택 가정
    - UNION을 선택한 방법
        
        ```sql
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE date_1 = '2013-11-01' AND flg_1 = 'T'
        
        UNION
        
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE date_2 = '2013-11-01' AND flg_2 = 'T'
        
        UNION
        
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE date_3 = '2013-11-01' AND flg_3 = 'T';
        ```
        
    - OR을 선택한 방법
        
        ```sql
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE (date_1 = '2013-11-01' AND flg_1 = 'T')
        	 OR (date_2 = '2013-11-01' AND flg_2 = 'T')
           OR (date_3 = '2013-11-01' AND flg_3 = 'T');
        ```
        
    - IN을 선택한 방법
        
        ```sql
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE ('2013-11-01', 'T')
        IN ((date_1, flg_1),
        		(date_2, flg_2),
        		(date_3, flg_3));
        ```
        
    - CASE을 선택한 방법
        
        ```sql
        SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
        FROM ThreeElements
        WHERE CASE WHEN date_1 = '2013-11-01' THEN flg_1,
        					 WHEN date_2 = '2013-11-01' THEN flg_2,
        					 WHEN date_3 = '2013-11-01' THEN flg_3
        					 ELSE NULL END = 'T';
        ```

3. 만약 `INSERT INTO ThreeElements VALUES (’7’, ‘g’, ‘2013-11-01’, ‘F’, null, null, ‘2013-11-01’, ‘T’)`이 추가된다면
    - CASE는 7번 데이터 미표시
        - CASE의 WHEN 구문이 단락 평가를 수행하기 때문
        - 앞에 있는 조건이 TRUE라면 거기에서 평가를 중단하고 나머지 분기의 평가를 생략
    - UNION, IN, OR은 7번 데이터 표시
        - (date_n, flg_n)의 짝들을 모두 평가하기 때문
        - UNION과 무조건 동치인 것은 IN 쿼리

## 절차 지향형과 선언형

1. 구문 기반과 식 기반
    - 구문 기반 - 절차 지향형
    - 식 기반 - 선언형