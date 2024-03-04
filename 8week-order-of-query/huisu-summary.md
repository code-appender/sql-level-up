# Order of Query

## 레코드에 순번 붙이기

1. 기본 키가 한 개의 필드인 경우

    | student_id | weight |
    | --- | --- |
    | A100 | 50 |
    | A101 | 55 |
    | A124 | 55 |
    | B343 | 60 |
    | B346 | 72 |
    | C563 | 72 |
    | C345 | 72 |
    - 윈도우 함수를 사용하는 경우
        
        ```sql
        SELECT student_id, ROW_NUMBER() OVER (ORDER BY student_id) AS seq
        FROM Weights;
        ```
        
    - 상관 서브 쿼리를 사용하는 경우
        
        ```sql
        SELECT student_id, 
        	(SELECT COUNT(*) 
        	FROM Weights W2
        	WHERE W2.student_id <= W1.student_id) AS seq
        FROM Weights W1;
        ```

2. 기본 키가 여러 개로 구성되는 경우

    | class | student_id | weight |
    | --- | --- | --- |
    | 1 | 100 | 50 |
    | 1 | 101 | 55 |
    | 1 | 102 | 56 |
    | 2 | 100 | 60 |
    | 2 | 101 | 72 |
    | 2 | 102 | 73 |
    | 2 | 103 | 73 |
    - 윈도우 함수를 사용하는 경우
        
        ```sql
        SELECT class, student_id, ROW_NUMBER() OVER (ORDER BY class, student_id) AS seq
        FROM Weights;
        ```
        
    - 상관 서브 쿼리를 사용하는 경우
        
        ```sql
        SELECT student_id, 
        	(SELECT COUNT(*) 
        	FROM Weights W2
        	WHERE (W2.class, W2.student_id) <= (W1.class, W1.student_id)) AS seq
        FROM Weights W1;
        ```

3. 그룹마다 순번을 붙이는 경우
    - 윈도우 함수를 사용하는 경우

        ```sql
        SELECT class, student_id, ROW_NUMBER() OVER (PARTITION BY class ORDER BY student_id) AS seq
        FROM Weights;
        ```

    - 상관 서브 쿼리를 사용하는 경우

        ```sql
        SELECT student_id, 
        	(SELECT COUNT(*) 
        	FROM Weights W2
        	WHERE W2.class = W1.class AND
        				W2.student_id <= W1.student_id) AS seq
        FROM Weights W1;
        ```


## 레코드에 순번 붙이기 응용

1. 중앙값 구하기

    | student_id | weight |
    | --- | --- |
    | A100 | 50 |
    | A101 | 55 |
    | A124 | 55 |
    | B343 | 60 |
    | B346 | 72 |
    | C563 | 72 |
    | C345 | 72 |
    - 집합 지향적 방법: 테이블을 상위 집합과 하위 집합으로 분할하고 공통 부분 검색
        
        ```sql
        SELECT AVG(weight)
        	FROM (SELECT W1.weight
        		FROM Weights W1, Weights W2
        		GROUP BY W1.weight
        		HAVING SUM (CASE WHEN W2.weight >= W1.weight THEN 1 ELSE 0 END)
        			>= COUNT(*)/2
        		AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE 0 END)
        			>= COUNT(*)/2) TMP;
        ```
        
        - 코드가 복잡해서 무엇을 하고 있는지 정확하게 이해 힘듦
        - 성능 저조 → 자기 결합 수행
    - 절차 지향적 방법 1 → 양쪽 끝에서 레코드 하나씩 세어 중간을 찾음
        
        ```sql
        SELECT AVG(weight) AS median
        	FROM (SELECT weight,
        				ROW_NUMBER() OVER (ORDER BY weight ASC, student_id ASC) AS hi,
        				ROW_NUMBER() OVER (ORDER BY weight DESC, student_id DESC) AS lo
        	FROM Weights) TMP
        WHERE hi IN (lo, lo + 1, lo - 1);
        ```
        
    - 절차 지향적 방법 2 → 반환점 발견
        
        ```sql
        SELECT AVG(weight) 
        FROM (SELECT weight, 
        		2 * ROW_NUMBER() OVER (ORDER BY weight) - COUNT(*) OVER() AS diff 
        		FROM Weights) TMP
        WHERE diff BETWEEN 0 AND 2;
        ```

