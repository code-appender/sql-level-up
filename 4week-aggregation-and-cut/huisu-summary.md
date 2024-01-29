## 집약과 자르기

### 집약

1. 여러 개의 레코드를 한 개의 레코드로 집약
    - 집약 함수: 여러 개의 레코드를 한 개의 레코드로 집약하는 함수
    - COUNT, SUM, AVG, MAX, MIN
2. 집약 함수 예제 - CASE 식과 GROUP BY 응용

    | id | data_type | data_1 | data_2 | data_3 | data_4 | data_5 | data_6 |
    | --- | --- | --- | --- | --- | --- | --- | --- |
    | Jim | A | 100 | 10 | 34 | 346 | 54 |  |
    | Jim | B | 45 | 2 | 167 | 77 | 90 | 157 |
    | Jim | C |  | 3 | 687 | 1355 | 324 | 457 |
    | Ken | A | 78 | 5 | 724 | 457 |  | 1 |
    | Ken | B | 123 | 12 | 178 | 436 | 85 | 235 |
    | Ken | C | 45 |  | 23 | 46 | 687 | 33 |
    | Beth | A | 75 | 0 | 190 | 25 | 356 |  |
    | Beth | B | 435 | 0 | 183 |  | 4 | 325 |
    | Beth | C | 96 | 128 |  | 0 | 0 | 12 |
    - GROUP BY로 집약할 수 있는 것
        - 상수
        - GROUP BY 구에서 사용한 집약 키
        - 집약 함수
    - 집합이 아닌 요소를 대상으로 해서 오류가 나는 쿼리
        
        ```sql
        SELECT id,
        CASE WHEN data_type ='A' THEN data_1 ELSE NULL END) AS data_1,
        CASE WHEN data_type ='A' THEN data_2 ELSE NULL END) AS data_2,
        CASE WHEN data_type ='B' THEN data_3 ELSE NULL END) AS data_3,
        CASE WHEN data_type ='B' THEN data_4 ELSE NULL END) AS data_4,
        CASE WHEN data_type ='C' THEN data_5 ELSE NULL END) AS data_5,
        CASE WHEN data_type ='C' THEN data_6 ELSE NULL END) AS data_6
        FROM NonAggTbl
        GROUP BY id;
        ```
        
    - 집합론의 원리를 통해 집약 함수를 사용한 올바른 쿼리
        
        ```sql
        SELECT id,
        MAX(CASE WHEN data_type ='A' THEN data_1 ELSE NULL END) AS data_1,
        MAX(CASE WHEN data_type ='A' THEN data_2 ELSE NULL END) AS data_2,
        MAX(CASE WHEN data_type ='B' THEN data_3 ELSE NULL END) AS data_3,
        MAX(CASE WHEN data_type ='B' THEN data_4 ELSE NULL END) AS data_4,
        MAX(CASE WHEN data_type ='C' THEN data_5 ELSE NULL END) AS data_5,
        MAX(CASE WHEN data_type ='C' THEN data_6 ELSE NULL END) AS data_6
        FROM NonAggTbl
        GROUP BY id;
        ```

3. 집약, 해시, 정렬
    - 집약할 때는 정렬 혹은 해시를 사용
    - 최근에는 정렬보다 해시 자주 사용
        - GROUP BY 구에 지정되어 있는 필드를 해시 함수를 사용해 해시 키로 변환하고, 같은 해시 키를 가진 그룹을 모아 집약
        - 해시의 특성상 GROUP BY의 유일성이 높으면 더 효율적
    - 해시와 정렬 모두 메모리를 많이 사용하기에 충분한 워킹 메모리 확보 필요
    - TEMP 탈락
        - 정렬 또는 해시를 위해 PGA라는 메모리 사용
        - PGA 크기가 집약 대상 데이터양에 비해 부족하면 일시 저장소를 사용해 부족한 만큼 채우는 현상
        - 극단적인 성능 하락
        - 메모리와 저장소 접근 속도 차이가 많이 나기 때문
        - 최악의 경우 TEMP 영역을 모두 써 버려 SQL 구문이 비정상적으로 종료될 가능성 존재
