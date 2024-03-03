# 8장:SQL의 순서

## 23장 : 레코드에 순서 붙이기

1. 기본키가 한개의 필드인 경우
    1. 윈도우 함수를 이용

        ```sql
         SELECT id, ROW_NUMBER() OVER(ORDER BY id)AS seq
        FROM weights;
        ```

    2. 상관 서브쿼리를 사용

        ```sql
        SELECT id, 
        	(SELECT COUNT(*)
        		FROM Weights W2
        		WHERE W2.id <= W1.id) AS seq
        FROM Weights W1;
        ```

2. 기본 키가 여러개의 필드로 구성되는 경우
    1. 윈도우 함수를 이용

        ```sql
         SELECT id, student_id, 
        	ROW_NUMBER() OVER(ORDER BY id, student_d)AS seq
        FROM weights;
        ```

    2. 상관 서브쿼리를 사용

        ```sql
        SELECT id, student_id
        	(SELECT COUNT(*)
        		FROM Weights W2
        		WHERE (W2.id,W2.student_id) <= (W1.id,W1.student_id)) AS seq
        FROM Weights W1;
        ```

3. 그룹마다 순번을 붙이는 경우
    1. 윈도우 함수를 이용(id를 기준으로 붙이는 경우)

        ```sql
         SELECT id, student_id,
        	ROW_NUMBER() OVER(PARTITION BY id ORDER BY student_d)AS seq
        FROM weights;
        ```

    2. 상관 서브쿼리를 사용

        ```sql
        SELECT id, 
        	(SELECT COUNT(*)
        		FROM Weights W2
        		WHERE (W2.class = W1.class AND W2.id <= W1.id,W1.student_id) AS seq
        FROM Weights W1;
        ```

4. 순번과 갱신
    1. 윈도우 함수를 사용

        ```sql
        UPDATE Weights3
        	SET seq= (SELECT seq
        		FROM (SELECT class, student_id,
        			ROW_NUMBER(
        				OVER (PARTITION BY class
        				ORDER BY student_id) AS seq
        			FROM Weights3) SeqTb1
        	WHERE Weights3.class = SeqTb1.class
        		AND Weights3.student_id = SeqTb1.student id);
        ```

    2. 상관서브쿼리를 사용

        ```sql
        UPDATE Weights3
        	SET seq = (SELECT COUNT (*)
        			FROM Weights3 W2
        		WHERE W2.class =Weights3.class
        			AND W2.student_id <- Weights3.student _id);
        ```


## 24장 레코드에 순번 붙이기 응용

1. 중앙값 구하기
    1. 집합 지향적 방법

        ```sql
        SELECT AVG(weight)
        	FROM (SELECT W1 .weight
        		FROM Weights W1, Weights W2
        			GROUP BY W1 weight
        			HAVING SUMCASE WHEN W2.weight >= W1.weight THEN I ELSE 0 END)				
        				>= COUNT (*) / 2
        			AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE O END)
        				>= COUNT (*) / 2 ) TMP;
        ```

    2. 절차 지향적 방법

        ```sql
        SELECT AVG(weight) AS median
        	FROM (SELECT weight,
        		ROW_NUMBER() OVER (ORDER BY weight ASC, student_id ASC) AS hi,
        		ROW_NUMBER() OVER (ORDER BY weight DESC, student_id DESC) AS lo,
        			FROM Weights) TMP
        	WHERE hi IN (lo, lo +1, lo -1);
        ```

        ```sql
        SELECT AVG(weight)
        	FROM (SELECT weight,
        		2 * ROW NUMBER() OVER(ORDER BY weight)
        			- COUNT (*) OVER() AS diff
        		FROM Weights) TMP
        	WHERE diff BETWEEN O AND 2;
        ```

