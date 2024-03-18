# 9. 갱신과 데이터 모델

# 9.1 갱신을 효율적으로

## 9.1.1 NULL 채우기

- val 컬럼에 NULL인 부분은 이전 레코드와 같은 값임을 의미한다.
- 즉, NULL 부분에 이전 레코드와 같은 값을 채워넣어야 한다.

- 같은 keycol 필드를 가진다.
- 현재 레코드보다 작은 seq 필드를 가진다.
- val필드가 NULL이 아니다.

```sql
UPDATE OmitTbl
SET val = (SELECT val 
						FROM OmitTbl OT1
	          WHERE OT1.keycol = OmitTbl.keycol
            AND OT1.seq = (SELECT MAX(seq) 
								            FROM OmitTbl OT2
                            WHERE OT2.keycol = OmitTbl.keycol
                            AND OT2.seq < OmitTbl.seq
                            AND OT2.val IS NOT NULL))
WHERE val IS NULL;
```

- OT2에 대해 테이블 스캔의 결과를 MAX함수로 집약하고 OT1 테이블의 레코드를 특정한다.
- 데이터 양이 늘어나면 기본 키 인덱스를 사용해 풀스캔보다 효율적으로 접근 가능하다.

## 9.1.2 반대로 NULL 작성

- 이전 레코드와 같은 값이면 NULL값으로 채우기

```sql
UPDATE OmitTbl
SET val = CASE WHEN val 
            = (SELECT val 
		            FROM OmitTbl OT1
                WHERE OT1.keycol = OmitTbl.keycol
                AND OT1.seq = (SELECT MAX(seq) 
								                FROM OmitTbl OT2
                                WHERE OT2.keycol = OmitTbl.keycol
                                AND OT2.seq < OmitTbl.seq))
          THEN NULL
          ELSE val END;
```

- 조건 1~3에 해당하면 NULL 입력, 해당하지 않으면 val 값 입력
- 스칼라 서브쿼리이므로 CASE문의 매개 변수로 사용 가능하다.
    - 스칼라 서브쿼리: 서브쿼리가 하나의 값을 반환함

# 9.2 레코드에서 필드로의 갱신

## 9.2.1 필드를 하나씩 갱신

- 레코드 기반 테이블에서 필드 기반 테이블로 과목별 점수 이동

```sql
UPDATE ScoreCols
SET score_en=(SELECT score 
							FROM ScoreRows SR
              WHERE SR.student_id = ScoreCols.student_id
              AND subject='영어'),
    score_nl=(SELECT score 
					    FROM ScoreRows SR
              WHERE SR.student_id = ScoreCols.student_id
              AND subject='국어'),
    score_mt=(SELECT score 
					    FROM ScoreRows SR
              WHERE SR.student_id = ScoreCols.student_id
              AND subject='수학');
```

- 한 과목씩 갱신하는 서브쿼리는 간단하고 명확하지만 3개의 서브쿼리를 실행해야 하므로 성능 저하

## 9.2.2 다중 필드 할당

```sql
UPDATE ScoreCols
SET (score_en, score_nl, score_mt)
    = (SELECT MAX(CASE WHEN subject='영어'
                        THEN score
                        ELSE NULL END) AS score_en,
              MAX(CASE WHEN subject='국어'
                        THEN score
                        ELSE NULL END) AS score_nl,
              MAX(CASE WHEN subject='수학'
                        THEN score
                        ELSE NULL END) AS score_mt
        FROM ScoreCols SR
        WHERE SR.student_id=ScoreCols.student_id);
```

- 여러 개의 필드를 리스트화한 후 한 번에 갱싱하는 방법
- 서브쿼리 수를 한꺼번에 처리하는 효과 → 테이블 접근 횟수 감소로 인한 성능 향상, 코드 간결화
- CASE문에 MAX 처리를 하는 이유는?
    - 다른 과목에서 반환된 NULL값과 해당 과목의 점수들이 반환되므로 그 중 실제 사용할 해당 과목의 점수 값을 가져오기 위해서다.

## 9.2.3 NOT NULL 제약이 걸려있는 경우

### UPDATE 구문 사용

