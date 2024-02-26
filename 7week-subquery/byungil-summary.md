# 7. 서브쿼리

# 7.1 서브쿼리가 일으키는 폐해

## 7.1.1 서브쿼리의 문제점

- 서브쿼리의 성능적 문제는 결과적으로 서브쿼리가 실체적인 데이터를 저장하고 있지 않다는 점에서 기인한다.

### 문제점 1) 연산 비용 추가

- 실체적인 데이터를 저장하고 있지 않다는 것은 서브쿼리에 접근할 때마다 SELECT 구문을 실행해서 데이터를 만들어야 한다는 뜻이다.
- 따라서 SELECT 구문 실행에 발생하는 비용이 추가된다. 서브쿼리의 내용이 복잡할수록 실행 비용 높아진다.

### 문제점 2) 데이터 I/O 비용 발생

- 연산 결과는 어딘가에 저장하기 위해 써두어야 한다. 메모리 용량이 충분하다면 이러한 오버헤드가 적지만, 데이터양이 큰 경우 등에는 DBMS가 저장소에 있는 파일에 결과를 쓸 때도 있다.
- TEMP 탈락 현상의 일종이라고 말할 수 있는데, 저장소 성능에 따라 접근 속도가 급격하게 떨어진다.

### 문제점 3) 최적화를 받을 수 없음

- 서브쿼리로 만들어지는 데이터는 구조적으로는 테이블과 차이가 없다.
- 하지만 명시적인 제약 또는 인덱스가 작성되어 있는 테이블과 달리, 서브퀴리에는 그러한 메타 정보가 하나도 존재하지 않는다. 따라서 옵티마이저가 쿼리를 해석하기 위해 필요한 정보를 서브쿼리에서는 얻을 수 없다.
- 이러한 문제점 때문에 내부적으로 복잡한 연산을 수행하거나 결과 크기가 큰 서브쿼리를 사용할 때는 성능 리스크를 고려해야 한다.

## 7.1.2 서브쿼리 의존증

- 고객의 구입 명세 정보를 저장하는 테이블(Receipts)에 순번(Seq 또는 AI idx) 필드는 오래전에 구입했을수록 낮은 값을 가진다.
- 이때 고객별 최소 순번을 구하는 상황을 가정 하자.
- 처음 직면하는 문제점은 최소 순번이 고객마다 다르다는 것이다.

```
user_id | seq | price
--------*-----*------
A       |   1|    500
B       |   5|    100
C       |  10|    600
D       |   3|   2000
```

### 서브쿼리를 사용한 방법

- 고객들의 최소 순번 값을 저장하는 서브쿼리(R2)를 만들고, 기존의 Receipts 테이블과 결합하는 방법이 있다.

```sql
SELECT R1.cust_id, R1.seq, R1.price
	FROM Receipts R1
		INNER JOIN
			(SELECT cust_id, MIN(seq) AS min_seq
				FROM Receipts
				GROUP BY cust_id) R2
	ON R1.cust_id = R2.cust_id
	AND R1.seq = R2.min_seq; 
```

### 두 가지 단점이 존재한다.

- 첫 번째는 코드가 복잡해서 읽기 어렵다. 특히 서브쿼리를 사용하면 코드가 여러 계층에 걸쳐 만들어지므로 가독성이 떨어진다.
- 두 번째는 성능이다. 이러한 코드의 성능이 나쁜 이유로 4가지를 꼽을 수 있다.
    - 서브쿼리는 대부분 일시적인 영역(메모리 또는 디스크)에 확보되므로 오버헤드가 생긴다.
    - 서브쿼리는 인덱스 또는 제약 정보를 가지지 않기 때문에 최적화되지 못한다.
    - 이 쿼리는 결합을 필요로 하기 때문에 비용이 높고 실행 계획 변동 리스크가 발생한다.
    - Receipts 테이블에 스캔이 두 번 필요하다.

### 상관 서브쿼리는 답이 될 수 없다.

- 흔히 하는 실수부터 짚어보자.
- 바로 상관 서브쿼리를 사용한 동치 변환이다.

```sql
SELECT cust_id, seq, price
FROM Receipts R1
WHERE seq = (SELECT MIN(seq)
							FROM Receipts R2
							WHERE R1.cust_id = R2.cust_id);
```

상관 서브쿼리를 사용하더라도 Receipts 테이블에 접근이 두 번 발생한다.

### 윈도우 함수로 결합을 제거

- 일단 개선해야 하는 부분은 Receipts 테이블에 대한 접근을 1회로 줄이는 것이다.
- SQL 튜닝에서 가장 중요한 부분이 바로 I/O를 줄이는 것이다.
- 접근을 줄이려면 윈도우 함수 ROW_NUMBER를 다음과 같은 형태로 사용한다.

