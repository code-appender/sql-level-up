# SubQuery

## 서브쿼리가 일으키는 폐해

1. 서브쿼리의 문제점
- 연산 비용 추가: 실체적인 데이터를 저장하고 있지 않아서 서브쿼리에 접근할 때마다 SELECT 구문 사용
- 데이터 I/O 비용 발생: 데이터양이 큰 경우 연산 결과를 저장소가 있는 파일에 쓰면 성능 저하 (TEMP)
- 최적화를 받을 수 없음: 메타 정보가 없는 비실체적 데이터라 옵티마이저의 최적화 대상에서 제외
1. 서브쿼리가 일으키는 폐해


| cust_id | seq | price |
| --- | --- | --- |
| A | 1 | 500 |
| A | 2 | 1000 |
| A | 3 | 700 |
| B | 5 | 100 |
| B | 6 | 5000 |
| B | 7 | 300 |
| B | 9 | 200 |
| B | 12 | 1000 |
| C | 10 | 600 |
| C | 20 | 100 |
| C | 45 | 200 |
| C | 70 | 50 |
| C | 3 | 2000 |
- 서브 쿼리 이용

    ```sql
    SELECT R1.cust_id, R1.seq, R1.price
    FROM Receipts R1
    INNER JOIN (SELECT cust_id MIN(seq) AS min_seq
    						FROM Receipts
    						GROUP BY cust_id) R2
    	ON R1.cust_id = R2.cust_id
    	AND R1.seq = R2.min_seq;
    ```

    - 서브쿼리는 대부분 일시적인 영역에 확보되기에 오버헤드 발생
    - 서브쿼리는 인덱스 또는 제약 정보를 가지지 않기 때문에 최적화 어려움
    - 이 쿼리는 결합을 필요로 하기 때문에 비용이 높고 실행 계획 변동 리스크 발생
    - Receipts 테이블에 스캔 두 번 필요
- 상관 서브쿼리 사용

    ```sql
    SELECT cust_id, seq, price
    FROM Receipts R1
    WHERE seq = (SELECT MIN(seq) 
    							FROM Receipts R2 
    							WHERE R1.cust_id = R2.cust_id);
    ```

    - 그래도 전체 테이블 스캔 두 번 발생
- 윈도우 함수로 결합 제거

    ```sql
    SELECT cust_id, seq, price
    FROM (SELECT cust_id, seq. price, ROW_NUMBER() OVER
    			(PARTITION BY cust_id ORDER BY seq) AS row_seq
    			FROM Receipts) WORK
    WHERE WORK.row_seq = 1;
    ```

    - row_seq 를 추가한 게 WORK 테이블
    - 전체 테이블 접근 1회 발생
1. 장기적 관점에서의 리스크 관리
    - 결합을 사용한 쿼리의 불안정 요소
        - 결합 알고리즘의 변동 리스크
        - 환경 요인에 의한 지연 리스크
    - 알고리즘 변동 리스크
        - 결합 알고리즘에는 크게 Nested Loops, Sort Merge, Hash가 존재
        - 처음에는 테이블의 개수가 적어 Nested Loops을 사용하다가도 시스템을 계속 운용하면 레코드가 늘어나 실행 계획 변동이 발생
        - 데이터 양이 많아지면 TEMP 현상으로 저장소를 사용하기도 하는데 성능 하락의 원인
    - 환경 요인에 의한 지연 리스크
        - Nested Loops 내부 테이블 결합 키에 인덱스가 존재하면 성능이 크게 개선됨
        - TEMP 탈락이 발생했을 때 작업 메모리를 늘려 주면 됨
        - 결합을 사용한다는 것은 장기적 관점에서 고려해야 할 리스크를 늘린다는 것
2. 서브쿼리는 정말 나쁠까?
    - 서브쿼리 자체가 나쁜 것은 아니지만 생각의 보조 도구

## 서브쿼리 사용이 더 나은 경우

1. 결합과 집약 순서

    | co_cd | district |
    | --- | --- |
    | 001 | A |
    | 002 | B |
    | 003 | C |
    | 004 | D |

    | co_cd003 | shop_id | emp_nbr | main_flg |
    | --- | --- | --- | --- |
    | 001 | 1 | 300 | Y |
    | 001 | 2 | 400 | N |
    | 001 | 3 | 250 | Y |
    | 002 | 1 | 100 | Y |
    | 002 | 2 | 20 | N |
    | 003 | 1 | 400 | Y |
    | 003 | 2 | 500 | Y |
    | 003 | 3 | 300 | N |
    | 003 | 4 | 200 | Y |
    | 004 | 1 | 999 | Y |

    - 회사마다 main_flg가 Y인 사업소의 직원 수 구하기
    - 결합을 먼저 수행
        
        ```sql
        SELECT C.co_cd, MAX(C.district), SUM(emp_nbr) AS sum_emp
        FROM Companies C INNER JOIN Shops S
        ON C.co_cd = S.co_cd
        WHERE main_flg = 'Y'
        GROUP BY C.co_cd;
        ```
        
        - 결합 대상 레코드 수
            - 회사 테이블: 레코드 4개
            - 사업소 테이블: 레코드 10개
    - 집약을 먼저 수행
        
        ```sql
        SELECT C.co_cd, C.district, sum_emp
        FROM Companies C INNER JOIN (SELECT co_cd, SUM(emp_nbr) AS sum_emp
        														FROM Shops
        														WHERE main_flg = 'Y'
        														GROUP BY co_cd) CSUM
        ON C.co_cd = CSUM.co_cd;
        ```
        
        - 결합 대상 레코드 수
            - 회사 테이블: 레코드 4개
            - 사업소 테이블: 레코드 4개
    - 데이터양이 많다면 결합 대상 레코드 수를 집약하는 편이 I/O 비용을 더 줄일 수 있음