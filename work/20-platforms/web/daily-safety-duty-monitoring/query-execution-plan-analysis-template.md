# Oracle 실행계획 분석 템플릿

쿼리 개선 전후의 실제 실행비용을 같은 조건에서 비교하기 위한 표준 절차다. 운영 환경에서는 권한·부하 정책을 먼저 확인한다.

## 1. 실행 통계 활성화

```sql
Alter Session Set Statistics_Level = All;
```

## 2. 대상 쿼리에 식별 주석 추가

실행할 `Select` 문장의 시작 위치에 날짜와 식별자를 포함한 주석을 넣는다. 실행마다 고유한 식별자를 사용한다.

```sql
/* 20260713test1 */
/*+ Gather_Plan_Statistics */
```

## 3. SQL_ID 확인

```sql
Select
    SQL_ID,  -- SQL 식별자
    CHILD_NUMBER,  -- 하위 커서 번호
    SQL_TEXT  -- 실행 SQL 원문
From
    V$SQL
Where 1 = 1
And SQL_TEXT Like '%20260713test1%'
Order By
    LAST_ACTIVE_TIME Desc;
```

가장 최근 실행의 `SQL_ID`와 필요하면 `CHILD_NUMBER`를 기록한다.

## 4. 실제 실행계획 조회

`SQL_ID_VALUE` 자리에 앞 단계에서 확인한 SQL_ID를 넣는다. CHILD_NUMBER를 특정해야 하면 두 번째 인자에 값을 입력한다.

```sql
Select
    *  -- 실제 실행 통계
From
    Table(
        Dbms_Xplan.Display_Cursor(
            'SQL_ID_VALUE',
            Null,
            'Allstats Last'
        )
    );
```

## 기록 항목

- 동일 파라미터와 데이터 시점 여부
- 반환 행 수
- 총 `Buffers`, `A-Time`
- 주요 테이블의 `Starts`, `A-Rows`, `Buffers`
- 예상 행 수(`E-Rows`)와 실제 행 수(`A-Rows`)의 큰 차이
- 개선 전후 비교와 다음 개선 후보
