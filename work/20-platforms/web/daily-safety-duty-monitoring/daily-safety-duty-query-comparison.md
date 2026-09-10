# 일일안전당직자 모니터링 쿼리 비교 및 운영 조회

## 목적

`selectIndivActRateList`의 세 가지 쿼리안을 비교하고, 2026년 운영 데이터에서 **중복점검 미사용이며 일일안전당직자는 지정됐지만 항목별 안전당직자는 지정되지 않은 현장**을 조회하는 SQL을 보관한다.

## 원본 쿼리 보존 기준

| 구분 | 명칭 | 원본 확인 위치 | 상태 |
| --- | --- | --- | --- |
| 1 | 수정 전 쿼리 | Git `2dae80a16`의 `MontInqCompanyPersonInChargeMonitoringSql.xml` / `selectIndivActRateList` | 원본 확인 |
| 2 | 수정 후 쿼리_이유진 | 작업 소스 `MontInqCompanyPersonInChargeMonitoringSql.xml` / `selectIndivActRateList` | 원본 확인 |
| 3 | 수정 후_정현석 리더님 | 전달받은 SQL 원문 (`TR_DUTY`, `And Rownum >= 1` 포함) | 별도 비교안 |

> 원문 식별 기준: 1번은 `TR_CHK_LIST` 안에서 `TSF_SCRTCHK_DTLI`에 대해 `Not Exists`와 `Exists`를 각각 수행한다. 2번은 `TR_SCRTCHK_DTLI`, `TR_SCRTCHK_DTLI_CNT`, `TR_SCRTCHK_DTLI_USER`, `TR_SERCURITYCHECK_DETAIL` CTE를 추가한다. 3번은 `TR_DUTY` CTE와 의도적인 `And Rownum >= 1` no-merge 장치를 사용한다.

### 1번: 수정 전 쿼리

원본 핵심 조건은 아래와 같다. 항목별 지정 당직자 존재 여부와 현재 당직자의 지정 여부를 상관 서브쿼리 두 개로 판별한다.

```sql
And (
    Not Exists (
        Select 1
        From
            TSF_SCRTCHK_DTLI x
        Where 1 = 1
        And x.CHECK_TF = 'T'
        And x.COMPANY_ID = b.COMPANY_ID
        And x.PROJ_CODE = b.PROJ_CODE
        And x.ASSMNT_SEQ = b.ASSMNT_SEQ
        And x.ASSMNT_LIST_SEQ = b.ASSMNT_LIST_SEQ
        And x.RISK_FCT_SEQ = b.RISK_FCT_SEQ
        And x.IMPRV_MTHD_SEQ = b.IMPRV_MTHD_SEQ
        And x.CHECK_DATE = b.CHK_DATE
    )
    Or Exists (
        Select 1
        From
            TSF_SCRTCHK_DTLI x
        Where 1 = 1
        And x.CHECK_TF = 'T'
        And x.COMPANY_ID = b.COMPANY_ID
        And x.PROJ_CODE = b.PROJ_CODE
        And x.ASSMNT_SEQ = b.ASSMNT_SEQ
        And x.ASSMNT_LIST_SEQ = b.ASSMNT_LIST_SEQ
        And x.RISK_FCT_SEQ = b.RISK_FCT_SEQ
        And x.IMPRV_MTHD_SEQ = b.IMPRV_MTHD_SEQ
        And x.CHECK_DATE = b.CHK_DATE
        And x.USER_NO = c.USER_NO
    )
)
```

전체 원문은 Git의 `2dae80a16:src/main/resources/sqlmap/mappers/mont/MontInqCompanyPersonInChargeMonitoringSql.xml`에서 `selectIndivActRateList` 구간을 기준으로 보관한다.

### 2번: 수정 후 쿼리_이유진

1번의 반복 상관 서브쿼리를 사전 범위 제한 및 집계 CTE로 치환한 원본이다.

```sql
TR_SCRTCHK_DTLI_CNT As (
    Select
        a.COMPANY_ID,
        a.PROJ_CODE,
        a.ASSMNT_SEQ,
        a.ASSMNT_LIST_SEQ,
        a.RISK_FCT_SEQ,
        a.IMPRV_MTHD_SEQ,
        a.CHECK_DATE,
        Count(*) USER_CNT
    From
        TR_SCRTCHK_DTLI a
    Where 1 = 1
    Group By
        a.COMPANY_ID,
        a.PROJ_CODE,
        a.ASSMNT_SEQ,
        a.ASSMNT_LIST_SEQ,
        a.RISK_FCT_SEQ,
        a.IMPRV_MTHD_SEQ,
        a.CHECK_DATE
),
TR_SCRTCHK_DTLI_USER As (
    Select
        a.COMPANY_ID,
        a.PROJ_CODE,
        a.ASSMNT_SEQ,
        a.ASSMNT_LIST_SEQ,
        a.RISK_FCT_SEQ,
        a.IMPRV_MTHD_SEQ,
        a.CHECK_DATE,
        a.USER_NO
    From
        TR_SCRTCHK_DTLI a
    Where 1 = 1
    Group By
        a.COMPANY_ID,
        a.PROJ_CODE,
        a.ASSMNT_SEQ,
        a.ASSMNT_LIST_SEQ,
        a.RISK_FCT_SEQ,
        a.IMPRV_MTHD_SEQ,
        a.CHECK_DATE,
        a.USER_NO
)
```

