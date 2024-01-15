# 2장 : SQL 기초

## 6장 : Select 구문

검색 == 질의 == query

### Select 구

어떤 방법으로 선택할지 일절 쓰여 있지 않다. 이 방법은 DBMS에게 맡긴다.

사용자는 단지 어떤 데이터가 필요한지만 알린다.

<aside>
💡 NULL
불명한 데이터를 `공란` 으로 취급한다.

</aside>

### From 구

from [테이블 이름]

테이블 이름이 적히지 않아도 되는경우는 `select 1`  과 같은 상수 조회의 경우이다.

### Where 구

추가적인 조건을 지정할 수 있는 방법이다.

~라는 경우를 나타내는 관계 부사 이다.

사용 연산자

| 연산자    | 의미      |
|--------|---------|
| ' = '  | ~와 같음   |
| ' <> ' | ~와 같지 않음 |
| ' >= ' | ~이상     |
| ' > '  | ~보다 큼 |

```sql
SELECT name, adderss, age
From address
Where address = '서울시' AND age >= 30; -- or도 가능함
```

IN, OR 조건 간단하게 작성

```sql
-- 성별이 '남성'이거나 '여성'인 사용자 검색 (IN 사용)
SELECT * FROM 사용자
WHERE 성별 IN ('남성', '여성');
```

```sql
-- 성별이 '남성'이거나 '여성'인 사용자 검색 (IN 사용)
SELECT * FROM 사용자 WHERE 성별 = '남성' OR 성별 = '여성';
```

null조회 예시

```sql
-- 사용자 테이블 예제 (이름과 이메일은 NULL 허용)
CREATE TABLE 사용자 (
    사용자_ID INT PRIMARY KEY,
    이름 VARCHAR(50),
    이메일 VARCHAR(100)
);

-- 사용자 데이터 삽입 (일부 사용자는 이름 또는 이메일이 NULL)
INSERT INTO 사용자 VALUES
(1, '홍길동', 'hong@example.com'),
(2, '김철수', NULL),
(3, NULL, 'lee@example.com'),
(4, '박미영', NULL),
(5, '정기석', 'jung@example.com');

-- 이름이 NULL인 사용자 검색
SELECT * FROM 사용자
WHERE 이름 IS NULL;

-- 이메일이 NULL인 사용자 검색
SELECT * FROM 사용자
WHERE 이메일 IS NULL;

-- 이름 또는 이메일 중 하나라도 NULL인 사용자 검색
SELECT * FROM 사용자
WHERE 이름 IS NULL OR 이메일 IS NULL;
```

<aside>
💡 SELECT 구문은 절차 지향형 언어의 함수

</aside>

SELECT구문은 일종의 읽기 전용의 함수
입력과 출력 자료형 : 테이블(관계) 이외에는 어떤 자료형도 없다 → 관계가 닫혀있다는 의미의 미로 폐쇄성이라 한다.

### Group by

데이터

```sql
-- 주문 테이블 예제
CREATE TABLE 주문 (
    주문_ID INT PRIMARY KEY,
    사용자_ID INT,
    금액 DECIMAL(10, 2),
    주문일 DATE
);

-- 주문 데이터 삽입
INSERT INTO 주문 VALUES
(1, 1, 100.50, '2022-01-05'),
(2, 2, 75.20, '2022-01-08'),
(3, 1, 50.00, '2022-01-10'),
(4, 3, 120.75, '2022-01-12'),
(5, 2, 30.90, '2022-01-15');
```

```sql
-- 사용자별 주문 금액의 합계를 조회
SELECT 사용자_ID, SUM(금액) AS 총주문금액
FROM 주문
GROUP BY 사용자_ID;

-- 주문일별 주문 건수를 조회
SELECT 주문일, COUNT(*) AS 주문건수
FROM 주문
GROUP BY 주문일;

-- 사용자별 주문 금액의 평균을 조회하되, 평균이 70보다 큰 경우만 출력
SELECT 사용자_ID, AVG(금액) AS 평균주문금액
FROM 주문
GROUP BY 사용자_ID
HAVING 평균주문금액 > 70;
```

이러한 그룹을 통해서 집계가 가능하다.

1. **COUNT():** 그룹 내에서 행의 수를 계산합니다.

    ```sql
    SELECT 부서, COUNT(*) AS 직원수
    FROM 직원
    GROUP BY 부서;
    
    | 부서    | 직원수 |
    |--------|------|
    | 인사   | 3    |
    | 영업   | 2    |
    | 개발   | 4    |
    ```

2. **SUM():** 그룹 내에서 특정 열의 합을 계산합니다.

    ```sql
    SELECT 부서, SUM(급여) AS 총급여
    FROM 직원
    GROUP BY 부서;
    
    | 부서    | 총급여   |
    |--------|--------|
    | 인사   | 20000  |
    | 영업   | 15000  |
    | 개발   | 30000  |
    ```

3. **AVG():** 그룹 내에서 특정 열의 평균을 계산합니다.

    ```sql
    SELECT 부서, AVG(평점) AS 평균평점
    FROM 프로젝트
    GROUP BY 부서;
    
    | 부서    | 평균평점 |
    |--------|------|
    | 영업   | 4.2  |
    | 개발   | 4.6  |
    
    ```

