# 6. 결합

# 6.1 기능적 관점으로 구분하는 결합의 종류

- 기능적인 관점으로 분류한 것
    - 크로스 결합
    - 내부 결합
    - 외부 결합
- 등가 결합, 비등가 결합은 결합 조건으로 등호(=)를 사용하는지, 이 이외의 부등호(>, ≥ 등)를 사용하는지의 차이를 의미한다.

## 6.1.1 크로스 결합 - 모든 결합의 모체

- 실무에서 사용할 기회 거의X

### 크로스 결합의 작동

```sql
SELECT *
 FROM Employees
		   CROSS JOIN
			Departments;
```

- 회사 테이블의 레코드 6개와 부서 테이블의 레코드 4개를 곱하는 것(24개 레코드 나온다)

### 크로스 결합이 실무에서 사용되지 않는 이유

- 이러한 결과가 필요한 경우가 없다.
- 비용이 매우 많이 드는 연산이다.

## 6.1.2 내부 결합 - 왜 ‘내부’라는 말을 사용할까?

### 내부 결합의 작동

- 내부는 ‘다크르트 곱의 부분 집합’이라는 의미다.
- 크로스 결합으로 결과를 내고 결합 조건으로 필터링 하는 것 → 이런 무식한 알고리즘 사용X
- 처음부터 결합 대상을 최대한 축소하는 형태로 작동한다.

## 6.1.3 외부 결합 - 왜 ‘외부’라는 말을 사용할까?

- 데카르트 곱의 부분 집합이 아니다라는 의미이다.
- 한쪽 테이블의 결과는 조인되지 않아도 모두 갖고있다.

# 6.2 결합 알고리즘과 성능

- 옵티마이저가 선택 가능한 결합 알고리즘은 세 가지다.
    - Nested Loops
    - Hash
    - Sort Merge
- 옵티마이저가 어떤 알고리즘을 선택할지 여부는 데이터 크기 또는 결합 키의 분산이라는 요인에 의존한다.
- 가장 빈번하게 볼 수 있는 알고리즘은 Nested Loops로, 각종 결합 알고리즘의 기본이 되는 알고리즘이다.
- 다음으로 중요한 것은 Hash, Sort Merge는 중요성이 한 단계 떨어진다.

## 6.2.1 Nested Loops

- Nested Loops는 이름 그대로 중첩 반복을 사용하는 알고리즘이다.
- 세부 처리
    - 결합 대상 테이블에서 레코드를  하나씩 반복해가며 스캔한다. 이 테이블을 구동 테이블 또는 외부 테이블이라고 부른다. 다른 테이블은 내부 테이블이라고 부른다.
    - 구동 테이블의 레코드 하나마다 내부 테이블의 레코드를 하나씩 스캔해서 결합 조건에 맞으면 리턴한다.
    - 이러한 작동을 구동 테이블의 모든 레코드에 반복한다.
- 특징
    - 접근되는 레코드 수는 R(A) x R(B)가 된다. Nested Loops의 실행 시간은 이러한 레코드 수에 비례한다.
    - 한 번의 단계에서 처리하는 레코드 수가 적으므로 Hash 또는 Sort Merge에 비해 메모리 소비가 적다.
    - 모든 DBMS에서 지원한다.

### 구동 테이블의 중요성

- `내부 테이블의 결합 키 필드에 인덱스가 존재한다면`, 구동 테이블로는 작은 테이블을 선택해야 한다.

### Nested Loops의 단점

- 결국 절대적인 양이 너무 많으면 반복이 많이 일어난다. 즉, 지연이 일어난다는 것이다.

## 6.2.2 Hash

### Hash의 작동

- 해시 결합은 일단 작은 테이블을 스캔하고, 결합 키에 해시 함수를 적용해서 해시값으로 변환한다.
- 이어서 다른 테이블(큰 테이블)을 스캔하고, 결합 키가 해시값에 존재하는지를 확인하는 방법으로 결합을 수행한다.
- 작은 테이블에서 해시 테이블을 만드는 이유는, 해시 테이블은 DBMS의 워킹 메모리에 저장되므로 조금이라도 작은 것이 효율적이기 때문이다.