핵심 판정은 `And (e.USER_CNT Is Null Or f.USER_NO Is Not Null)`이며, 현재 작업 소스의 `selectIndivActRateList` 전문이 2번 원문이다.

### 3번: 수정 후_정현석 리더님

`TR_DUTY`에서 회차 범위의 당직 데이터를 먼저 구성하고, `TR_PIC` 병합을 방지하기 위한 의도적 조건을 둔 원본이다.

```sql
TR_DUTY As (
    Select
        e.COMPANY_ID,
        e.PROJ_CODE,
        e.ASSMNT_SEQ,
        e.CHECK_DATE,
        b.COMPANY_ID EMP_COMPANY_ID,
        b.USER_NO,
        b.USER_NM,
        b.POSITION_CD
    From
        TSF_SERCURITYCHECK_DETAIL e,
        TCC_EMPLOYE b
    Where 1 = 1
    And e.CHECK_TF = 'T'
    And b.USER_NO = e.USER_NO
    And e.COMPANY_ID = :COMPANY_ID
    And Exists (
        Select 1
        From
            TR_TIME x
        Where 1 = 1
        And x.COMPANY_ID = e.COMPANY_ID
        And x.PROJ_CODE = e.PROJ_CODE
        And x.ITEM_SN = e.ASSMNT_SEQ
    )
    -- 삭제 금지: 항상 참인 ROWNUM 조건으로 Oracle의 TR_PIC 병합(merge)을 막는 의도적인 no-merge 장치.
    -- TR_DUTY를 한 번만 계산하도록 유지하며, 제거 시 병합으로 반복 계산될 수 있음.
    And Rownum >= 1
)
```

`Rownum >= 1`은 일반적인 최적화 규칙으로 유지하면 안 된다. 아래의 검증 항목(결과 건수, Starts, Buffers, 경과 시간)을 만족할 때만 유지한다.

## 평가 결과

> 아래 점수는 현재 운영의 유지보수성, 팀의 기존 `Exists` / `Not Exists` 사용 방식, 확인된 성능 측정값을 함께 반영한 평가다. 단순 Buffers 절감만으로 유지보수성이 낮은 구조를 기본안으로 정하지 않는다.

| 순위 | 쿼리 | 점수 | 판단 |
| ---: | --- | ---: | --- |
| 1 | 3번 — 수정 후_정현석 리더님 | 90/100 (검증 조건부) | 1번의 업무 규칙을 보존하면서 당직자 원천 범위를 먼저 축소하는 운영 우선 검토안 |
| 2 | 1번 — 수정 전 | 88/100 | 업무 규칙을 가장 직접적으로 표현하므로 안정적인 기준안 |
| 3 | 2번 — 수정 후_이유진 | 82/100 | 반복 접근 제거에는 유리하지만 CTE·외부 조인 조건이 업무 규칙을 간접적으로 표현하므로 성능 대안으로 관리 |

### 유지보수성 관점: 1번을 기본안으로 권고하는 이유

핵심 업무 규칙은 다음 두 경우다.

- 항목별 지정 당직자가 없으면 해당 점검일자의 당직자 전체에게 항목을 배분한다.
- 지정 당직자가 있으면 현재 당직자가 지정 대상일 때만 항목을 배분한다.

1번과 3번의 조건은 이 규칙을 SQL에서 그대로 읽을 수 있다.

```sql
And (
    Not Exists (...)  -- 항목별 지정 당직자가 없으면
    Or Exists (...)   -- 현재 당직자가 지정 당직자이면
)
```

반면 2번의 `e.USER_CNT Is Null Or f.USER_NO Is Not Null`은 같은 업무 규칙을 구현하지만, `TR_SCRTCHK_DTLI_CNT`와 `TR_SCRTCHK_DTLI_USER`의 역할 및 외부 조인 결과를 함께 해석해야 한다. 기존 팀의 `Exists` / `Not Exists` 중심 방식과 일관된 1번·3번이 신규 유지보수자에게 이해하기 쉽다.