4. 합쳐서 하나

    | reserve_id | low_age | high_age | price |
    | --- | --- | --- | --- |
    | 제품1 | 0 | 50 | 2000 |
    | 제품1 | 51 | 100 | 3000 |
    | 제품2 | 0 | 100 | 4200 |
    | 제품2 | 0 | 20 | 500 |
    | 제품3 | 31 | 70 | 800 |
    | 제품3 | 71 | 100 | 1000 |
    | 제품4 | 0 | 99 | 8900 |
    - 해당 테이블은 (reserve_id, low_age) 혹은 (reserve_id, high_age) 키로 레코드가 유일하게 정해짐
    - 0~100 세까지 모든 연령이 가지고 놀 수 있는 제품을 구하는 코드
    
    ```sql
    SELECT reserve_id
    FROM PriceByAge
    GROUP BY reserve_id
    HAVING SUM (high_age - low_age + 1) = 101;
    ```


### 자르기

1. 자르기와 파티션

    | name | age | height | weight |
    | --- | --- | --- | --- |
    | Anderson | 30 | 188 | 90 |
    | Adela | 21 | 167 | 55 |
    | Bates | 87 | 158 | 48 |
    | Becky | 54 | 187 | 70 |
    | Bill | 39 | 177 | 120 |
    | Chris | 90 | 175 | 48 |
    | Darwin | 12 | 160 | 55 |
    | Dawson | 25 | 182 | 90 |
    | Donald | 30 | 176 | 53 |
    - 첫 문자 알파벳마다 몇 명의 사람이 존재하는지 계산
        
        ```sql
        SELECT SUBSTRING(name, 1, 1) AS label, COUNT(*)
        FROM Persons
        GROUP BY SUBSTRING(name, 1, 1);
        ```
        
    - 나이에 따라 어린이, 성인, 노인으로 자르기
        
        ```sql
        SELECT CASE WHEN age < 20 THEN '어린이'
        			 CASE WHEN age BETWEEN 20 AND 69 THEN '성인'
        			 CASE WHEN age >= 70 THEN '노인'
        			 ELSE NULL END AS age_class,
        			 COUNT(*)
        FROM Persons
        GROUP BY CASE WHEN age < 20 THEN '어린이'
        				 CASE WHEN age BETWEEN 20 AND 69 THEN '성인'
        				 CASE WHEN age >= 70 THEN '노인'
        				 ELSE NULL END;
        ```
        
        - 자르기의 기준이 되는 키를 GROUP BY와 SELECT 구에 모두 입력하는 것이 포인트
        - GROUP BY 구에서 CASE 식 또는 함수를 사용해도 실행 계획에는 크게 영향 가지 않음
    - BMI로 자르기
        - 18.5 미만을 저체중, 18.5 이상을 정상, 25 이상을 과체중으로 표현
        
        ```sql
        SELECT CASE WHEN weight / POWER(height / 100, 2) < 18.5 THEN '저체중'
        			 CASE WHEN 18.5 <= weight / POWER(height / 100, 2) 
        								 AND weight / POWER(height / 100, 2) < 25 THEN '정상'
        		   CASE WHEN weight / POWER(height / 100, 2) >= 25 THEN '과체중'
        			 ELSE NULL END AS bmi,
        			 COUNT(*)
        FROM Persons
        GROUP BY CASE WHEN weight / POWER(height / 100, 2) < 18.5 THEN '저체중'
        			 CASE WHEN 18.5 <= weight / POWER(height / 100, 2) 
        								 AND weight / POWER(height / 100, 2) < 25 THEN '정상'
        		   CASE WHEN weight / POWER(height / 100, 2) >= 25 THEN '과체중'
        			 ELSE NULL END;
        ```

2. PARTITION BY 구로 자르기
    - GROUP BY 구에서 집약 기능을 제오하고 자르는 기능만 남긴 것이 PARTITION BY

    ```sql
    SELECT name, age,
    			 CASE WHEN age < 20 THEN '어린이'
    			 CASE WHEN age BETWEEN 20 AND 69 THEN '성인'
    			 CASE WHEN age >= 70 THEN '노인'
    			 ELSE NULL END AS age_class
    RANK() OVER(PARTITION BY CASE WHEN age < 20 THEN '어린이'
    												 CASE WHEN age BETWEEN 20 AND 69 THEN '성인'
    												 CASE WHEN age >= 70 THEN '노인'
    												 ELSE NULL END ORDER BY age) AS age_rank_in_class
    ```