2.  순번을 사용한 테이블 분할
    1. 단절 구간 찾기
    2. 집합 지향적 방법

        ```sql
        SELECT (N1. num + 1) AS gap-start,
        		'~'
        		(MIN(N2.num) - 1) AS gap_end
        	FROM Numbers N1 INNER JOIN Numbers N2
        		ON N2.num > N1.num
        	GROUP BY N1.num
        HAVING (N1.num + 1) < MIN(N2.num);
        ```

    3. 절차 지향적 방법

        ```sql
        SELECT num +1 AS gap start, 
        		'~'
        		(num + diff - 1) AS gap_end
        FROM (SELECT num,
        		MAX(num)
        			OVER(ORDER BY num
        				ROWS BETWEEN 1 FOLLOWING
        					AND 1 FOLLOWING) - num
        		FROM Numbers) TMP (num, diff)
        WHERE diff <> 1:
        ```

3. 테이블에 존재하는 시퀀스 구하기
    1. 집합 지향적 방법

        ```sql
        SELECT MIN(num) AS low,
        		'~'
        		MAX(num) AS high
        FROM (SELECT N1 .num,
        		COUNT (N2.num) - N1.num
        	FROM Numbers N1 INNER JOIN Numbers N2
        		ON N2.num <= N1.num
        	GROUP BY N1 .num) N(num, gp)
        GROUP BY gp;
        ```

    2. 절차 지향적 방법

        ```sql
        SELECT low, high
        	FROM (SELECT low,
        		CASE WHEN high IS NULL
        			THEN MIN(high)
        					OVER (ORDER BY seq
        					ROWS BETWEEN CURRENT ROW
        						AND UNBOUNDED FOLLOWING)
        			ELSE high END AS high
        		FROM (SELECT CASE WHEN COALESCE (prev_diff, 0) <> 1
        									THEN num ELSE NULL END AS low, 
        								 CASE WHEN COALESCE(next_diff, 0) <>1
        									THEN num ELSE NULL END AS high,
        								 seq
        					FROM (SELECT num,
        											 MAX(num)
        											 OVER(ORDER BY num
        														ROWS BETWEEN 1 FOLLOWING
        																AND 1 FOLLOWING) - num AS next_diff,
        											  num - MAX(num)
        															OVER(ORDER BY num
        																		ROWS BETWEEN 1 PRECEDING
        																					AND 1 PRECEDING) AS prev diff,
        												ROW_NUMBER() OVER (ORDER BY num) AS seg
        									FROM Numbers) TMP1 ) TMP2) ТМР3
        WHERE low IS NOT NULL;
        ```


## 25장 : 시퀀스 객체, IDENTITY 필드, 채번 테이블

**모두 최대한 사용하지 않기**

1. 시퀀스 객체
    1. 문제점
        1. 유일성
        2. 연속성
        3. 순서성
    2. 해당 문제 해결을 위한 CACHE, NOORDER객체가 있음
        1. cache : 새로운 값이 필요할 때마다 메모리에 읽어들일 필요가 있는 값의 변수를 설정하는 옵션이다.
            1. 시스템에 장애가 생기면 연속성 담보할 수 없음, 빈숫자가 생길 수 있음
        2. Noorder : 순서성을 담보하지 않아서 오버헤드를 줄여준다.
        3. 연속성, 순서성이 담보돼지 않아도 된다면 CACHE, NOORDER를 채택하여 개선할 수 있다.
2. IDENTITY 필드
    1. 자동 순번 필드
    2. 성능적, 기능적 측면에서  시퀀스 객체보다 심각한 문제를 가짐
    3. IDENTITY 필드는 특정 테이블과 연결되어 CACHE, NOORDER을 지정할 수 없거나 특정제하적으로 사용가능해서 이점은 없다.
3. 채번 테이블
    1. 요즘에는 잘 사용하지 않음
    2. 직접 시퀀스 객체를 구현한것으로 성능이 좋지 않음
    3. 어플리케이션에서 자체적으로 순번을 부여하고 시퀀스 객체의 락 메커니즘을 활용한 방법임
    4. 개선 방법이 없음