```sql
UPDATE ScoreCols
SET (score_en, score_nl, score_mt)
    = (SELECT COALESCE(MAX(CASE WHEN subject='영어'
                                THEN score
                                ELSE NULL END),0) AS score_en,
              COALESCE(MAX(CASE WHEN subject='국어'
                                THEN score
                                ELSE NULL END),0) AS score_nl,
              COALESCE(MAX(CASE WHEN subject='수학'
                                THEN score
                                ELSE NULL END),0) AS score_mt
        FROM ScoreCols SR
        WHERE SR.student_id=ScoreCols.student_id)
WHERE EXISTS(SELECT *
            FROM ScoreRows
            WHERE student_id = ScoreCols.student_id);
```

- EXISTS를 사용해 2개 테이블 사이에 학생 ID가 일치하는 레코드만 갱신하도록 한다.
- 학생은 존재하지만 특정 과목이 존재하지 않을 경우 COALESCE 함수로 NULL을 1로 변경하여 대응한다.

### MERGE 구문 사용

```sql
MERGE INTO ScoreCols
USING (SELECT student_id,
              COALESCE(MAX(CASE WHEN subject='영어'
                                THEN score
                                ELSE NULL END),0) AS score_en,
              COALESCE(MAX(CASE WHEN subject='국어'
                                THEN score
                                ELSE NULL END),0) AS score_nl,
              COALESCE(MAX(CASE WHEN subject='수학'
                                THEN score
                                ELSE NULL END),0) AS score_mt
        FROM ScoreRows
        GROUP BY student_id) SR
        ON (ScoreCols.student_id=SR.student_id)
WHEN MATCHED THEN
    UPDATE SET ScoreCols.score_en=SR.score_en,
                ScoreCols.score_nl=SR.score_nl,
                ScoreCols.score_mt=SR.score_mt;
```

- ON구로 결합 조건을 한번에 정의하여 코드 간결화
- MERGE 구문은 원래 INSERT와 UPDATE를 한번에 실행하기 위해 만들어진 구문이다.

# 9.3 필드에서 레코드로 변경

```sql
UPDATE ScoreRows
SET score=(SELECT CASE ScoreRows.subject
                        WHEN '영어' THEN score_en
                        WHEN '국어' THEN score_nl
                        WHEN '수학' THEN score_mt
                        ELSE NULL
                    END
            FROM ScoreCols
            WHERE student_id=ScoreRows.student_id);
```

- CASE식을 사용해 3개의 필드 내용을 레코드에 하나씩 넣는 쿼리
- 기본 키 인덱스 사용되고 테이블에 한 번 접근하여 성능이 좋다.

# 9.4 같은 테이블의 다른 레코드로 갱신

- 이전 종가와 현재 종가를 비교해 올랐다면 ↑, 내렸다면 ↓, 그대로라면 → 값 입력
- 해당 종목 처음 거래 시 NULL 입력

## 9.4.1 상관 서브쿼리 사용

```sql
INSERT INTO Stocks2
SELECT brad, sale_date, price,
        CASE SIGN(price-
                    (SELECT price
                    FROM Stocks S1
                    WHERE brand = Stocks.brand
                    AND sale_date = (SELECT MAX(sale_Date)
                                     FROM Stocks S2
                                     WHERE brand=Stocks.brand
                                     AND sale_date < Stocks.sale_date)))
        WHEN -1 THEN '↓'
        WHEN 0 THEN '→'
        WHEN 1 THEN '↑'
        ELSE NULL
        END
FROM Stocks;
```

- SIGN함수를 통해 양수면 1, 음수면 -1, 0이면 0을 반환
- (현재 종가) - (이전 종가) 값을 서브쿼리로 구한다.

## 9.4.2 윈도우 함수 사용

```sql
INSERT INTO Stocks2
SELECT brad, sale_date, price,
        CASE SIGN(price-
                    MAX(sale_Date) OVER (PARTITION BY brand
                                         ORDER BY sale_date
                                         ROWS BETWEEN 1 PRECEDING
                                                     AND 1PRECEDING))
        WHEN -1 THEN '↓'
        WHEN 0 THEN '→'
        WHEN 1 THEN '↑'
        ELSE NULL
        END
FROM Stocks S2;
```

- 테이블에 대한 접근이 1회 일어나 서브쿼리보다 간단하다.

## 9.4.3 INSERT와 UPDATE 어떤 것이 좋을까?

- INSERT 장점
    - UPDATE에 비해 성능적으로 나으므로 고속 처리를 기대할 수 있다.
    - MySQL처럼 갱신 SQL에서 자기 참조 허용하지 않는 데이터베이스에서도 사용 가능하다.
