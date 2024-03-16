# Renewal and Data Model

## 갱신은 효율적으로

1. NULL 채우기

    | keycol | seq | val |
    | --- | --- | --- |
    | A | 1 | 50 |
    | A | 2 |  |
    | A | 3 |  |
    | A | 4 | 70 |
    | A | 5 |  |
    | A | 6 | 900 |
    | B | 7 | 10 |
    | B | 8 | 20 |
    | B | 9 |  |
    | B | 10 | 3 |
    | B | 11 |  |
    | B | 12 |  |
    - 본래 값은 있지만 이전 레코드와 같은 값을 가지기에 생략한 것
    - 아래 조건을 만족시키는 상관 서브쿼리 이용
        - 같은 keycol 값을 가짐
        - 현재 레코드보다 작은 seq 필드를 가짐
        - val 필드가 null이 아님
    
    ```sql
    UPDATE OmitTbl
    	SET val = (SELECT val
    			FROM OmitTbl OT1
    			WHERE OT1.keycol = OmitTbl1.keycol
    				AND OT1.seq = (SELECT MAX(seq(
    							FROM OmitTbl1 OT2
    							WHERE OT2.keyco; = OmitTbl1.keycol
    							AND OT2.seq < OmitTbl.seq
    							AND OT2.val IS NOT NULL))
    	WHERE val IS NULL;
    ```

2. 반대로 nulldmf wkrtjd
    - 채우기 역연산 시행

    ```sql
    UPDATE OmitTbl
    	SET val = CASE WHEN val 
    			= (SELECT val
    				FROM OmitTbl 01
    				WHERE O1.keycol = OmitTbl.keycol
    				AND O1.seq
    					= (SELECT MAX(seq)
    						FROM OmitTBl O2
    						WHERE O2.keycol = OmitTbl1.keycol
    						AND O2.seq < OmitTbl1.seq))
    	THEN NULL
    	ELSE val END;
    ```


## 레코드에서 필드로의 갱신

| student_id | subject | score |
| --- | --- | --- |
| A001 | 영어 | 100 |
| A001 | 국어 | 58 |
| A001 | 수학 | 90 |
| B002 | 영어 | 77 |
| B002 | 국어 | 60 |
| C001 | 영어 | 52 |
| C003 | 국어 | 49 |
| C003 | 사회 | 100 |

| student_id | score_en | score_nl | score_mt |
| --- | --- | --- | --- |
| A001 |  |  |  |
| B002 |  |  |  |
| C003 |  |  |  |
1. 필드를 하나씩 갱신
    - 명확하고 간단하지만 3개의 상관 서브쿼리를 실행해야 함

    ```sql
    UPDATE ScoreCols
    	SET score_en = (SELECT score
    		FROM ScoreRows SR
    		WHERE SR.student_id = ScoreCols.student_id
    		AND subject = '영어'),
    	SET score_nl = (SELECT score
    		FROM ScoreRows SR
    		WHERE SR.student_id = ScoreCols.student_id
    		AND subject = '국어'),
    	SET score_mt = (SELECT score
    		FROM ScoreRows SR
    		WHERE SR.student_id = ScoreCols.student_id
    		AND subject = '수학');
    ```

2. 다중 필드 할당
    - 여러 개의 필드를 리스트화하고 한 번에 갱신하는 방법

    ```sql
    UPDATE ScoreCols
    	SET (score_en, score_nl, score_mt)
    		= (SELECT MAX(CASE WHEN subject = '영어'
    						THEN score
    						ELSE NULL END) AS score_en,
    						MAX(CASE WHEN subject = '국어'
    						THEN score
    						ELSE NULL END) AS score_nl,
    						MAX(CASE WHEN subject = '수학'
    						THEN score
    						ELSE NULL END) AS score_mt
    			FROM ScoreRows SR
    			WHERE ST.student_id = ScoreCols.student_id);
    ```

    - 상관 서브 쿼리가 하나로 정리된 대신 ScoreRows 테이블에 대한 접근이 유일 검색이 아닌 범위 검색으로 변함
    - MAX 함수의 정렬 연산이 추가됨
    - 다중 필드 할당 사용
    - 스칼라 서브 쿼리: CASE 식으로 점수를 검색할 때 MAX 함수를 적용해서 스칼라값을 받아 옴
3. NULL 제약이 걸려 있는 경우 (모든 필드 NOT NULL)

| student_id | score_en | score_nl | score_mt |
| --- | --- | --- | --- |
| A001 | 0 | 0 | 0 |
| B002 | 0 | 0 | 0 |
| C003 | 0 | 0 | 0 |
| D004 | 0 | 0 | 0 |
- UPDATE 구문 사용

    ```sql
    UPDATE ScoreColsNN
    	SET score_en = COALESCE((SELECT score
    		FROM ScoreRows
    		WHERE srudent_id - ScoreColsNN.student_id
    		AND subject = '영어'), 0),
    	score_nl = COALESCE((SELECT score
    		FROM ScoreRows
    		WHERE srudent_id - ScoreColsNN.student_id
    		AND subject = '국어'), 0),
    	score_mt = COALESCE((SELECT score
    		FROM ScoreRows
    		WHERE srudent_id - ScoreColsNN.student_id
    		AND subject = '수학'), 0)
    WHERE EXISTS (SELECT *
    	FROM ScoreRows
    	WHERE student_id = ScoreColsNN.student_id);
    ```