### Hash의 특징

- 결합 테이블로부터 해시 테이블을 만들어서 활용하므로, Nested Loops에 비해 메모리를 크게 소모한다.
- 메모리가 부족하면 저장소를 사용하므로 지연이 발생한다.
- 출력되는 해시값은 입력값의 순서를 알지 못하므로, 등치 결합에만 사용할 수 있다.

### Hash과 유용한 경우

- Nested Loops에서 적절한 구동 테이블(상대적으로 충분히 작은 테이블)이 존재하지 않는 경우
- 앞서 ‘Nested Loops의 단점’에서 본 것처럼 구동 테이블로 사용할만한 작은 테이블은 있지만, 내부 테이블에서 히트되는 레코드 수가 너무 많은 경우
- Nested Loops의 내부 테이블에 인덱스가 존재하지 않는(또는 여러 가지 사정에 의해 인덱스를 추가할 수 없는) 경우
- 한마디로 말하자면, Nested Loops가 효율적으로 작동하지 않는 경우의 차선책이 Hash다.

### 주의해야 하는 트레이드오프

- 초기에 해시 테이블을 만들어야 하므로, Nested Loops에 비해 소비하는 메모리 양이 많다는 것이다. 따라서 동시 실행성이 높은 OLTP 처리를 할 때 Hash가 사용되면, DBMS가 사용할 수 있는 메모리가 부족해져 저장소가 사용된다.
- Hash 결합은 반드시 양쪽 테이블의 레코드를 전부 읽어야 하므로, 테이블 풀 스캔이 사용되는 경우가 많다. 따라서 테이블의 규모가 굉장히 크다면, 이런 풀 스캔에 걸리는 시간도 고려해야 한다.

## 6.2.3 Sort Merge

### Sort Merge의 작동

- Sort Merge는 간단하게 Merge 또는 Merge join이라 부르기도 합니다.
- Sort Merge는 결합 대상 테이블들을 각각 결합 키로 정렬하고, 일치하는 결합 키를 찾으면 결합한다.

### Sort Merge의 특징

- 대상 테이블을 모두 정렬해야 하므로 Nested Loops보다 많은 메모리를 소비한다. Hash와 비교하면 규모에 따라 다르지만, Hash는 한쪽 테이블에 대해서만 해시 테이블을 만드므로 Hash보다 많은 메모리를 사용하기도 한다. 메모리 부족으로 TEMP 탈락이 발생하면 I/O 비용이 늘어나고 지연이 발생할 위험이 있습니다(이는 Hash와 마찬가지다.)
- Hash와 다르게 동치 결합뿐만 아니라 부등호(<, >, ≤, ≥)를 사용한 결합에도 사용할 수 있다. 하지만 부정 조건(<>) 결합에서는 사용할 수 없다.
- 원리적으로 테이블이 결합 키로 정렬되어 있다면 정렬을 생략할 수 있다. 다만 이는 SQL에서 테이블에 있는 레코드의 물리적인 위치를 알고 있을 때다. 따라서 이러한 생략은 구현 의존적이다.
- 테이블을 정렬하므로 한쪽 테이블을 모두 스캔한 시점에 결합을 완료할 수 있다.

### Sort Merge가 유효한 경우

- Sort Merge 결합 자체에 걸리는 시간은 결합 대상 레코드 수가 많더라고 나쁘지 않은 편이지만, 테이블 정렬에 많은 시간과 리소스를 요구할 가능성이 있다.

## 6.2.4 의도하지 않은 크로스 결합

- 의도하지 않게 크로스 결합이 나타나는 경우 → ‘삼각 결합’

```sql
SELECT A.col_a, B.col_b, C.col_c
	FROM Table_A A
			 INNER JOIN Table_B B
					ON A.col_a = B.col_b
			 INNER JOIN Table_C C
					ON A.col_a = C.col_c;
```

결합 조건은 ‘Table_A - Table_B’와 ‘Table_A - Table_C’에만 존재한다.

‘Table_B - Table_C’에는 결합 조건이 존재하지 않는다는 점이 포인트다.

