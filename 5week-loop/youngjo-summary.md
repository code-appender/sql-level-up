# 5장. 반복문

## 반복계의 공포

### 반복계의 단점

- 성능 비교
반복계로 구현한 코드 vs 포장계로 구현한 코드
-> 포장계 압승

레코드 수가 적을 때 : 큰 차이 X
레코드 수가 많을 때 : 반복계 - 선형 (레코드 수에 비례하여 처리 시간 증가) / 포장계 - 로그함수형

### SQL 실행의 오버헤드

후처리
1. SQL 구문을 네트워크로 전송
2. 데이터베이스 연결 -> `커넥션 풀`
3. SQL 구문을 파싱
4. SQL 구문 실행 계획 생성 또는 평가

전처리
1. 결과 집합을 네트워크로 전송

오버헤드가 가장 큰 작업 -> 3번과 4번

SQL 구문 파싱 : SQL 구문을 분석해서 실행 계획을 생성하는 작업.
              데이터베이스가 SQL문을 받을 때마다 수행하기 때문에 작은 SQL을 여러 번 실행하는 반복계에서 오버헤드가 높아진다.

### 병렬 분산이 힘들다

반복계에서는 반복 1회의 처리를 단순화하기 때문에 리소스를 분산하여 병렬 처리하는 최적화가 어렵다.

### 데이터베이스의 진화로 인한 혜택을 받을 수 없다.
DBMS 버전이 상승에 따라 옵티마이저는 효율적으로 실행 계획을 세우며 더 발전한다. 하지만 반복계는 이런 진화에 대한 혜택을 받을 수 없다.
-> '대규모 데이터를 다루는 복잡한 SQL 구문의 성능 향상'을 목표로 하기 때문

### 반복계를 빠르게 만드는 방법
1. 반복계를 포장계로 다시 작성 -> 애플리케이션의 수정. 사실상 어렵다.
2. 각각의 SQL을 빠르게 수정 -> 반복계에서 사용하는 SQL 구문은 너무 단순해서 튜닝해야할 부분이 마땅치 않다.
3. 다중화 처리 -> 가장 희망적인 선택지. 리소스에 여유가 있고 처리를 나눌 수 있는 키가 있다면 선형에 가깝게 스케일 가능

## 반복계의 장점

SQL이 단순하여 실행 계획도 엄청 단순하다는 것

### 실행 계획의 안정성

변동 위험이 거의 없다. = 포장계의 단점
안정적인 성능을 확보할 수 있다는 점에서는 어마어마한 장점.

### 예상 처리 시간의 정밀도

예상 처리 시간을 정확하게 예측할 수 있다.

### 트랜잭션 제어가 편리

트랜잭션의 정밀도를 미세하게 제어할 수 있다.
ex) 특정 반복 횟수마다 커밋한다면 중간에 커밋을 진행하기 때문에 오류가 발생해도 최근 커밋 시점으로 롤백하기 쉽다.

## SQL에서는 반복을 어떻게 표현할까?

### 포인트는 CASE 식과 윈도우 함수
CASE = IF-THEN-ELSE
SQL에서도 CASE 식과 윈도우 함수는 세트이다.

### 반복 횟수가 정해지지 않은 경우

1. 인접 리스트 모델과 재귀 쿼리
2. 중첩 집합 모델