```sql
SELECT cust_id, seq, price
FROM (SELECT cust_id, seq, price,
						 ROW_NUMBER()
							OVER (PARTITION BY cust_id
											ORDER BY seq) AS row_seq
			FROM Receipts) WORK
WHERE WORK.row_seq = 1;
```

- ROW_NUMBER로 각 사용자의 구매 이력에 번호를 붙였다.
- 이렇게 하면 Receipts 테이블에 한 번만 접근한다.

## 7.1.3 장기적 관점에서의 리스크 관리

- 저장소의 I/O 양을 감소시키는 것이 SQL 튜닝의 가장 기본 원칙이다.
- 처음 사용했던 쿼리와 비교해보면 결합을 사용한 부분을 제거했다.
- 이렇게 하면 단순한 성능 향상뿐만 아니라 성능의 안정성 확보도 기대할 수 있다.
- 결합을 사용한 쿼리는 두 개의 불안정 요소 존재
    - 결합 알고리즘의 변동 리스크
    - 환경 요인에 의한 지연 리스크(인덱스, 메모리, 매개변수 등)
    - 상관 서브쿼리를 사용한 쿼리의 실행 계획 역시 결합을 사용한 쿼리와 비슷하게 나온다. 따라서 상관 서브쿼리를 사용한 쿼리도 리스크에 해당한다.

### 알고리즘 변동 리스크

- 레코드 수가 적은 테이블이 포함된 경우에는 Nested Loops가 선택되기 쉽고, 큰 테이블을 결합하는 경우에는 Sort Merge 또는 Hash가 선택되기 쉽다.
- 따라서 처음에는 테이블의 레코드 개수가 적어 Nested Loops를 사용하다가도, 시스템을 계속 운용하면서 레코드가 늘어나, 어느 순간 역치를 넘으면 실행 계획 변동이 생긴다.
- 이따 성능에 큰 변화가 일어난다. (좋아지는 경우도 있지만 악화되는 경우도 많다.)
- 결국 결합을 사용하면 이러한 변동 리스크를 안을 수 밖에 없다.
- 또한 같은 실행 계획이 계속해 선택되는 경우에도 문제가 있다.
- 데이터 양이 많아지면서 Sort Merge 또는 Hash에 필요한 메모리가 부족해지면 일시적으로 저장소를 사용한다. 결국 그 시점을 기준으로 성능이 대폭 떨어지게 된다.(TEMP 탈락 현상)

### 환경 요인에 의한 지연 리스크

- Nested Loops의 내부 테이블 결합 키에 인덱스가 존재하면 성능이 크게 개선된다.
- 또한 Sort Merge 또는 Hash가 선택되어 TEMP 탈락이 발생하는 경우 작업 메모리를 늘려주면 성능을 개선할 수 있다.
- 하지만 항상 결합 키에 인덱스가 존재하는 것은 아니다.
- 또한 메모리 튜닝은 한정된 리소스 내부에서의 트레이드오프를 발생시킨다.
- 다시 말해, 결합을 사용한다는 것은 곧 장기적 관점에서 고려해야 할 리스크를 늘리게 된다는 뜻이다.
- 따라서 우리는 옵티마이저가 이해하기 쉽게 쿼리를 단순하게 작성해야 한다.
    - 실행 계획이 단순할수록 성능이 안정적이다.
    - 엔지니어는 기능(결과)뿐만 아니라 비기능적인 부분(성능)도 보장할 책임이 있다.

## 7.1.4 서브쿼리 의존증 - 응용편

- 앞에서는 고객이 가지는 순번의 최솟값을 가진 레코드를 찾아보았는데,
- 이번에는 최댓값을 가지는 레코드와 함께, 양쪽 price 필드의 차이도 구해보자.

```
cust_id | diff
---------------
A       |  -200
B       |  -900
C       |   550
D       |     0
```

### 다시 서브쿼리 의존증

- 서브쿼리를 사용한다면 최솟값의 집합을 찾고, 최댓값의 집합을 찾은 뒤에 고객 ID를 키로 결합하면 된다.

```sql
SELECT TMP_MIN.cust_id,
        TMP_MIN.price - TMP_MAX.price AS diff
FROM (SELECT R1.cust_id, R1.seq, R1.price
        FROM Receipts R1
            INNER JOIN
            (SELECT cust_id, MIN(seq) AS min_seq
            FROM Receipts
            GROUP BY cust_id) R2
        ON R1.cust_id=R2.cust_id
        AND R1.seq=R2.min_seq) TMP_MIN
        INNER JOIN
            (SELECT R3.cust_id, R3.seq, R3.price
            FROM Receipts R3
            INNER JOIN
                (SELECT cust_id, MAX(seq) AS min_seq
                FROM Receipts
                GROUP BY cust_id) R4
        ON R3.cust_id=R4.cust_id
        AND R3.seq=R4.min_seq) TMP_MAX
        ON TMP_MIN.cust_id=TMP_MAX.cust_id;
```

