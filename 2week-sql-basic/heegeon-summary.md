# 2장. SQL 기초

---

## 2.1 SELECT 구문
### 2.1.1 SELECT 구와 FROM 구
- SELECT 구문에서 SELECT구는 필수이고, FROM 구는 필수가 아니다.
- FROM 구를 사용하지 않는 경우는 다음과 같다.
  - 상수를 출력하는 경우
  - 다른 SELECT 구문의 결과를 출력하는 경우
  - 테이블이 아닌 다른 객체를 출력하는 경우
- 하지만 Oracle에서는 FROM 구를 생략할 수 없다.
- RDB에서 불안정한 정보는 공란으로 취급한다.

### 2.1.2 WHERE 구
- WHERE 구에서는 다양한 연산자를 사용해 조건을 지정할 수 있다.
    - = : 같다
    - <> : 같지 않다
    - < : 작다
    - <= : 작거나 같다
    - \> : 크다
    - \>= : 크거나 같다

- WHERE 구에서는 다음과 같은 키워드로 조건을 지정할 수 있다.
  - BETWEEN A AND B : A와 B 사이에 있다
  - IS NULL : NULL이다
  - IS NOT NULL : NULL이 아니다
  - IN (A, B, ...) : A, B, ... 중 하나이다
  - LIKE : 문자열 패턴 매칭
  - AND : 논리곱
  - OR : 논리합
  - NOT : 논리 부정

### 2.1.3 GROUP BY 구
- GROUP BY 구는 특정 컬럼의 값이 같은 행을 하나로 묶어서 집계 함수를 사용할 수 있게 해준다.
- GROUP BY 구를 사용하면 집계 함수를 사용할 수 있다.

### 2.1.4 HAVING 구
- HAVING 구는 GROUP BY 구와 함께 사용한다.
- WHERE 구는 레코드에 조건을 지정한다면 HAVING은 집합에 조건을 지정하는 기능이다.

### 2.1.5 ORDER BY 구
- ORDER BY 구는 결과를 정렬한다.
- ORDER BY 구는 SELECT 구문의 마지막에 위치한다.
- ORDER BY 구는 다음과 같은 키워드로 정렬 순서를 지정할 수 있다.
  - ASC : 오름차순(기본)
  - DESC : 내림차순

### 2.1.6 뷰와 서브쿼리
- #### 뷰
  - 뷰는 SELECT 구문을 저장한 것이다.
  - 즉, 뷰는 테이블과 같은 역할을 하지만 데이터를 보유하지 않는다.
    ```sql
    CREATE VIEW {뷰이름} ({컬럼명1}, {컬럼명2}, ...) AS {SELECT 구문};
    ```
    > Q. p83.__뷰를 생성할 때 컬럼명을 지정하지 않으면 어떻게 될까?__
  - 위와 같이 저장하고 싶은 SELECT 구문을 이어서 작성함으로써 뷰를 생성할 수 있다.
    - 이렇게 생성된 뷰는 일반적인 테이블처럼 사용할 수 있다.
    ```sql
    SELECT {컬럼명1}, {컬럼명2}, ...
    FROM {뷰이름};
    ```
- #### 익명 뷰
  - 뷰는 사용 방법이 테이블과 같지만 내부에는 데이터를 보유하지 않는다.
  - 뷰 대신 FROM 구에 직접 SELECT 구문을 지정하여 사용할 수 있다.
    ```sql
    SELECT {컬럼명1}, {컬럼명2}, ...
    FROM ({SELECT 구문});
    ```
    > Q. p84.__익명 뷰와 서브 쿼리의 차이점이 무엇인가요?__

## 2.2 조건 분기, 집합 연산, 윈도우 함수, 갱신
### 2.2.1 CASE 표현식
- SQL에서 조건 분기를 수행하기 위해서 CASE 표현식을 사용한다.
- CASE 표현식은 "단순 CASE 표현식"과 "검색 CASE 표현식"으로 나뉘는데 검색 CASE 표현식은 단순 CASE 표현식을 포함하고 있다.
- 단순 CASE 표현식은 다음과 같이 사용한다.
  ```sql
  CASE 
    WHEN {평가식} THEN {결과1}
    WHEN {평가식} THEN {결과2}
    ...
    ELSE {결과n}
  END
  ```

