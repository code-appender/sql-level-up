# 4장 : 집약과 자르기

## 12장 집약

sql에는 5개의 집약 함수가 있다.

1. count
2. sum
3. avg
4. max
5. min

### 여러개의 레코드를 한 개의 레코드로 집약

Group BY구로 집약했을때 SELCET 구에 입력할 수 있는것

- 상수
- Group BY 구에서 사용한 집약 키
- 집약 함수

아래는 여러개의 레코드를 한개의 레코드로 집약한다는것을 잘 보여준 예제이다.

```sql
SELECT id,
    MAX(CASE WHEN data_type ='A' THEN data_1 ELSE NULL END) AS data_1,
    MAX(CASE WHEN data_type ='A' THEN data_2 ELSE NULL END) AS data_2,
    MAX(CASE WHEN data_type ='B' THEN data_3 ELSE NULL END) AS data_3,
    MAX(CASE WHEN data_type ='B' THEN data_4 ELSE NULL END) AS data_4,
    MAX(CASE WHEN data_type ='C' THEN data_5 ELSE NULL END) AS data_5,
    MAX(CASE WHEN data_type ='C' THEN data_6 ELSE NULL END) AS data_6
FROM NonAggTbl
GROUP BY id;

```

이런 집약 쿼리의 실행계회은 어떻게 될까?

스캔 → group by 집약 (해시 알고리즘)

group by는 해시 or 정렬이지만 요즘에는 해시알고리즘을 많이 사용한다.

정렬 또는 해시를 위해 PGA라는 메모리 영역을 사용한다. 이때 PGA 크기가 집약 대상 데이터 양에 비해 부족하면 일시 영역(저장소)를 사용해 부족한 만큼 채운다. 이걸 Temp 탈락 이라고 한다. 해당 현상이 발생화면 성능이 극단적으로 떨어진다. 따라서 연산 대상 레코드 수가 많은 GROUP BY구를 사용하는 SQL에서는 충분한 성능검증을 진행해야 한다. 최악의 경우 SQL이 비정상적으로 종료할 수 있다.

### 합쳐서 하나

여러개의 레코드로 한 개의 범위를 커버

`모든 연령 범위의 사람들에게 제공할 수 있는 제품`

```sql
SELECT product_id
FROM priceByAge
GROUP BY product_id
HAVING SUM(high_age - low_age + 1) = 101;
```

`여러 레코드에서 운영된 날을 연산`

```sql
SELECT room_ngr,
        SUM(end_date -start_date) AS working_days
FROM HoterRooms
GROUP BY room_nbr
HAVING SUM(end_date - start_date) >= 10;
```

## 13장 자르기

### 자르기와 파티션

첫 문자 알파벳마다 몇 명의 사람이 존재하는지 계산

```sql
SELECT SUBSTRING(name, 1, 1)AS label,
        COUNT(*)
    FROM Persons
GROUP BY SUBSTRING(name, 1, 1);
```

| label | COUNT(*) |
| --- | --- |
| A | 4 |
| B | 2 |
| J | 3 |
| M | 1 |

이렇게 Group BY 구로 잘라 만든 하나하나의 부분 집합을 수학적으로 ‘파티션’이라고 부릅니다.

자르기의 기준이 되는 키를 `GROUP BY 구와 SELECT 구 모두에 입력하는것이` 포인트이다.

어린이, 성인, 노인 분리 계산

```sql
SELCET CASE WHEN age<20 THEN '어린이'
                    WHEN age BETWEEN 20 AND 69 THEN '성인'
                    WHEN age>= 70 THEN '노인'
                    ELSE NULL END AS age_class
			COUNT(*)
FROM PERSONS 
GROUP BY CASE WHEN age<20 THEN '어린이'
                    WHEN age BETWEEN 20 AND 69 THEN '성인'
                    WHEN age>= 70 THEN '노인'
                    ELSE NULL END AS age_class
```

### Partition by 구를 사용한 자르기

```sql
SELECT name,
        age,
        CASE WHEN age<20 THEN '어린이'
                    WHEN age BETWEEN 20 AND 69 THEN '성인'
                    WHEN age>= 70 THEN '노인'
                    ELSE NULL END AS age_class
        RANk() OVER(PARTITION BY  CASE WHEN age<20 THEN '어린이'
                        WHEN age BETWEEN 20 AND 69 THEN '성인'
                        WHEN age>= 70 THEN '노인'
                        ELSE NULL END
        ORDER BY age) AS age_rank_in_class
FROM persons
ORDER BY age_class, age_rank_in_class 
```

groupby 와 partition by구에는 사실 집약 이라는 기능을 제외하면 차이가 없다.

group by 구는 입력 집합을 집약하므로 전혀 다른 레벨의 출력으로 변환하지만 partition by 구는 입력에 정보를 추가할 뿐이므로 원본 테이블 정보를 완전히 그대로 유지합니다 .

group by 구가 식을 매개변수로 받는 이상 partition by 구 또한 마찬가지라는 것은 논리적으로 아무 문제 없는 결론이다.