- MERGE 구문 사용

    ```sql
    MERGE INTO ScoreColsNN
    	USING (SELECT student_id,
    		COALESCE(MAX(CASE WHEN subject = '영어'
    			THEN score
    			ELSE NULL END), 0) AS score_en,
    		COALESCE(MAX(CASE WHEN subject = '국어'
    			THEN score
    			ELSE NULL END), 0) AS score_nl,
    		COALESCE(MAX(CASE WHEN subject = '수학'
    			THEN score
    			ELSE NULL END), 0) AS score_mt,
    	FROM ScoreRows
    	GROUP BY student_id) SR
    	ON (ScoreColsNN.student_id = SR.student_id)
    WHEN MATCHED THEN
    	UPDATE SET ScoreColsNN.score_en = SR.score_en,
    						 ScoreColsNN.score_nl = SR.score_nl,
    						 ScoreColsNN.score_mt = SR.score_mt;
    ```


## 같은 테이블의 다른 레코드로 갱신

| brand | sale_date | price |
| --- | --- | --- |
| A철강 | 2008-07-01 | 1000 |
| A철강 | 2008-07-04 | 1200 |
| A철강 | 2008-08-12 | 800 |
| B상사 | 2008-06-04 | 3000 |
| B상사 | 2008-09-11 | 3000 |
| C전기 | 2008-07-01 | 9000 |
| D산업 | 2008-06-04 | 5000 |
| D산업 | 2008-06-05 | 5000 |
| D산업 | 2008-06-06 | 2800 |
| D산업 | 2008-12-01 | 5100 |

| brand | sale_date | price | trend |
| --- | --- | --- | --- |
- trend는 올랐다면 ↑, 내렸다면 ↓, 일정하다면 → 지정
1. 상관 서브쿼리 사용

    ```sql
    INSERT INTO Stocks2
    SELECT brand, sale_date, price,
    	CASE SIGN (price -
    		(SELECT price
    		FROM Stocks S1
    		WHERE brand = Stocks.brand
    			AND sale_date = 
    				(SELECT NAX(sale_date)
    				FROM Stocks S2
    				WHERE brand = Stocks.brand
    				AND sale_date < Stocks.sale_date)))
    	WHEN - 1 THEN '↓'
    	WHEN 0 THEN '->'
    	WHEN 1 THEN '↑'
    	ELSE NULL
    END
    FROM Stocks;
    ```

2. 윈도우 함수 사용

    ```sql
    INSERT INTO Stocks2
    SELECT brand, sale_date, price,
    	CASE SIGN (price -
    		(SELECT price
    		FROM Stocks S1
    		WHERE brand = Stocks.brand
    			AND sale_date = 
    				(SELECT NAX(sale_date)
    				FROM Stocks S2
    				WHERE brand = Stocks.brand
    				AND sale_date < Stocks.sale_date)))
    	WHEN - 1 THEN '↓'
    	WHEN 0 THEN '->'
    	WHEN 1 THEN '↑'
    	ELSE NULL
    END
    FROM Stocks;
    ```

3. INSERT와 UPDATE 어떤 것이 좋을까?
    - INSERT 장점
        - 일반적으로 고속 처리 기대 가능
        - MySQL처럼 갱신 SQL에서의 자기 참조를 허락하지 않는 데이터베이스에서 사용 가능
    - INSERT 단점
        - 같은 크기와 구조를 가진 데이터를 두 개만 만들어야 해서 저장소 과부하

## 갱신이 초래하는 트레이드오프

| order_id | order_shop | order_name | order_date |
| --- | --- | --- | --- |
| 10000 | 서울 | 윤인성 | 2011/08/22 |
| 10001 | 인천 | 연하진 | 2011/09/01 |
| 10002 | 인천 | 패밀리마트 | 2011/09/20 |
| 10003 | 부천 | 한빛미디어 | 2011/08/05 |
| 10004 | 수원 | 동네슈퍼 | 2011/08/22 |
| 10005 | 성남 | 야근카페 | 2011/08/29 |

| order_id | order_receipt_id | item_group | delivery_date |
| --- | --- | --- | --- |
| 10000 | 1 | 식기 | 2011/08/24 |
| 10000 | 2 | 과자 | 2011/08/25 |
| 10000 | 3 | 소고기 | 2011/08/26 |
| 10001 | 1 | 어패류 | 2011/09/04 |
| 10002 | 1 | 과자 | 2011/09/22 |
| 10002 | 2 | 조미료세트 | 2011/09/22 |
| 10003 | 1 | 쌀 | 2011/08/06 |
| 10003 | 2 | 소고기 | 2011/08/10 |
| 10003 | 3 | 식기 | 2011/08/10 |
| 10004 | 1 | 야채 | 2011/08/23 |
| 10005 | 1 | 음료수 | 2011/08/30 |
| 10005 | 2 | 과자 | 2011/08/30 |
1. 주문일과 배송 예정일의 차이 구하기

    ```sql
    SELECT O.order_id,
    	O.order_name,
    	ORC.delivery_date - O.order_date AS diff_days
    FROM orders 0
    	INNER JOIN OrderReceipts ORC
    	ON O.order_id = ORC.order_id
    WHERE ORC.delivery_date - O.order_date >= 3;
    ```


2. 주문 단위로 집약

```sql
SELECT O.order_id,
	MAX(O.order_name),
	MAX(ORC.delivery_date - O.order_date) AS diff_days
FROM orders 0
	INNER JOIN OrderReceipts ORC
	ON O.order_id = ORC.order_id
WHERE ORC.delivery_date - O.order_date >= 3
GROUP BY O.order_id;
```

## 모델 갱신의 주의점

1. 높아지는 갱신 비용
    - 배송 지연 플래그에 필드 값는 처리가 필요
    - 따라서 검색 부하를 갱신 부하로 미루는 모습
2. 갱신까지의 시간 랙 (Time Rag) 발생
    - 데이터의 실시간성이라나ㅡㄴ 문제 발생
3. 모델 갱신 비용 발생
    - RDB 데이터 모델 갱신은 코드 기반의 수정에 비해 대대적인 수정이 요구됨