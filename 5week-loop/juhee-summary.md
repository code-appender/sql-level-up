
# 5장 : 반복문

## 15장 반복계의 공포

### 반복계의 단점 : 성능

처리해야하는 레코드 수가 많아질 수록 성능 차이가 벌어진다.

반복계는 포장게에 성능적으로 질 수밖에 없다.

그럴수 밖에 없는 이유

1. SQL 실행의 오버헤드
    1. 전처리
        1. SQL구문을 네트워크로 전송
        2. 데이터 베이스 연결
        3. SQL 구문 파스
        4. SQL 구문의 실행 계획 생성, 평가
    2. 후처리
        1. 결과 집합을 네트워크로 전송

   ⇒ 이런 이유로 비용이 얼마나 들어갈지 몰라 작은것 100개보단 큰거 1개를 선호한다.

2. 병렬 분산이 힘들다 .
    1. 반복계는 리소스 사용 효율이 나쁘다.
3. 데이터 베이스의 진화로 인한 혜택을 받을 수 없다.
    1. DBMS의 버전이 올르 수록 옵티마이저는 보다 효율적으로 실행 계획을 세우고, 데이터에 고속으로 접근할 수 있는 아키텍처를 구현한다.

포장계의 SQL은 반복계에 비해 복잡하다.

하지만 포장계의 SQL구문은 튜닝 가능성이 굉장히 높으므로 제대로 튜닝한다면 처음과 비교하여 현격한 성능 차이가 있을것입니다.

하지만 반복계는 느리고 튜닝가능성도 거의 없다 ( 치명적 단점 )

### 반복계를 빠르게 만드는 법

- 반복계를 포장계로 다시 작성
    - 하지만 실제 해당 방법을 사용하는것은 어렵다.
- 각각의 SQL을 빠르게 수정
    - 튜닝 가능성이 제한된다.
- 다중화 처리
    - 가장 희망적 선택지
    - CPU또는 디스크 같은 리소스에 여유가 있고 처리를 나눌 수 있는 키가 명확하다면 처리를 다중화 해서 성능을 선형에 가깝게 스케일 할 수 있습니다.

### 반복계의 장점

1. 실행 계획의 안정성
    1. 실행 계획이 단순하다 == 핻당 실행 계회의 변동성이 거의 없ㄷ.
2. 예상 처리시간의 정밀도
    1. 처리시간 = 한번의 실행 시간 * 실행 횟수
3. 트랜잭션 제어가 편리
    1. 트랜잭션의 정밀도를 세세하게 제어할 수 있다.
    2. 중간까지 실행한 이후에 이어서 할 수 있다.(문제 발생 이후부터 다시 실행 가능, 포장계는 불가능)

## 16장 SQL에서는 반복문을 어떻게 표현할까

(코드 위주, 책 참고)

### 1. 포인트는 CASE식과 윈도우 함수

### 2. 최대 반복 횟수가 정해진 경우
코드 5-8은 FROM 절 내부 부터 설명하겠습니다.
첫 번째 CASE WHEN에서 pcode에 따른 hit_code를 계산합니다. 이후 윈도우 함수중 MIN함수를 통해 hit_code값들중의 최소값을 찾습니다. 이후 over 절에서 hit_code를 순서로 정렬하여 윈도우를 생성합니다.
이렇게 한 이후 외부 쿼리에 의해 hit_code 와 같은 min_code를 찾아 값을 출력합니다.
즉 이렇게 하여 각 우체번호에 따라 코드를 부여하고 코드중 가장 작은 값에 대해서만 같은 데이터를 출력하도록 합니다.(즉 가장 작은 값이 여러개인 경우는 여러개의 로우를 출력합니다)
즉 이렇게 하여 스캔 횟수 줄이고 원하는 우편번호와 가장 가까운 우편번호를 출력할 수 있게 됩니다.  
````sql
SELECT pcode, district_name
FROM (
  SELECT 
    pcode,
    district_name,
    CASE 
      WHEN pcode = '1234' THEN 0
      WHEN pcode LIKE '123%' THEN 1
      WHEN pcode LIKE '12%%' THEN 2
      WHEN pcode LIKE '1#%%' THEN 3
      ELSE NULL 
    END AS hit_code,
    MIN(
      CASE 
        WHEN pcode = '1234' THEN 0
        WHEN pcode LIKE '123%' THEN 1
        WHEN pcode LIKE '12%%' THEN 2
        WHEN pcode LIKE '1#%%' THEN 3
        ELSE NULL 
      END
    ) OVER (ORDER BY 
      CASE 
        WHEN pcode = '1234' THEN 0
        WHEN pcode LIKE '123%' THEN 1
        WHEN pcode LIKE '12%%' THEN 2
        WHEN pcode LIKE '1#%%' THEN 3
        ELSE NULL 
      END
    ) AS min_code
  FROM PostalCode
) AS FOO
WHERE hit_code = min_code;
````
### 3. 반복 횟수가 정해지지 않은 경우

1. 인접 리스트 모델과 재귀 쿼리
2. 중첩 집합 모델
    1. 인접 리스트 모델
    2. 중첩 집합 모델
    3. 경로 열거 모델

### 17장 바이어스의 공죄

SQL은 ‘절차 지향형에서의 탈출’을 목표로 설계된 언어이다.

하지만 이러한 절차적 계층을 은폐하는것이 SQL의 이념이다.

하지만 정말로 RDB에서 고성능을 실현하고 싶다면 절차 지향적인 바이어스를 때어내고 자유로워질 필요가 있다.

반복계와 포장계의 장단점을 파악하고 어떤 처리 방식을 채택할지를 냉정하게 판단해야한다.

SQL이 가진 강력한 도구와 튜닝방법을 활용하려면 집합 지향의 사고방식을 가져야한다.