- A를 구동 테이블로 B와 결합하고 그 결과를 C와 결합
- A를 구동 테이블로 C와 결합하고 그 결과를 B와 결합
- B를 구동 테이블로 A와 결합하고 그 결과를 C와 결합
- C를 구동 테이블로 A와 결합하고 그 결과를 B와 결합

### Nested Loops가 선택되는 경우

- A를 구동 테이블로 B와 겨랍하고 그 결과를 C와 결합하는 순서로 Nested Loops를 사용해 결합하는 것이다.

### 크로스 결합이 선택되는 경우

- B와 C처럼 결합 조건이 없는 경우, 결합 조건이 없는 테이블들을 크로스 결합으로 결합해버리는 경우가 있다.
- 이는 ‘테이블B와 테이블C를 먼저 결합하고 그 결과를 테이블A와 결합’하는 순서로 결합을 수행한다.
- 그런데 앞에서 말했던 것처럼 테이블B와 테이블C사이에는 결합 조건이 없으므로 크로스 결합할 수 밖에 없다.
- 옵티마이저가 테이블B와 테이블C의 크기를 작다고 평가해 작은 테이블을 먼저 결합하였기 때문이다.

### 의도하지 않은 크로스 결합을 회피하는 방법

- 결합 조건이 존재하지 않는 테이블 사이에 불필요한 결합 조건을 추가해주는 방법이 있다.
- 현재 상황에서는 테이블B와 테이블C 사이에 결합 조건을 설정해준다는 것이다.
- 이 방법은 테이블B와 테이블C 사이에 결합 조건을 설정할 수 있고, 결합 조건을 설정해도 결과에 아무런 영향을 주지 않는 경우에만 사용할 수 있다.

```sql
SELECT A.col_a, B.col_b, C.col_c
 FROM TABLE_A A
			INNER JOIN Table_B B
							ON A.col_a = B.col_b
			INNER JOIN Table_C C
							ON A.col_a = C.col_c
						 AND C.col_c = B.col_b; -- Table_B와 Table_C의 결합 조건
```

# 6.3 결합이 느리다면

## 6.3.1 상황에 따른 최적의 결합 알고리즘

| 이름 | 장점 | 단점 |
| --- | --- | --- |
| Nested Loops | 1. (작은 구동테이블 + 내부 테이블의 인덱스) 라는 조건이 충족된다면 빠르다.
2. 메모리 또는 디스크 소비가 적으므로 OLTP에 적합3. 비등가 결합에서도 사용 가능 | 1. 대규모 테이블들의 결합에는 부적합
2. 내부 테이블의 인덱스가 사용되지 않거나, 내부 테이블의 선택률이 높으면 느리다. |
| Hash | 1. 대규모 테이블들을 결합할 때 적합 | 1. 메모리 소비량이 큰 OLTP에 부적합
2. 메모리 부족이 일어나면 TEMP 탈락 발생
3. 등가 결합에서만 사용 가능 |
   | Sort Merge | 1. 대규모 테이블들을 결합할 때 적합
2. 비등가 결합에서도 사용 가능 | 1. 메모리 소비량이 큰 OLTP에 부적합
2. 메모리 부족이 일어나면 TEMP 탈락 발생
3. 데이터가 정렬되어 있지 않다면 비효율적 |
- 소규모 - 소규모
    - 결합 대상 테이블이 작은 경우에는 어떤 알고리즘을 사용해도 성능 차이가 크지 않다.
- 소규모 - 대규모
    - 소규모 테이블을 구동 테이블로 하는 Nested Loops를 사용한다.
    - 대규모 테이블의 결합 키에 인덱스를 만들어주는 것을 잊지 말자!
    - 하지만 내부 테이블의 결합 대상 레코드가 너무 많다면 구동 테이블과 내부 테이블을 바꾸거나, Hash를 사용해볼 것을 검토
- 대규모 - 대규모
    - 일단 Hash를 사용한다. 결합 키로 처음부터 정렬되어 있는 상태라면 Sort Merge를 사용한다.
- 기본적으로 ‘일단 Nested Loops, 잘 안되면 Hash’ 라는 순서로 기억하자.