- INSERT 단점
    - 같은 크기와 구조를 가진 데이터를 두개씩 만들어 저장소 용량 2배 이상 소비한다.

# 9.5 갱신이 초래하는 트레이드오프

- Orders: 주문 하나에 대응
- OrderReceipts: 주문된 제품 하나에 대응
- 한 번에 여러 제품 주문 가능하므로 일대다 관계다.

주문일과 배송 예정일의 차이가 3일 이상인 주문 번호를 구해보자.

## 9.5.1 SQL을 사용하는 방법

- order_id로 내부 결합하여 주문일과 배송 예정일의 차가 3일 이상인 레코드들 검색

```sql
SELECT O.order_id, O.order_name,
        ORC.delivery_date - O.order_date AS diff_days
FROM Orders O
    INNER JOIN OrderReceipts ORC
        ON O.order_id=ORC.order_id
WHERE ORC.delivery_date-O.order_date>=3;
```

- group by구와 MAX함수를 이용해 집약하면 주문 번호 별 최대 배송 지연일을 구할 수 있다.

```sql
SELECT O.order_id, 
       MAX(O.order_name),
       MAX(ORC.delivery_date - O.order_date) AS max_diff_days
FROM Orders O
    INNER JOIN OrderReceipts ORC
        ON O.order_id=ORC.order_id
WHERE ORC.delivery_date-O.order_date>=3
GROUP BY O.order_id;
```

## 9.5.2 모델 갱신을 사용하는 방법

- 결합 또는 집약을 포함한 SQL 구문을 사용하지 않는 방법
- Orders 테이블에 배송 지연 플래그를 저장하는 del_late_flg 필드를 추가
- 지연되었다면 1, 아니라면 0을 저장

# 9.6 모델 갱신의 주의점

## 9.6.1 높아지는 갱신 비용

- SQL 구문을 사용해 검색하는 부하를 갱신 부하로 변경하는 것
- 상황에 따라 플래그 필드를 UPDATE해야 할 경우 갱신 비용 증가한다.

## 9.6.2 갱신까지의 타임 랙(time rag)

- 배송 예정일이 주문 등록 후 갱신된다면 Orders.del_late_flg와 OrderReceipts.delivery_date가 실시간 동기화가 되지 않아 차이가 발생할 수 있다.
- 완전한 실시간을 요구하는 경우 3, 4번 과정을 동일 트랜잭션으로 처리해야 한다.
- 성능과 실시간성 사이에 트레이드오프 발생한다.

## 9.6.3 모델 갱신비용 발생

- 갱신 대상 테이블을 사용하는 다른 처리에 문제 발생하거나 시스템 품질, 개발 일정에 차질 발생
- 즉, 개발 막바지 단계나 실제 운용중일 때 모델 변경 불가
- 사전에 모든 요인을 생각해두며 모델링을 해야 한다.

# 9.7 시야 협착: 관련 문제

- 주문 번호마다 몇 개의 상품이 주문되었는지 검색한다.

## 9.7.1 다시 SQL을 사용한다면

- 집약 함수를 사용

```sql
SELECT O.order_id, 
       MAX(O.order_name) AS order_name,
       MAX(O.order_date) AS order_date,
       COUNT(*) AS item_count
FROM Orders O
    INNER JOIN OrderReceipts ORC
        ON O.order_id=ORC.order_id
GROUP BY O.order_id;
```

- 윈도우 함수를 사용

```sql
SELECT O.order_id, 
       O.order_name,
       O.order_date,
       COUNT(*) OVER (PARTITION BY O.order_id) AS item_count
FROM Orders O
    INNER JOIN OrderReceipts ORC
        ON O.order_id=ORC.order_id;
```

## 9.7.2 다시 모델 갱신을 사용한다면


- 상품 수 필드를 만들어 Orders 테이블에 레코드가 등록될 때 함께 입력되도록 한다.
- 한 번 등록한 주문을 변경하면 상품 수가 변경되므로 수정 필요하다.

## 9.7.3 초보자보다 중급자가 경계해야

- 어려운 문제를 어렵고 복잡한 방법으로 풀려고 하기보다 넓은 시야를 가지고 해결 방법을 찾아야 한다.

# 9.8 데이터 모델의 중요성

- 데이터 모델 설계를 잘하는 것이 코드를 잘 짜는 것보다 중요하다.
- 처음부터 잘 설계된 데이터 구조를 가지는 것이 필요하다.