4. **MIN():** 그룹 내에서 특정 열의 최소값을 계산합니다.

    ```sql
    SELECT 부서, MIN(나이) AS 최소나이
    FROM 직원
    GROUP BY 부서;
    
    | 부서    | 최소나이 |
    |--------|--------|
    | 인사   | 25     |
    | 영업   | 28     |
    | 개발   | 30     |
    ```

5. **MAX():** 그룹 내에서 특정 열의 최대값을 계산합니다.

    ```sql
    SELECT 부서, MAX(경력년수) AS 최대경력
    FROM 직원
    GROUP BY 부서;
    
    | 부서    | 최대경력 |
    |--------|--------|
    | 인사   | 8      |
    | 영업   | 6      |
    | 개발   | 10     |
    ```


Group by()의 ()는 키를 지정하지 않는다는 뜻이다.

보통은 생략할 수 있어 생략하지만 일반적인 코드를 볼때 Group by()이 있다고 생각하고 보는것이 좋다.

⇒ GROUP BY () 자르는 기준이 없다 라고 보는것이 좋다. (일부 DBMS 는 지원하지 않는다)

### Having 구

where구가 레코드에 조건을 지정한다면 having은 집합에 조건을 지정하는 기능이다.

```sql
-- 부서별로 평균 연봉이 50000 이상인 부서를 찾는 쿼리
SELECT 부서, AVG(연봉) AS 평균연봉
FROM 직원
GROUP BY 부서
HAVING 평균연봉 >= 50000;
```

### ORDER BY 구

select만 하는 경우는 순서가 없이 엉터리로 출력한다. (DBMS마다 다르다)

따라서 명시적으로 순서를 지정해줘야한다.

```sql
-- 사용자 테이블을 부서명으로 오름차순으로 정렬하는 쿼리
SELECT *
FROM 사용자
ORDER BY 부서 ASC;
```

```sql
-- 부서별로 평균 연령이 30세 이상이고, 부서명으로 내림차순으로 정렬하는 쿼리
SELECT 부서, AVG(나이) AS 평균연령
FROM 사용자
GROUP BY 부서
HAVING 평균연령 >= 30
ORDER BY 부서 DESC;
```

### 뷰와 서브쿼리

자주 사용하는 SELECT구문은 텍스트 파일에 따로 저장해 둬도 좋을것이다. 이렇게 SELECT구문을 데이터베이스 안에 저장하는것이 뷰이다.

하지만 테이블과 달리 내부에 데이터를 보유하지는 않고 뷰는 단순히 SELECT구문을 저장한것 뿐이다.

뷰 생성 방법

```sql
CREATE VIEW [뷰이름]([필드이름1],[필드이름2] ...) AS 

-- 뷰 생성: 연봉이 70000 이상인 사용자만을 조회하는 뷰
CREATE VIEW 뷰_고연봉사용자 AS
SELECT * FROM 사용자
WHERE 연봉 >= 70000;

-- 생성한 뷰 조회
SELECT 연봉 FROM 뷰_고연봉사용자;
```

익명 뷰

```sql
-- 익명 뷰를 사용한 조회: 연봉이 70000 이상인 사용자만을 조회
SELECT * FROM (
    SELECT * FROM 사용자
    WHERE 연봉 >= 70000
) AS 익명뷰;

-- 명명된 뷰를 정의하기보다는 즉석에서 필요한 일회성 작업에 유용
```

From 구에 직접 지정하는 SELECT 구문을 서브쿼리 라고 부른다.

서브쿼리를 매개변수로 사용하는 예제 코드

```sql
-- 부서별로 평균 연봉이 높은 사용자 조회
SELECT *
FROM 사용자
WHERE 연봉 > (
    SELECT AVG(연봉)
    FROM 사용자
    GROUP BY 부서
    HAVING 부서 = '개발'
);
```

## 7장

### CASE

작동은 switch 조건문과 거의 유사하다.

when 구의 평가식 부터 평가되고 조건이 맞으면 THEN 구에 지정식이 리턴되면 CASE 식 전체가 종료된다.

만약 조건이 맞지 않으면 다음 WHEN구로이동해 같은 처리를 반복한다.

단순 CASE ≤ 검색 CASE 이다.

```sql
SELECT 
    이름,
    CASE 
        WHEN 소속도 LIKE '서울%' THEN '수도권'
        WHEN 소속도 LIKE '경기%' THEN '수도권'
        WHEN 소속도 LIKE '인천%' THEN '수도권'
        WHEN 소속도 LIKE '부산%' THEN '경상도'
        WHEN 소속도 LIKE '대구%' THEN '경상도'
        WHEN 소속도 LIKE '광주%' THEN '전라도'
        WHEN 소속도 LIKE '대전%' THEN '충청도'
        WHEN 소속도 LIKE '울산%' THEN '경상도'
        ELSE '기타'
    END AS 대규모지역
FROM 기도자;
```

이 코드는 일종의 교환 코드

CASE는 SELECT, WHERE, GROUP BY, HAVING, ORDER BY구와 같은 어디에서나 사용가능하다.

