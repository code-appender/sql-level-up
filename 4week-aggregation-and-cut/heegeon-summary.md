# 3장. 집약과 자르기

---

## 3.1 집약

### 3.1.1 집약 함수
집약 함수에는 다음과 같이 5가지가 있다.
- COUNT : 행의 개수를 센다.
- SUM : 합계를 구한다.
- AVG : 평균을 구한다.
- MAX : 최댓값을 구한다.
- MIN : 최솟값을 구한다.

### 3.1.2 CASE 식과 GROUP BY 응용
GROUP BY로 집약했을 때 SELECT 구에서 입력할 수 있는 것은 상수, 집약 함수, GROUP BY로 그룹화한 컬럼명 뿐이다. 
GROUP BY로 그룹화한 컬럼들에서 공통된 값들을 추출할 때 여러 요소가 존재한다면 에러가 발생할 수 있다.

이럴 경우, 집약 함수를 사용하여 컬럼을 하나로 묶음으로써 해결할 수 있다.

### 3.1.3 집약, 해시, 정렬
GROUP BY로 집약을 수행할 때에는 해시 알고리즘을 사용한다. 떄에 따라서는 정렬 알고리즘을 사용하기도 한다.

이는 GROUP BY 구에 지정되어 있는 필드를 해시 함수를 사용해 해시키로 변환하고 같은 해시 키를 가진 그룹을 모아 집약하는 방법이다.

정렬과 해시 모두 메모리를 많이 사용하기 떄문에 충분한 워킹 메모리가 확보되지 않으면 스왑이 발생하여 성능 저하가 발생할 수 있다.

## 3.2 합쳐서 하나
특정 레코드로 전체를 커버하지 못하더라도 여러 레코드를 조합해서 커버할 수 있다는 개념이 '합쳐서 하나'이다.

```sql

SELECT product_id
FROM PriceByAge
GROUP BY product_id
HAVING SUM(high_age - low_age + 1) >= 100;
```

위의 코드는 product_id가 같은 레코드들을 하나로 합쳐서 연령 커버값이 100 이상인 것들을 추출한다. 이 과정에서 제품별로 중복연령은 없다고 가정하였다.

같은 방식으로 날짜, 시간에도 적용할 수 있다.

## 3.3 자르기

### 3.3.1 파티션
자르기는 원래 모든 집합인 테이블을 작은 부분 집합들로 분리하는 것이다.

이 관점에서 GROUP BY 구는 자르기와 집약을 동시에 수행하는 연산이다.

자르기는 다음과 같이 수행한다.

```sql
SELECT product_id, COUNT(*) AS cnt
FROM PriceByAge
GROUP BY product_id
HAVING cnt >= 2;
```

이 때, GROUP BY 구와 SELECT 구에 모두 자르기의 기준이 되는 키를 입력하는 것이 포인트이다.

SELECT 구에서 붙인 별칭을 GROUP BY 구에서 사용해서 단순하게 작성할 수도 있지만 표준 방법은 아니다.

### 3.3.1 PARTITION BY 구

PARTITION BY 구는 GROUP BY 구와 비슷하지만 집약을 수행하지 않는다. 그 외에 실질적인 기능에 차이가 없다.

```sql
SELECT product_id, COUNT(*) OVER (PARTITION BY CASE WHEN low_age <= 20 THEN '어린이'
                                                   WHEN low_age <= BETWEEN 20 AND 69 THEN '성인'
                                                   WHEN low_age >= 70 THEN '노인'
                                                   ELSE '기타' END) AS age_group
FROM PriceByAge
WHERE cnt >= 2;
```

PARTITION BY 구는 집약을 수행하지 않기 때문에 테이블의 레코드가 모두 원래 형태로 출력된다.

GROUP BY 구는 입력 집합을 집약하므로 전혀 다른 레벨의 출력으로 변환하지만 PARTITION BY 구는 입력에 정보를 추가할 뿐이다.