### 수정 전과 현석 리더님 안의 비교

중복점검 미사용(`DUP_CHK_PSBL_TF = 'F'`) 대상에서는 3번이 1번의 업무 규칙을 바꾸지 않고 당직자 원천 데이터를 먼저 제한한다.

| 항목 | 1번 — 수정 전 | 3번 — 수정 후_정현석 리더님 |
| --- | --- | --- |
| 항목 배분 규칙 | `Not Exists (...) Or Exists (...)` | 동일하게 유지 |
| 당직자 데이터 접근 | `TR_PIC`에서 `TSF_SERCURITYCHECK_DETAIL`, `TCC_EMPLOYE`를 직접 조인 | `TR_DUTY`에서 `CHECK_TF`, 회사, `TR_TIME` 회차 범위를 먼저 적용 |
| 이후 조인 확장 전 입력 집합 | 옵티마이저 조인 순서에 의존 | 당직자 집합을 별도 CTE로 구성하여 선제한 의도를 명시 |
| CTE 병합 제어 | 없음 | `TR_DUTY`의 `And Rownum >= 1`로 병합에 따른 재계산을 방지하려는 의도 |
| 운영 권고 | 결과 비교 기준안 | 결과·계획 검증 후 우선 적용안 |

3번의 `TR_DUTY`는 아래 순서로 당직자 데이터를 구성한다.

1. `TSF_SERCURITYCHECK_DETAIL`에서 `CHECK_TF = 'T'` 및 회사 조건을 적용한다.
2. `TR_TIME`에 존재하는 회사·현장·위험성평가회차만 `Exists`로 남긴다.
3. 남은 당직자에 대해서만 `TCC_EMPLOYE` 정보를 결합한다.
4. 이후 `TR_PIC`에서 원청/협력사, 직책, 직위, 사용자명 조건을 적용한다.

따라서 조인 확장 전에 원천 당직자 범위를 줄일 수 있으며, 업무 규칙도 기존과 같은 `Exists` / `Not Exists`로 남는다. 이 점에서 3번은 유지보수성과 성능 개선 가능성을 함께 갖춘 안이다.

### 2번을 성능 대안으로 보관하는 이유

- 1번의 `Not Exists` / `Exists` 반복 접근을 `TR_SCRTCHK_DTLI_CNT`와 `TR_SCRTCHK_DTLI_USER`의 집계 결과 조인으로 전환했다.
- `TR_TIME`으로 `TSF_SCRTCHK_DTLI`와 `TSF_SERCURITYCHECK_DETAIL` 범위를 먼저 제한한다.
- 기존 측정에서 반환 건수는 26건으로 동일했고, Buffers는 `12,094 → 10,685`로 1,409(약 11.7%) 감소했으며 경과 시간은 `0.06초 → 0.05초`였다.
- 항목 지정자가 없으면 모든 당직자에게 배분하고, 지정자가 있으면 해당 사용자에게만 배분하는 기존 규칙을 `e.USER_CNT Is Null Or f.USER_NO Is Not Null`로 보존한다.
- 다만 현재 측정의 `0.01초` 차이만으로 더 복잡한 구조를 기본안으로 채택하지 않는다. 데이터량 증가 또는 반복 상관 접근이 실제 병목으로 확인될 때 적용할 성능 대안으로 관리한다.

### 3번의 장점과 검증 조건

- 장점: `TR_DUTY`에서 당직자·사원 정보를 미리 결합하므로 `TR_PIC`의 입력 범위가 줄 수 있다. 또한 `TR_PIC`의 반복 소비로 `TR_DUTY`가 병합되어 재평가되는 실행계획이라면 no-merge 장치가 유효할 수 있다.
- 검증 조건: 3번은 1번과 같이 `TSF_SCRTCHK_DTLI` 상관 서브쿼리 2개를 유지한다. 또한 `Rownum >= 1`의 효과는 Oracle 버전, 통계, 인덱스, 바인드 값에 의존한다.
- 결론: 결과 건수가 1번과 동일하고, 동일 바인드·동일 시점 데이터에서 전체 Buffers 또는 경과 시간, `TR_DUTY`/`TR_PIC` Starts가 개선되면 3번을 기본 운영안으로 채택한다. 개선 효과가 재현되지 않으면 `Rownum >= 1`을 유지하지 않고 1번을 기준안으로 둔다.

## 운영 데이터 속도 비교용 쿼리

조건: 2026년, 중복점검 미사용, 일일안전당직자는 지정되어 있으나 항목별 안전당직자가 지정되지 않은 현장.

