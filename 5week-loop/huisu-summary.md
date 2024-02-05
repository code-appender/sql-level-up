# Iteration

## 반복문 의존증

1. 반복문 의존증적 코드
    - SQL에는 반복문이 없음 → 그것이 더 좋기 때문

   > 관계 조작은 관계 전체를 모두 조작이 대상으로 삼는다. 이러한 것의 목적은 반복을 제외하는 것이다. 최종 사용자의 생산성을 생각하면 이러한 조건을 만족해야 한다. 그래야만 응용 프로그래머의 생산성에도 기여할 수 있을 것이다.
   >
    - 호스트 언어에서 반복문을 사용하여 SQL 반복 해결 → 반복계 코드의 탄생
    - 반복계 코드 지양
    - 물론 CASCADE처럼 미들웨어나 프레임워크가 내부적으로 반복계 코드를 사용해서 선택권이 없는 경우도 존재

## 반복계의 공포

1. 반복계 코드의 장점
    - SQL을 잘 모르는 사람도 사용 가능 → 구문 단순화
    - 실행 계획의 안정성: 실행 계획에 변동 위험이 거의 없음
    - 예상 처리 시간의 정밀도
    - 트랜잭션 정밀 제어 편리
        - 중간에 오류가 발생했어도 커밋 지점 근처에서 다시 처리
2. 반복계 코드의 단점
    - 성능
        - 처리 시간 = 처리 횟수 * 한 회에 걸리는 처리 시간
    - SQL 실행의 오버헤드
        - SQL 을 실행하기까지는 다음과 같은 과정 진행

            ```java
            전처리
            1. SQL 구문을 네트워크로 전송
            2. 데이터베이스 연결
            3. SQL 구문 파싱
            4. SQL 구문의 실행 계획 생성 또는 평가
            
            후처리
            5. 결과 집합을 네트워크로 전송
            ```

        - 애플리케이션 서버와 데이터베이스 서버를 문리적으로 분리해서 사용하기 때문에 1 ~ 5 과정 필요
        - 2 의 대안 → Connection Fool
        - SQL 파스는 DBMS마다 하는 방법도 미묘하게 다르고 종류도 많음 → 오버헤드 발생
    - 병렬 분산의 어려움
        - 반복계는 반복 1회마다의 처리가 굉장히 단순해 리소스를 분산하는 병렬 처리 최적화가 어려움
    - 데이터베이스의 진화로 인한 혜택의 사각지대
        - 옵티마이저/SSD 등 다양한 진화가 이루어지고 있음
        - 대규모 데이터를 다루는 복잡한 SQL 구문을 익히지 않으면 이러한 발전의 혜택을 누릴 수 없음
    - 느린 구문을 튜닝할 수 있는 가능성도 거의 없음
3. 포장계 코드의 장점
    - 성능
4. 포장계 코드의 단점
    - 구문이 복잡해져서 유지 보수성이 떨어지는 SQL 구문 탄생
    - 실행 계획 변동 가능성이 매우 큼
    - 실행 계획에 따라 성능이 전혀 달라지므로 프로그램 사양을 사전에 예상하기 힘듦
    - 트랜잭션 처리 어려움 → 갱신 처리 중간에 오류가 생기면 처음부터 다시 시행
5. 반복계를 빠르게 만드는 방법
    - 반복계를 포장계로 전면 수정
    - 각각의 SQL을 빠르게 수정 → 효과 거의 없음
    - 다중화 처리 → 데이터를 분할할 수 있는 명확한 키가 없거나 순서가 중요한 처리, 병렬화했을 때 물리 리소스가 부족하다면 불가능

## SQL에서는 반복문을 어떻게 표현할까?

