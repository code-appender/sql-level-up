# SQL의 순서

## 8.1 레코드에 순번 붙이기

### 8.1.1 키본 키가 한 개의 필드인 경우

#### 윈도우 함수를 사용
```SQL
SELECT ROW_NUMBER() OVER(ORDER BY {컬럼명}) AS 순번
FROM {테이블명};
```

#### 상관 서브쿼리를 사용
```SQL
SELECT (SELECT COUNT(*) FROM {테이블명} AS T2 WHERE T2.{컬럼명} <= T1.{컬럼명}) AS 순번
FROM {테이블명} AS T1;
```

두 방법은 모두 기능적으로는 동일하지만 성능 측면에서는 윈도우 함수가 좋다.

윈도우 함수는 스캔 횟수가 1회이고 상관 서브쿼리는 스캔 횟수가 2회이기 때문이다.

### 8.1.2 키본 키가 여러 개의 필드로 구성되는 경우

#### 윈도우 함수를 사용
```SQL
SELECT ROW_NUMBER() OVER(ORDER BY {컬럼명1}, {컬럼명2}, ...) AS 순번
FROM {테이블명};
```

#### 상관 서브쿼리를 사용
```SQL
SELECT (
    SELECT COUNT(*) FROM {테이블명} AS T2 
    WHERE T2.{컬럼명1} < T1.{컬럼명1} OR (T2.{컬럼명1} = T1.{컬럼명1} AND T2.{컬럼명2} <= T1.{컬럼명2})) AS 순번
FROM {테이블명} AS T1;
```
- 다중 필드 비교를 이용한다.
- 필드가 여러 개인 경우에도 간단하게 확장할 수 있으며 필드 자료형을 원하는대로 지정할 수 있다.

### 8.1.3 그룹마다 순번을 붙이는 경우

#### 윈도우 함수를 사용
```SQL
SELECT ROW_NUMBER() OVER(PARTITION BY {컬럼명} ORDER BY {컬럼명}) AS 순번
FROM {테이블명};
```

#### 상관 서브쿼리를 사용
```SQL
SELECT (
    SELECT COUNT(*) FROM {테이블명} AS T2 
    WHERE T2.{컬럼명} = T1.{컬럼명} 
    AND T2.{컬럼명} <= T1.{컬럼명}) AS 순번
FROM {테이블명} AS T1;
```

## 8.1.4 순번과 갱신
검색이 아닌 갱신에서도 순번을 매기는 방법이 있다.

윈도우 함수를 사용하는 경우 순번 할당 쿼리를 SET 구로 넣을 수 있다.
만약 ROW_NUMBER를 사용하는 경우 서브쿼리를 사용해야 한다.

## 8.2 레코드에 순번 붙이기 응용

### 8.2.1 중앙값 구하기

중앙값이란 숫자를 정렬하고 양쪽 끝에서부터 수를 세는 경우 정중앙에 오는 값이다.
단순 평균과 다르게 아웃라이어에 영향을 받지 않는다는 장점이 있다.

#### 중앙값을 구하는 방법

```sql
SELECT AVG({컬럼명})
FROM (
    SELECT {컬럼명}
    FROM {테이블명1}, {테이블명2}
    GROUP BY {컬럼명}
    HAVING SUM(CASE WHEN {컬럼명} = {컬럼명} THEN 1 ELSE 0 END) >= COUNT(*) / 2
    AND SUM(CASE WHEN {컬럼명} = {컬럼명} THEN 1 ELSE 0 END) < COUNT(*) / 2) TMP;
```

이 방식에는 두 가지 단점이 있다.

1. 코드가 복잡하다
2. 성능이 좋지 않다

실행 계획을 살펴보면 자기 결합을 수행하게 된다. 이를 해결하는 방법이 윈도우 함수이다.

## 8.3 시퀀스 객체, IDENTITY 필드, 채번 테이블

### 8.3.1 시퀀스 객체

시퀀스 객체는 순번을 생성하는 객체이다. 시퀀스 객체는 다음과 같이 생성한다.

```sql
CREATE SEQUENCE {시퀀스명} START WITH {시작값} INCREMENT BY {증가값};
```

#### 시퀀스 객체의 문제점
1. 표준화가 늦어서 구현에 따라 구문이 달라 이식성이 없다.
2. 시스템에서 자동으로 생성되므로 실제 엔티티 속성이 아니다.
3. 성능 문제가 발생한다.

#### 시퀀스 성능 문제

시퀀스는 다음과 같은 세 가지 속성을 가진다.
- 유일성
- 연속성
- 순서성

이 속성을 만족하기 위해서는 Lock을 필요로 한다. 따라서 사용자가 시퀀스 객체를 사용하면 다른 사용자로부터의 접근을 블록하는 베타 제어를 수행한다.

#### 시퀀스 성능 문제 대처

시퀀스 객체의 성능 문제를 해결하기 위해서는 다음과 같은 방법이 있다.
- CACHE
- NOORDER

CACHE는 새로운 값이 필요할 때마다 메모리에서 읽어들일 필요가 있는 값의 수를 설정하는 옵션이다. 하지만 이 옵션은 부작용으로 시스템 장애가 발생할 때 연속성을 보장할 수 없다.

NOORDER는 순서성을 보장하지 않는 대신 오버 헤드를 줄일 수 있다.

#### 순번을 키로 사용할 때의 성능 문제

시퀀스 객체가 성능 문제를 일으키는 두 번째 경우는 'Hot Spot'과 관련이 있다.

이는 DBMS의 물리적인 저장 방식 때문에 발생하는데 순번처럼 비슷한 데이터를 연속적으로 저장하면 물리적으로 같은 영역에 저장된다.

이 때, 특정 물리 영역엠ㄴ I/O 부하가 커지므로 성능 악화가 발생한다. 이 부분을 'Hot Spot'이라고 한다.

#### 순번을 키도 사용할 때의 성능 문제 대처

두 가지 방법이 있다.
- Oracle 역 키 인덱스처럼 연속된 값을 도입하는 경우라도 DBMS 내부에서 변화를 주어 제대로 분산할 수 있는 구조를 사용하는 것이다. 예를 들어, 해시가 있다.
- 인덱스에 일부러 복잡한 필드를 추가해 데이터의 분산도를 높이는 것이다.

역키 인덱스와 같은 구조는 삽입구문의 속도를 빨라지지만 검색 속도가 느려진다.

### 8.3.2 IDENTITY 필드

IDENTITY 필드는 자동 순번 필드라고 부르며 테이블에 삽입이 이루어질 때마다 자동으로 순번을 붙여주는 기능이다.

IDENTITY 필드는 모든 방면에서 시퀀스 객체보다 심각한 성능 문제를 야기하므로 사용을 권장하지 않는다.

### 8.3.3 채번 테이블

과거에 사용하던 순번 테이블이다. 락 매커니즘을 사용하며 구시대의 유물이고 사용하지 않는다.