```sql
With
    /* 일일안전당직자 현황
    - 일자/현장/위험성평가회차/조직별 일일안전당직자 수 집계 */
    DAILY_DUTY As (
        Select
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.DEPT_NO,
            a.CHECK_DATE,
            Count(Distinct a.USER_NO) DLY_DUTY_USER_CNT
        From
            TSF_SERCURITYCHECK_DETAIL a
        Where 1 = 1
        And a.CHECK_TF = 'T'
        And a.CHECK_DATE >= To_Date('20260101', 'yyyyMMdd')
        And a.CHECK_DATE < To_Date('20270101', 'yyyyMMdd')
        Group By
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.DEPT_NO,
            a.CHECK_DATE
    ),
    /* 정기점검 대상 일자
    - 정기점검으로 등록된 위험성평가 점검일자 조회 */
    CHECK_TARGET As (
        Select
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.CHK_DATE
        From
            TSF_RISK_ASSMNT_CHK_LST a
        Where 1 = 1
        And a.REG_CHECK_TF = 'T'
        And a.CHK_DATE >= To_Date('20260101', 'yyyyMMdd')
        And a.CHK_DATE < To_Date('20270101', 'yyyyMMdd')
        Group By
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.CHK_DATE
    ),
    /* 항목별 안전당직자 지정 현황
    - 점검결과 항목에 점검관리자(안전당직자)가 지정된 조직/일자 목록 */
    ITEM_DUTY As (
        Select
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.DEPT_NO,
            a.CHK_DATE
        From
            TSF_RISK_ASSMNT_CHK_RSLT a
        Where 1 = 1
        And a.CHK_MNGR_USER_NO Is Not Null
        And a.CHK_DATE >= To_Date('20260101', 'yyyyMMdd')
        And a.CHK_DATE < To_Date('20270101', 'yyyyMMdd')
        Group By
            a.COMPANY_ID,
            a.PROJ_CODE,
            a.ASSMNT_SEQ,
            a.DEPT_NO,
            a.CHK_DATE
    )
/* 정기점검 대상 중 일일안전당직자는 존재하지만
 항목별 안전당직자가 아직 지정되지 않은 대상 조회 */
Select
    a.COMPANY_ID,  -- 회사ID
    SFCC_GET_COMPANY_NAME(a.COMPANY_ID) COMPANY_NAME,  -- 회사명
    a.PROJ_CODE,  -- 현장코드
    c.PROJ_NAME,  -- 현장명
    a.ASSMNT_SEQ,  -- 위험성평가회차
    a.DEPT_NO,  -- 현장귀속조직번호
    To_Char(a.CHECK_DATE, 'yyyy-MM-dd') CHECK_DATE,  -- 점검일자
    a.DLY_DUTY_USER_CNT  -- 일일안전당직자수
From
    DAILY_DUTY a,
    CHECK_TARGET b,
    TCC_PROJ_CODE c,
    TSF_RISK_ASSMNT_MASTR d
Where 1 = 1
And Nvl(d.DUP_CHK_PSBL_TF, 'F') = 'F'
And a.COMPANY_ID = b.COMPANY_ID
And a.PROJ_CODE = b.PROJ_CODE
And a.ASSMNT_SEQ = b.ASSMNT_SEQ
And a.CHECK_DATE = b.CHK_DATE
And a.COMPANY_ID = c.COMPANY_ID
And a.PROJ_CODE = c.PROJ_CODE
And a.COMPANY_ID = d.COMPANY_ID
And a.PROJ_CODE = d.PROJ_CODE
And a.ASSMNT_SEQ = d.ASSMNT_SEQ
And Not Exists (
    Select 1  -- 항목별안전당직자존재여부
    From
        ITEM_DUTY x
    Where 1 = 1
    And x.COMPANY_ID = a.COMPANY_ID
    And x.PROJ_CODE = a.PROJ_CODE
    And x.ASSMNT_SEQ = a.ASSMNT_SEQ
    And x.DEPT_NO = a.DEPT_NO
    And x.CHK_DATE = a.CHECK_DATE
)
Order By
    c.PROJ_NAME,
    a.ASSMNT_SEQ,
    a.DEPT_NO,
    a.CHECK_DATE
```

## 3번 검증 기준

| 항목 | 2번 | 3번 | 판정 |
| --- | ---: | ---: | --- |
| 반환 건수 | 기록 | 기록 | 반드시 동일 |
| 경과 시간 | 기록 | 기록 | 낮은 값 우선 |
| 전체 Buffers | 기록 | 기록 | 낮은 값 우선 |
| `TR_DUTY` / `TR_PIC` Starts | 해당 없음 | 기록 | no-merge 재계산 방지 확인 |
| `TSF_SCRTCHK_DTLI` Buffers·Starts | 기록 | 기록 | 반복 상관 접근 영향 확인 |