2. 순번을 이용한 테이블 분할

    | num |
    | --- |
    | 1 |
    | 3 |
    | 4 |
    | 7 |
    | 8 |
    | 9 |
    | 12 |
    - 집합 지향적 방법 → 비어 있는 숫자 모음을 표시
        
        ```sql
        SELECT (N1.num + 1) AS gap_start,
        	'~',
        	(MIN(N2.num) - 1) AS gap_end
        FROM Numbers N1 INNER JOIN Numbers N2
        ON N2.num > N1.num
        GROUP BY N1.um
        HAVING (N1.num + 1) < MIN(N2.num);
        ```
        
    - 절차 지향적 방법 → 다음 레코드와 비교
        
        ```sql
        SELECT num + 1 AS gap_start, 
        	'~',
        	(num + diff - 1) AS gap_end
        FROM (SELECT num.
        		MAX(num)
        		OVER (ORDER BY num
        			ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING) - num
        		FROM Numbers) TMP (num, diff)
        WHERE diff <> 1;
        ```

3. 테이블에 존재하는 시퀀스 구하기
    - 집합 지향적 방법

        ```sql
        SELECT MIN(num) AS low,
        	'~',
        	MAX(num) AS high
        FROM (SELECT N1.num,
        	COUNT(N2.num) - N1.num
        	FROM Numbers N1 INNER JOIN Numbers N2
        	ON N2.num <= N1.num
        	GROUP BY N1.num) N(num, gp)
        GROUP BY gp;
        ```

    - 절차 지향적 방법

        ```sql
        SELECT low, high 
        	FROM (SELECT low,
        		CASE WHEN hight IS NULL
        			THEN MIN(high)
        				OVER (ORDER BY seq
        					ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING)
        			ELSE high END AS high
        	FROM (SELECT CASE WHEN COALESCE(prew_diff, 0) <> 1
        			THEN num ELSE NULL END AS low,
        		CASE WHEN COALESCE(next_diff, 0) <> 1
        			THEN num ELSE NULL END AS high,
        		seq
        	FROM (SELECT num,
        			MAX(num)
        			OVER(ORDER BY num
        				ROWS BETWEEN 1 FOLLOWING
        					AND 1 FOLLOWING) - num AS next_diff,
        			num - MAX(num)
        				OVER (ORDER BY num
        					ROWS BETWEEN 1 PRECEDING
        						AND 1 PRECEDING) AS prev_diff,
        			ROW_NUMBER() OVER (ORDER BY num) AS seq
        	FROM Numbers) TMP1) TMP2) TMP3
        WHERE low IS NOT NULL;
        ```


## 시퀀스 객체 IDENTITY 필드, 채번 테이블

1. 시퀀스 객체
    - 테이블 또는 뷰처럼 스키마 내부에 존재하는 객체 중 하나
    - START: 초기값
    - INCREMENT: 증가값
    - MAXVALUE: 최대값
    - MINVALUE: 최소값
    - CYCLE: 최대값에 도달했을 때 순환 유무
2. 시퀀스 객체의 문제점
    - 표준화가 늦어서 구현에 따라 구민이 달라 이식성이 없고 사용할 수 없는 구문도 존재
    - 시스템에서 자동으로 생성되는 값이므로 실제 엔티티 속성이 아님
    - 성능적인 문제
3. 시퀀스의 성능적인 문제점
    - 유일성: 중복값이 생성될 수 있음
    - 연속성: 생성된 값에 비어 있는 부분이 존재할 수 있음
    - 순서성: 순번의 대소 관계가 역전될 수 있음
    - 이러한 성능을 만족하기 위해 락 매커니즘 사용됨
        - 시퀀스 객체에 베타 락을 적용
        - NEXT VALUE를 검색
        - CURRENT VALUE를 1만큼 증가
        - 시퀀스 객체에 베타 락을 해제
4. 시퀀스 객체로 발생하는 성능 문제의 대처
    - CACHE: 새로운 값이 필요할 때마다 메모리에 읽어들일 필요가 있는 값의 수를 설정
    - NOORDER: 순서성을담보하지 않아서 오버 헤드 줄임
5. IDENTITY 필드
    - 자동 순번 필드: 테이블의 필드로 정의하고 테이블에 INSERT가 발생할 때마다 자동으로 순번을 붙임
    - 이점이 없어서 거의 사용 안 하고 시퀀스 사용