- TMP_MIN이 최솟값의 집합이고 TMP_MAX가 최댓값의 집합이다.
- 쿼리가 굉장히 길어졌고 가독성도 좋지 않다.
- 서브쿼리의 계층이 굉장히 깊어서 어떤 부분이 서브쿼리인지 확인하는 것만도 힘들다.
- 이전의 쿼리를 두 번 붙여넣기 한 것이라 테이블에 대한 접근도 2배가 되어 4번 이루어진다.
- 성능이 좋을 리 없다.

### 레코드 간 비교에서도 결합은 불필요

- 윈도우 함수로 결합을 제거와 마찬가지로 `테이블 접근과 결합을 얼마나 줄일 수 있는지`가 개선포인트다.
- CASE식을 함께 사용한다.

```sql
SELECT cust_id,
        SUM(CASE WHEN min_seq=1 THEN price ELSE 0 END)
        - SUM(CASE WHEN max_seq=1 THEN price ELSE 0 END) AS diff
FROM (SELECT cust_id, price,
        ROW_NUMBER() OVER (PARTITION BY cust_id ORDER BY seq) AS min_seq,
        ROW_NUMBER() OVER (PARTITION BY cust_id ORDER BY seq DESC) AS max_seq
        FROM Receipts) WORK
WHERE WORK.min_seq=1
    OR WORK.max_seq=1
GROUP BY cust_id;
```

- 이렇게 하면 서브쿼리는 WORK 하나뿐이다. 결합도 발생하지 않는다.
- 테이블의 스캔 횟수가 1회로 감소했다.

## 7.1.5 서브쿼리는 정말 나쁠까?

- 서브쿼리 자체가 나쁜 것은 아니며, 무조건 사용해서 안 된다는 것은 아니다.
- 게다가 서브쿼리를 사용하지 않으면 해결할 수 없는 상황도 많다.
- 서브쿼리를 사용하면 문제를 분할하여 생각하기가 쉬워지는 만큼 생각의 보조 도구라고 할 수 있다.
- 바텀업 타입의 사고방식과 굉장히 좋은 상성을 가지고 있다.
- 코드 레벨에서 본다면 실제로 효율적인 코드가 되지는 않는다.

# 7.2 서브쿼리 사용이 더 나은 경우

- 결합과 관련된 쿼리다.
- 결합할 때는 최대한 결합 대상 레코드 수를 줄이는 것이 중요하다.
- 옵티마이저가 이러한 것을 잘 판별하지 못할 때는, 사람이 직접 연산 순서를 명시해주면 성능적으로 좋은 결과를 얻을 수 있다.

## 7.2.1 결합과 집약 순서

### 첫 번째 방법: 결합을 먼저 수행

```sql
SELECT C.co_cd, MAX(C.district),
        SUM(emp_nbr) AS sum_emp
FROM Companies C
    INNER JOIN
        Shops S
    ON C.co_cd = S.co_cd
WHERE main_flg = 'Y'
GROUP BY C.co_cd;
```

- 첫 번째 방법은 회사 테이블과 사업소 테이블의 결합을 먼저 수행하고, 결과에 GROUP BY를 적용해서 집약했다.

### 두 번째 방법: 집약을 먼저 수행

```sql
SELECT C.co_cd, C.district, sum_emp
FROM Companies C
    INNER JOIN
        (SELECT co_cd, SUM(emp_nbr) AS sum_emp
        FROM Shops
        WHERE main_flg = 'Y'
        GROUP BY co_cd) CSUM
    ON C.co_cd = CSUM.co_cd;
```

- 두 번째 방법은 먼저 사업소 테이블을 집약해서 직원 수를 구하고, 회사 테이블과 결합했다.

### 결합 대상 레코드 수

- 첫 번째 방법(결합을 먼저 수행)의 경우 결합 대상 레코드 수는 다음과 같다.
    - 회사 테이블: 레코드 4개
    - 사업소 테이블: 레코드 10개
- 두 번째 방법(집약을 먼저 수행)의 경우 결합 대상 레코드 수는 다음과 같다.
    - 회사 테이블: 레코드 4개
    - 사업소 테이블(CSUM): 레코드 4개
- 따라서 결합 비용을 낮출 수 있다.