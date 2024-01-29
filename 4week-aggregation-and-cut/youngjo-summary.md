# 4장. 집약과 자르기

## 집약

SQL에는 5개의 집약 함수가 있다.

1. COUNT
2. SUM
3. AVG
4. MAX
5. MIN

### 여러 개의 레코드를 한 개의 레코드로 집약

GROUP BY 구로 집약했을때 SELCET 구에 입력할 수 있는 것
- 상수
- Group BY 구에서 사용한 집약 키
- 집약 함수

잘못된 예시 -> 집합을 택해야 하나 요소를 선택하였음.
```sql
SELECT id,
    CASE WHEN data_type ='A' THEN data_1 ELSE NULL END) AS data_1,
    CASE WHEN data_type ='A' THEN data_2 ELSE NULL END) AS data_2,
    CASE WHEN data_type ='B' THEN data_3 ELSE NULL END) AS data_3,
    CASE WHEN data_type ='B' THEN data_4 ELSE NULL END) AS data_4,
    CASE WHEN data_type ='C' THEN data_5 ELSE NULL END) AS data_5,
    CASE WHEN data_type ='C' THEN data_6 ELSE NULL END) AS data_6
FROM NonAggTbl
GROUP BY id;
```

옳은 예시 -> 집약 함수를 통해 집합을 만들어 선택함
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

- 실행 계획

풀 스캔 -> GROUP BY
정렬, 해시 둘 중 하나를 사용 (최근에는 빠른 성능의 해시 알고리즘 사용)
정렬과 해시 모두 메모리를 많이 사용하기 때문에 충분한 워킹 메모리가 확보되지 않으면 스왑 발생
그래서 GROUP BY 구를 사용하는 SQL은 충분한 성능 검증이 필요함.

## 자르기

자르기 : 모집합인 테이블을 작은 부분 집합들로 분리하는 것.

### 자르기와 파티션

첫 문자 알파벳마다 몇 명의 사람이 존재하는지 계산하는 예시
```sql
SELECT SUBSTRING(name, 1, 1)AS label,
        COUNT(*)
    FROM Persons
GROUP BY SUBSTRING(name, 1, 1);
```

| label | COUNT(*) |
|-------|----------|
| A     | 2        |
| B     | 3        |
| C     | 1        |
| D     | 3        |

파티션 : 하나의 모집합을 GROUP BY 구로 잘라 만든 부분 집합 (중복 X)
자르기의 기준이 되는 키를 GROUP BY 구와 SELECT 구 모두에 입력하는 것이 포인트

### PARTITION BY 구를 사용한 자르기

```sql
SELECT name,
        age,
        CASE WHEN age < 20 THEN '어린이'
                    WHEN age BETWEEN 20 AND 69 THEN '성인'
                    WHEN age >= 70 THEN '노인'
                    ELSE NULL END AS age_class
        RANk() OVER(PARTITION BY  CASE WHEN age < 20 THEN '어린이'
                        WHEN age BETWEEN 20 AND 69 THEN '성인'
                        WHEN age >= 70 THEN '노인'
                        ELSE NULL END
        ORDER BY age) AS age_rank_in_class
FROM persons
ORDER BY age_class, age_rank_in_class 
```

| name     | age | age_class | age_rank_in_class |
|----------|-----|---|-------------------|
| Darwin   | 12  | 어린이 | 1                 |
| Adela    | 21  | 성인 | 1                 |
| Dawson   | 25  | 성인 | 2                 |
| Anderson | 30  | 성인 | 3                 |
| Donald   | 30  | 성인 | 3                 |
| Bill     | 39  | 성인 | 5                 |
| Becky    | 54  | 성인 | 6                 |
| Bates    | 87  | 노인 | 1                 |
| Chris    | 90  | 노인 | 2                 |

GROUP BY - 파티션(테이블의 레코드 원형 그대로 나옴) + 집약
PARTITION BY - 파티션(테이블의 레코드 원형 그대로 나옴)