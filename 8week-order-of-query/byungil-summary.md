# 8. SQL의 순서

# 8.1 레코드에 순번 붙이기

- 기본 키가 한 개의 필드일 경우
- 기본 키가 여러 개의 필드로 구성되는 경우
- 테이블을 그룹으로 분할했을 때 그룹 내부의 레코드에 순번을 붙이는 경우를 알아보자.

## 8.1.1 기본 키가 한 개의 필드일 경우

### 윈도우 함수를 사용

- 학생 ID를 오름차순으로 순번을 붙여본다.
- ROW_NUMBER 함수를 사용한다.

```sql
SELECT student_id,
	ROW_NUMBER() OVER (ORDER BY student_id) AS seq
FROM Weights;
```

### 상관 서브쿼리를 사용

- `MySQL`처럼 ROW_NUMBER 함수를 사용할 수 없는 환경에서는 상관 서브쿼리를 사용해야 한다.
    - 5.7 이상 버전부터는 윈도우 함수를 지원한다.

```sql
SELECT student_id,
	(SELECT COUNT(*)
	FROM Weights W2
	WHERE W2.student_id <= W1.student_id) AS seq
FROM Weights W1;
```

- `윈도우 함수`는 스캔 횟수가 1회이고 인덱스 온리 스캔을 사용해 직접적인 접근 회피하여 스캔 횟수가 2회인 상관 서브쿼리보다 성능이 좋다.

## 8.1.2 기본 키가 여러 개의 필드로 구성되는 경우

### 윈도우 함수를 사용

```sql
SELECT class, student_id,
	ROW_NUMBER() OVER (ORDER BY class, student_id) AS seq
FROM Weights2;
```

### 상관 서브쿼리를 사용

- 가장 간단한 방법은 `다중 필드 비교`를 사용하는 것이다.
- 다중 필드 비교는 이름 그대로 복합적인 필드를 하나의 값으로 연결하고 한꺼번에 비교하는 기능이다.

```sql
SELECT class, student_id,
	(SELECT COUNT(*)
	FROM Weights2 W2
	WHERE (W2.class, W2.student_id) <= (W1.class, W1.student_id) ) AS seq
FROM Weights2 W1;
```

- 다중 필드 비교의 장점
    - 필드 자료형을 원하는대로 지정할 수 있다. 숫자와 문자열, 문자열과 숫자라도 가능하다.
    - 암묵적인 자료형 변환도 발생하지 않으므로 기본 키 인덱스도 사용할 수 있다.
    - 또한 필드가 3개 이상일 때도 간단하게 확장할 수 있다.


## 8.1.3 그룹마다 순번을 붙이는 경우

- 학급마다 순번을 붙이는 경우다.
- 테이블을 그룹으로 나누고 그룹마다 내부 레코드에 순번을 붙이는 것이다.

### 윈도우 함수를 사용

- 윈도우 함수로 이를 구현하려면 class필드에 PARTITION BY를 적용해준다.

```sql
SELECT class, student_id,
	ROW_NUMBER() OVER (PARTITION BY class ORDER BY student_id) AS seq
FROM Weights2;
```

### 상관 서브쿼리를 사용

```sql
SELECT class, student_id,
	(SELECT COUNT(*)
	FROM Weights2 W2
	WHERE W2.class = W1.class
	AND W2.student_id <= W1.student_id) AS seq
FROM Weights2 w1;
```


## 8.1.4 순번과 갱신

- 검색이 아니라 갱신에서 순번을 매기는 방법을 살펴보자.
- Weights2 테이블을 변경해서 테이블에 순번 필드(seq)를 만든다.
- 이 필드에 순번을 갱싱하는 UPDATE 구문을 만들어보자.

### 윈도우 함수를 사용

- ROW_NUMBER를 쓸 경우에는 서브쿼리를 함께 사용해야 한다.

```sql
UPDATE Weights3
SET seq = (SELECT seq
          FROM (SELECT class, student_id,
                    ROW_NUMBER() OVER (PARTITION BY class ORDER BY student_id) AS seq
                FROM Weights3) SeqTbl
WHERE Weights3.class = SeqTbl.class
AND Weights3.student_id = SeqTbl.student_id);
```

- 서브쿼리로 SeqTbl 테이블을 만들어 class 그룹마다 순번 매긴 값을 seq 컬럼에 업데이트

### 상관 서브쿼리를 사용

```sql
UPDATE Weights3
SET seq = (SELECT COUNT(*)
          FROM Weights3 W2
          WHERE W2.class = Weights3.class
          AND W2.student_id <= Weight3.student_id);
```

# 8.2 레코드에 순번 붙이기 응용

- 테이블의 레코드에 순번을 붙일 수 있다면, SQL에서 자연 수열(순번)의 성질을 활용한 다양한 테크닉을 사용할 수 있다.
- 자연수의 성질 중에 ‘연속성’과 ‘유일성’을 사용해보자.

## 8.2.1 중앙값 구하기

- 중앙값이란? 숫자를 정렬하고 양쪽 끝에서부터 수를 세는 경우 정중앙에 오는 값이다.
- 단순 평균과 다르게 아웃라이어에 영향을 받지 않는다는 장점이 있다.
    - 아웃라이어는 숫자 집합 내부의 중앙에서 극단적으로 벗어나 있는 예외적인 값이다.
    - 예를 들어 (1, 0, 1, 2, 1, 3, 9999)라는 숫자 집합이 있다면 9999가 아웃라이어다.
- 홀수라면 중앙의 값을 사용한다.
- 짝수라면 중앙에 있는 두 개의 값의 평균을 사용한다.

### 집합 지향적 방법

- 테이블을 상위 집합과 하위 집합으로 분할하고 그 공통 부분을 검색하는 방법이다.