### 2.2.2 SQL의 집합 연산
#### UNION 연산
- SQL에서는 UNION 연산을 사용하여 여러 테이블의 데이터를 하나로 합칠 수 있다.
- UNION 연산은 다음과 같이 사용한다.
  ```sql
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명1}
  UNION
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명2};
  ```
- UNION 연산은 중복된 레코드를 제거한다.
- UNION ALL 연산은 중복된 레코드를 제거하지 않는다.

#### INTERSECT 연산
- INTERSECT 연산은 두 테이블의 교집합을 구한다.
- INTERSECT 연산은 다음과 같이 사용한다.
  ```sql
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명1}
  INTERSECT
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명2};
  ```
- INTERSECT 연산은 중복된 레코드를 제거한다.

#### EXCEPT 연산
- EXCEPT 연산은 두 테이블의 차집합을 구한다.
- EXCEPT 연산은 다음과 같이 사용한다.
  ```sql
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명1}
  EXCEPT
  SELECT {컬럼명1}, {컬럼명2}, ...
  FROM {테이블명2};
  ```
- 다른 연산과 달리 EXCEPT 연산은 먼저 쓴 테이블이 기준이 되기 때문에 주의해야 한다.

### 2.2.3 윈도우 함수
- 윈도우 함수는 '집약 기능이 없는 GROUP BY'라고 생각하면 된다.
- 윈도우 함수는 다음과 같이 사용한다.
  ```sql
  SELECT {컬럼명1}, {컬럼명2}, ...
  {윈도우 함수} OVER ({윈도우 명세})
  FROM {테이블명};
  ```
- 윈도우 함수는 다음과 같은 종류가 있다. (등등)
  - `ROW_NUMBER()` : 윈도우에 속한 행의 순서를 반환한다.
  - `RANK()` : 윈도우에 속한 행의 순위를 반환한다.
  - `DENSE_RANK()` : 윈도우에 속한 행의 순위를 반환한다. 단, 순위가 같은 경우에는 같은 순위를 반환한다. 순위를 건너 뛰지 않는다.

### 2.2.4 트랜잭션과 갱신
#### 데이터 삽입
- INSERT 구문은 데이터의 기본적인 단위인 레코드를 삽입하는 구문이다.
- INSERT 구문은 다음과 같이 사용한다.
  ```sql
  INSERT INTO {테이블명} ({컬럼명1}, {컬럼명2}, ...)
  VALUES ({값1}, {값2}, ...);
  ```
- 문자형 자료형에는 작은따옴표를 붙여야 한다.

#### 데이터 제거
- DELETE 구문은 테이블에서 레코드를 제거하는 구문이다.
- DELETE 구문은 다음과 같이 사용한다.
  ```sql
  DELETE FROM {테이블명}
  WHERE {조건};
  ```
- 조건이 없으면 테이블의 모든 레코드가 제거된다.
- DELETE 구문에는 필드 이름을 넣어 특정 필드만 제거할 수는 없다.
- 예를 들어, 다음과 같은 구문은 에러가 발생한다.
  ```sql
  DELETE ({컬럼명1}, {컬럼명2}, ...) 
  FROM {테이블명} 
  WHERE {조건};
  
  또는 
  
  DELETE *
  FROM {테이블명}
  ```
  
#### 데이터 갱신
- UPDATE 구문은 테이블의 레코드를 갱신하는 구문이다.
- UPDATE 구문은 다음과 같이 사용한다.
  ```sql
  UPDATE {테이블명}
  SET {컬럼명1} = {값1}, {컬럼명2} = {값2}, ...
  WHERE {조건};
  
  또는
  
  UPDATE {테이블명}
  SET ({컬럼명1}, {컬럼명2}, ...) = ({값1}, {값2}, ...)
  WHERE {조건};
  ```
  