1. 포인트는 CASE 식과 윈도우 함수
    - SQL에서 반복을 대신하는 수단은 CASE 식과 윈도우 함수
    - IF-THEN-ELSE 구문에 대응하는 기능
    - 전년도에 비한 매출 변화 칼럼 추가 예제 (최종적으로 구할 테이블)

        | company | year | sale | var |
        | --- | --- | --- | --- |
        | A | 2002 | 50 |  |
        | A | 2003 | 53 | + |
        | A | 2004 | 55 | + |
        | A | 2005 | 55 | = |
        | B | 2002 | 27 |  |
        | B | 2003 | 28 | + |
        | B | 2004 | 28 | = |
        | B | 2005 | 30 | + |
        | C | 2002 | 40 |  |
        | C | 2003 | 39 | - |
        | C | 2004 | 38 | - |
        | C | 2005 | 35 | - |
    - CASE식과 윈도우 함수를 이용한 해결
        
        ```sql
        INSERT INTO Sales@
        SELECT company,
        			 year,
               sale,
               CASE SIGN(sale - MAX(sale)
        												OVER (PARTITION BY company
        															ORDER BY year
        															ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING))
        			 WHEN 0 THEN '='
        			 WHEN 1 THEN '+'
        		   WHEN -1 THEN '-'
        			 ELSE NULL END AS var
        FROM Sales;
        ```
        
    - SIGN 함수
        - 숫자 자료형을 매개 변수로 받음
        - 음수면 -1, 양수면 1, 0이면 0 리턴
    - ROWS BETWEEN 옵션
        - 대상 범위의 레코드를 직전 1개로 제한
    - 윈도우 함수로 직전 회사명과 직전 매상 검색
        
        ```sql
        SELECT company,
        			 year,
               sale,
        			 MAX(company)
        					 OVER (PARTITION BY company
        								 ORDER BY year
        								 ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) AS pre_company,
        			 MAX(sale)
        					 OVER (PARTITION BY company
        								 ORDER BY year
        								 ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) AS pre_sales
        FROM Sales;
        			
        ```
        
    - 상관 서브쿼리로 직전 회사명과 직전 매상 검색
        
        ```sql
        SELECT company,
        			 year,
               sale,
        			 (SELECT company 
        				FROM Sales S2 
        				WHERE S1.company = S2.company
        				AND year = (SELECT MAX(year) 
        										WHERE S1.company = S3.company 
        										AND S1.year > S3.year)) AS pre_company,
        			(SELECT sale 
        				FROM Sales S2 
        				WHERE S1.company = S2.company
        				AND year = (SELECT MAX(year) 
        										WHERE S1.company = S3.company 
        										AND S1.year > S3.year)) AS pre_sale
        FROM Sales S1;
        ```
        
    1. 최대 반복 횟수가 정해진 경우

        | pcode | distric_name |
        | --- | --- |
        | 4130001 | 시즈오카 아타미 이즈미 |
        | 4130002 | 시즈오카 아타미 이즈산 |
        | 4130103 | 시즈오카 아타미 아지로 |
        | 4130041 | 시즈오카 아타미 아오바초 |
        | 4103213 | 시즈오카 이즈 아오바네 |
        | 4380824 | 시즈오카 이와타 아카 |
        - 우편 번호 4130033과 가장 가까운 지역 찾기
        - 일단 4130033이 존재하는지 검색
        - 이후 413003*이 존재하는지 검색
        - 반복 횟수가 7회로 고정
        - 몇 글자가 일치하는지에 따라 순위를 매기는 쿼리
        
        ```sql
        SELECT pcode, district_name
        FROM PostalCode
        WHERE CASE WHEN pcode '4130033' THEN 0
        					 WHEN pcode LIKE '413003%' THEN 1
        					 WHEN pcode LIKE '41300%' THEN 2
        					 WHEN pcode LIKE '4130%' THEN 3
        					 WHEN pcode LIKE '413%' THEN 4
        					 WHEN pcode LIKE '41%' THEN 5
        					 WHEN pcode LIKE '4%' THEN 6
        			ELSE NULL END = (SELECT MIN (CASE WHEN pcode '4130033' THEN 0
        																	 WHEN pcode LIKE '413003%' THEN 1
        																	 WHEN pcode LIKE '41300%' THEN 2
        																	 WHEN pcode LIKE '4130%' THEN 3
        																	 WHEN pcode LIKE '413%' THEN 4
        																	 WHEN pcode LIKE '41%' THEN 5
        																	 WHEN pcode LIKE '4%' THEN 6
        																	 ELSE NULL END)
        											 FROM PostalCode);
        ```
        
        - 테이블 풀 스캔 2회 발생
    2. 반복 횟수가 정해지지 않은 경우

        | name | pcode | new_pcode |
        | --- | --- | --- |
        | A | 4130001 | 4130002 |
        | A | 4130002 | 4130103 |
        | A | 4130103 |  |
        | B | 4130041 |  |
        | C | 4103213 | 4380824 |
        | C | 4380824 |  |
        - 인접리스트 모델의 경우
        - A는 4130001 → 4130002 → 4130103으로 이사
        - 가장 오래된 주소인 계층 구조를 검색하는 방법은 재귀 공통 테이블 식을 사용하는 방법
        
        ```sql
        WITH RECURSIVE Explosion (name, pcode, new_pcode, depth)
        AS
        (SELECT name, pcode, new_pcode, 1
        FROM PostalHistory
        WHERE name = 'A'
        AND new_pcode IS NULL
        UNION
        SELECT Child.name, Child.pcodem Child.new_pcodem depth + 1
        FROM Explosion AS Parent, PostalHistory AS Child
        WHERE Parent.pcode = Child.new_pcode
        AND Parent.name = Child.name)
        
        SELECT name, pcode, new_pcode
        FROM Explosion
        WHERE depth = SELECT MAX(depth) FROM Explosion);
        ```