```sql
SELECT AVG(weight)
FROM (SELECT W1.weight
      FROM Weights W1, Weights W2
      GROUP BY W1.weight
      HAVING SUM(CASE WHEN W2.weight >= W1.weight THEN 1 ELSE 0 END) >= COUNT(*) / 2
      AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE 0 END) >= COUNT(*) / 2) TMP;
```

- 코드가 복잡해서 무엇을 하고 있는 건지 한 번에 이해하기 힘들다.
- 성능이 나쁘다.

### 절차 지향적 방법 (1)

- 양쪽 끝에서 레코드 하나씩 세어 중간을 찾는다.

```sql
SELECT AVG(weight) AS median
FROM (SELECT weight,
        ROW_NUMBER() OVER (ORDER BY weight ASC, student_id ASC) AS hi,
        ROW_NUMBER() OVER (ORDER BY weight DESC, student_id DESC) AS lo
      FROM Weights) TMP
WHERE hi IN (lo, lo+1, lo-1);
```

- 홀수의 경우에는 hi = lo가 되어 중심점이 반드시 하나만 존재한다.
- 짝수의 경우에는 hi = lo + 1과 hi = lo - 1의 두 개가 존재한다. (IN 연산자로 비교)
- 집합 지향적 코드와 비교하면 결합을 제거한 대신 정렬이 1회 늘어났다. (성능 개선)

### 주의점

- 순번을 붙일 때 ROW_NUMBER 함수를 사용해야 한다.
    - RANK 또는 DENSE_RANK 함수는 7위 다음에 9위라던지 11위가 두 명이 되는 경우 발생
- ORDER BY의 정렬 케이 weight 필드뿐만 아니라 기본 키인 student_id도 포함해야 한다.

### 절차 지향적 방법 (2)

```sql
SELECT AVG(weight) AS median
FROM (SELECT weight,
        2 * ROW_NUMBER() OVER (ORDER BY weight) - COUNT(*) OVER() AS diff
      FROM Weights) TMP
WHERE diff BETWEEN 0 AND 2;
```

- 절차 지향적 방법 (1)과 비교할 때 정렬이 1회 줄었다.
- ROW_NUMBER == (모든 레코드 개수의 절반 +-1)이 될 때 중간 값이라고 볼 수 있다.

# 8.3 시퀀스 객체, IDENTITY 필드, 채번 테이블

- 최대한 사용하지 않기
- 사용한다면 시퀀스 객체를 사용하기

## 8.3.1 시퀀스 객체

- 테이블 또는 뷰처럼 스키마 내부에 존재하는 객체 중 하나
- CREATE로 생성한다.

```sql
CREATE SEQUENCE testseq
START WITH 1
INCREMENT BY 1
MAXVALUE 100000
MINVALUE 1
CYCLE;
```

- 초기값, 증가값, 최댓값, 최솟값, 최댓값에 도달했을 때 순환 유무 등의 옵션 지정 가능
- 시퀀스 객체가 생성하는 순번은 유일성, 연속성, 순서성을 가짐

### 시퀀스 객체 문제점

- 표준화가 늦어 구현에 따라 구문이 다름 → 이식성 없고 사용할 수 없는 구현도 있음
- 시스템에서 자동 생성된 값이므로 실제 엔티티 속성이 아니다.
- 성능적인 문제 발생

### 시퀀스 객체로 발생하는 성능 문제

- 사용자가 시퀀스 객체에서 NEXT VALUE를 검색할 때 처리 과정이다.
    - 시퀀스 객체에 배타 락 적용
    - NEXT VALUE를 검색
    - CURRENT VALUE를 1만큼 증가
    - 시퀀스 객체에 배타 락을 해제
- 위와 같은 과정으로 동시에 여러 사용자가 시퀀스 객체에 접근할 경우 `락 충돌으로 인한 성능 저하 문제 발생`

### 대처법

- CACHE, NOORDER 객체로 성능 문제 완화가능하다.
- CACHE
    - 새로운 값이 필요할 때마다 메모리에 읽어들일 필요가 있는 값의 수를 설정하는 옵션
    - 값이 클수록 접근 비용 줄일 수 있음
    - 시스템 장애 발생 시 연속성을 담보할 수 없음 (부작용)
- NOORDER
    - 순서성을 담보하지 않아 오버헤드 줄임

### 순번을 키로 사용할 때의 성능 문제

- DBMS의 저장 방식으로 인해 순번처럼 비슷한 데이터 연속으로 INSERT시 물리적으로 같은 영역에 저장되어 특정 물리적 블록에만 I/O 부하 커져 성능 악화 발생한다.

### 대처법

- 일종의 해시와 같이 DBMS 내부에서 연속된 값을 변화를 주어 제대로 분산할 수 있도록 구조 변경
    - I/O양이 늘어나 SELECT 구문 성능이 나빠질 수 있으며 구현의존적 방법이다.
- 인덱스에 복잡한 필드를 추가하여 데이터의 분산도를 높임
    - 복잡한 필드 추가할 경우 불필요한 의미를 생성하므로 다른 개발자가 이해하기 어려울 수 있음
- 리스크를 인지하고 사용하자.

## 8.3.2 IDENTITY 필드

- 테이블의 필드를 정의하고 테이블에 INSERT 발생할 때마다 자동으로 순번 붙여준다.
- 특정한 테이블과 연결되어 여러 테이블에서 사용 불가능하다.
- CACHE, NOORDER를 지정할 수 없거나 제한적으로만 사용 가능하다.

### 채번 테이블

- 과거 사용하던 순번 생성 테이블
- 테이블을 활용해 시퀀스 객체 락 메커니즘을 구현한 것