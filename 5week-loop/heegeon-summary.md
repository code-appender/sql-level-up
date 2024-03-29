# 5장. 반복문

---

## 5.1 반복문 의존증

SQL에는 설계에서부터 반복문이 제외되었다...

## 5.2 반복계의 공포

### 5.2.1 반복계의 단점

같은 기능을 구현할 때 반복계를 사용하면 포장계를 사용하는 것보다 더 많은 리소스를 사용한다.

레코드 수가 적을 때는 큰 차이가 없지만 반복계의 처리 시간은 선형적으로 증가하기 때문에 레코드 수가 많아질수록 처리 시간이 비례하여 증가한다.

이에 비해, 여러 레코드를 한 번에 처리하는 포장계는 처리 시간이 레코드 수에 비례하여 증가하지 않는다.

### 5.2.2 SQL 실행의 오버헤드
SQL을 실행할 때는 다음과 같은 연산 과정이 필요하다.

후처리
1. SQL 구문을 네트워크로 전송
2. 데이터베이스 연결
3. SQL 구문을 파싱
4. SQL 구문 실행 계획 생성 또는 평가

전처리
1. 결과 집합을 네트워크로 전송

2번 작업은 데이터베이스에 SQL 구문을 실행하기 위한 작업이다. 
하지만 최근에는 애플리케이션에서 미리 연결을 일정 수 확보해서 이런 오버헤드를 감소시키는 `커넥션 풀`이라는 기술을 사용한다.

오버헤드가 가장 큰 작업은 3번과 4번이다.

SQL 구문을 파싱하는 작업은 SQL 구문을 분석해서 실행 계획을 생성하는 작업이며 데이터베이스가 SQL문을 받을 때마다 수행하기 때문에 작은 SQL을 여러 번 실행하는 반복계에서 오버헤드가 높아진다.

반복계에서는 반복 1회의 처리를 단순화하기 때문에 리소스를 분산하여 병렬 처리하는 최적화가 어렵다. 즉, 리소스 사용 효율이 낮다.

### 5.2.3 데이터베이스의 진화로 인한 혜택을 받을 수 없다.
DBMS 버전이 올라가면서 성능이 향상되는 경우가 많다. 하지만 반복계에서는 이런 혜택을 받을 수 없다.
이유는 이 업그레이드의 내용은 대부분 "대규모 데이터를 다루는 복잡한 SQL 구문"의 성능 향상을 목표로 하기 때문이다.

포장계는 SQL을 잘 튜닝함으로써 성능을 향상시킬 수 있지만 반복계는 그렇지 않다.

### 5.2.3 반복계를 빠르게 만드는 방법
1. 반복계를 포장계로 다시 작성
2. 각각의 SQL을 빠르게 수정
3. 다중화 처리

이 중 다중화 처리가 가장 희망적이다. 처리를 다중화해서 성능을 선형에 가깝게 향상시킬 수 있다.

## 5.3 반복계의 장점

반복계의 가장 큰 장점은 SQL이 아주 단순하여 실행계획도 엄청 단순하다는 것이다.

### 5.3.1 실행 계획의 안정성

실행 계획이 단순하다는 것은 변동 위험이 거의 없다는 것이다. 이는 포장계의 단점이 될 수도 있다.

### 5.3.2 예상 처리 시간의 정밀도

실행 게획이 단순하다는 것은 예상 처리 시간을 정확하게 예측할 수 있다는 장점으로 이어진다. 

### 5.3.3 트랜잭션 관리의 용이성

반복계에서 특정 반복 회수마다 커밋한다고 가정할 때 중간에 커밋을 진행하기 때문에 오류가 발생해도 롤백하기 쉽다.

하지만 포장계에서는 이러한 트랜잭션 관리가 어렵다.

### 5.4.4 반복 횟수가 정해지지 않은 경우

반복 횟수가 정해지지 않은 경우에는 포인터 체인을 사용하여 처리할 수 있다. 이는 인접 리스트 모델로서 실제로 우체국에서 오래된 주소로 보내진 우편물을 새로운 주소로 전달하기 위해 사용하는 방법이다.

이 방법의 문제점은 몇 번을 올라가야 가장 오래된 주소를 찾을 수 있을 지 파악하기 어렵다는 것이다. 이 때, 절차 지향형 언어에서 반복문을 사용하면 쉽게 해결할 수 있다. 이를 '재귀 공통 테이블 식'이라고 한다.

### 5.4.5 중첩 집합 모델

SQL에서 계층 구조를 나타내는 방법은 크게 3가지가 있다.
1. 인접 리스트 모델
2. 중첩 집합 모델
3. 경로 열거 모델

이 중 중첩 집합 모델은 각 레코드의 데이터를 집합으로 보고 계층 구조를 집합의 중첩 관계로 나타내는 것이다. 