### UNION

단순 union ⇒ 중복된 레코드가 생략된다.

```sql
SELECT *
FROM Address 
UNION
SELECT *
FROM ADddress2 
```

만약 중복제거를 원하지 않는다.

```sql
SELECT *
FROM Address 
UNION ALL
SELECT *
FROM ADddress2 
```

### INTERSECT

교집합을 구한다. 중복된것은 제거된다.

```sql
SELECT *
FROM Address 
INTERSECT
SELECT *
FROM Address2
```

### EXCEPT

차집합을 구한다, 순서가 중요하다.

```sql
SELECT *
FROM Address
EXCEPT 
SELECT *
FROM Address2
```

### 윈도우 함수

집약 기능이 없는 GROUP BY구 이다.

GROUP BY가 자르기 + 집약이였다면

윈도우 함수는 자르기 기능만 있다.

윈도우 함수는 주로 윈도우(범위) 내에서 행 간의 계산을 수행하는 데 사용됩니다.

예를 들어, 각 행에 대해 이전 행들과의 비교, 누적 합산 등을 계산할 때 유용합니다.

PARTITION BY라는 구로 수행된다.

```sql
SELECT 
    사용자_ID,
    이름,
    부서,
    급여,
    AVG(급여) OVER (PARTITION BY 부서) AS 부서별평균급여
FROM 사용자;
```

```sql
-- 윈도우 함수를 사용하여 각 부서에서 사용자 수를 계산하는 예제 (비현실적인 사용) -> 주로 group by

SELECT
    department,
    COUNT(*) OVER (PARTITION BY department) AS "Number of Users by Department"
FROM users;
```

```sql
-- 윈도우 함수를 사용하여 각 부서에서 총 급여를 계산하는 예제 (비현실적인 사용) -> 주로 group by
SELECT
    department,
    SUM(salary) OVER (PARTITION BY department) AS "Total Salary by Department"
FROM users;
```

```sql
-- RANK()를 사용하여 급여 순위를 매기는 쿼리
-- 1,2,3,4,5 (동일한것에 대해 동일 급여를 매기고 등수를 건너뛴다 1,2,2,4,4)
SELECT
    이름,
    급여,
    RANK() OVER (ORDER BY 급여 DESC) AS 급여순위
FROM 사용자;
```

```sql
-- ROW_NUMBER()를 사용하여 급여에 따라 행 번호를 부여하는 쿼리
-- 1,2,3,4,5 (동일한것에 대해 유일한 순서를 매긴다. 동일 값이여도 고유한 순서를 매긴다 1,2,3,4,5)
SELECT
    이름,
    급여,
    ROW_NUMBER() OVER (ORDER BY 급여 DESC) AS 행번호
FROM 사용자;
```

```sql
-- DENSE_RANK()를 사용하여 급여 순위를 밀집순위로 매기는 쿼리 
-- 1,2,3,4,5 (동일한것에 대해 동일 급여를 매기고 등수를 건너뛰지 않는다 1,2,2,3,3과 같이 한다
SELECT
    이름,
    급여,
    DENSE_RANK() OVER (ORDER BY 급여 DESC) AS 밀집순위
FROM 사용자;
```

### 트랜잭션과 갱신

SQL은 쿼리를 위한 언어이기에 데이터를 갱신하는것은 부가적인 기능이다.

### insert(삽입)

값은 순서대로 제공해야한다. (데이터 형식등의 오류가 발생할 수 있다)

NULL을 삽입하는 경우 그대로 NULL을 입력한다. (따옴표로 감싸면  문자열로 인식된다)

```sql
INSERT INTO 테이블명 (열1, 열2, ...)
VALUES (값1, 값2, ...);
```

여러개의 줄을 함꺼번에 입력하는 기능을 지원하기도 한다( multi-row insert)

```sql
-- 사용자 테이블에 여러 사용자를 동시에 삽입하는 예제
INSERT INTO 사용자 (이름, 부서, 급여)
VALUES
    ('김철수', '영업', 70000),
    ('이영희', '인사', 75000),
    ('박미영', '영업', 82000);
```

모든 DBMS를 지원하지는 않을 수 있다.

오류가 발생했을때 찾기가 어렵다.

### delete(제거)

```sql
DELETE FROM 테이블명
WHERE 조건;
```

```sql
DELETE [특정 필드] FROM [테이블 이름] --오류가 발생한다. 특정 필드만 제거할 수 없다.
DELETE * FROM [테이블 이름] -- 오류 발생
-- 특정 필드만 지우고 싶다면 UPDATE사용
-- 테이블 이라는 상자는 남아있으므로 계속해서 데이터는 사용할 수 있다. 
```

### update(갱신)

등록된 데이터를 변경

```sql
-- 두 개의 필드를 별도로 나열하여 업데이트하는 예제
UPDATE 테이블명
SET 열1 = 값1, 열2 = 값2
WHERE 조건;

-- 괄호로 묶어서 필드를 나열하여 업데이트하는 예제
UPDATE 테이블명
SET (열1, 열2) = (값1, 값2)
WHERE 조건;
```
