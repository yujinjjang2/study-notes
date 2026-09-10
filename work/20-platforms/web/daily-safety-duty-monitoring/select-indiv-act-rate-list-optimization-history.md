# 개인조치율 쿼리 개선 이력

## 대상과 범위

- 화면: 일일안전당직자모니터링
- Query ID: `selectIndivActRateList`
- Mapper: `MontInqCompanyPersonInChargeMonitoringSql.xml`
- 원본 위치: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas\src\main\resources\sqlmap\mappers\mont\MontInqCompanyPersonInChargeMonitoringSql.xml`

이 문서는 실행한 쿼리의 버전별 핵심 구조와 실행계획을 기록한다. 전체 쿼리는 위 Mapper의 `selectIndivActRateList`를 기준으로 확인한다.

## 0. 원본 쿼리

### 핵심 구조

`TR_CHK_LIST`의 중복점검 미사용 분기는 각 대상 항목·당직자 행마다 `TSF_SCRTCHK_DTLI`를 두 번 상관 조회했다.

```sql
And (
    Not Exists (
        Select 1  -- 지정 당직자 존재 여부
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
        Select 1  -- 현재 당직자가 지정 대상인지 여부
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

### 측정 결과

| 항목 | 결과 |
| --- | ---: |
| 반환 행 수 | 26 |
| 실행 시간 | 0.06초 |
| 총 Buffers | 12,094 |
| 주요 반복 접근 | `TSF_SCRTCHK_DTLI`의 Id 62~65 |

`TSF_SCRTCHK_DTLI` 관련 경로는 167회 및 300회 시작했고, 인덱스 스킵 스캔과 범위 스캔이 반복됐다.

## 1. 1차 수정 쿼리

### 변경

조회 대상 회차의 `TSF_SCRTCHK_DTLI`만 CTE로 추출하고, 항목별 지정 당직자 존재 여부와 사용자별 지정 여부를 각각 집계했다.

```sql
TR_SCRTCHK_DTLI_CNT As (
    Select
        a.COMPANY_ID,  -- 회사 ID
        a.PROJ_CODE,  -- 현장 코드
        a.ASSMNT_SEQ,  -- 회차 번호
        a.ASSMNT_LIST_SEQ,  -- 점검 항목 번호
        a.RISK_FCT_SEQ,  -- 위험요인 번호
        a.IMPRV_MTHD_SEQ,  -- 개선대책 번호
        a.CHECK_DATE,  -- 점검 일자
        Count(*) USER_CNT  -- 지정 당직자 수
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
        a.COMPANY_ID,  -- 회사 ID
        a.PROJ_CODE,  -- 현장 코드
        a.ASSMNT_SEQ,  -- 회차 번호
        a.ASSMNT_LIST_SEQ,  -- 점검 항목 번호
        a.RISK_FCT_SEQ,  -- 위험요인 번호
        a.IMPRV_MTHD_SEQ,  -- 개선대책 번호
        a.CHECK_DATE,  -- 점검 일자
        a.USER_NO  -- 지정 당직자 번호
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

`TR_CHK_LIST`에서는 두 CTE를 외부 조인하고 다음 조건으로 기존 업무 규칙을 보존했다.

```sql
And (e.USER_CNT Is Null Or f.USER_NO Is Not Null)
```

- `e.USER_CNT Is Null`: 항목에 지정 당직자가 없으므로 해당 일자 당직자 전체에 배분한다.
- `f.USER_NO Is Not Null`: 지정 당직자가 있으면 현재 당직자와 일치하는 경우만 남긴다.

### 측정 결과

| 항목 | 원본 | 1차 수정 | 변화 |
| --- | ---: | ---: | ---: |
| 반환 행 수 | 26 | 26 | 동일 |
| 실행 시간 | 0.06초 | 0.05초 | 0.01초 감소 |
| 총 Buffers | 12,094 | 10,685 | 1,409 감소 (약 11.7%) |

1차 실행계획에는 CTE용 임시 결과가 추가됐지만, 전체 정렬·집계 구간의 Buffers가 `3,153`에서 `1,649`로 감소했다.

## 1.5. 1차 개선 근거와 설명 포인트

### 왜 두 집합으로 나눴는가

원본은 `TR_CHK_LIST`의 후보 행마다 `TSF_SCRTCHK_DTLI`를 두 번 상관 서브쿼리로 확인했다.

1. 해당 점검항목·점검일자에 지정 당직자가 한 명이라도 있는지 확인한다.
2. 지정 당직자가 있으면 현재 당직자(`c.USER_NO`)가 그 대상에 포함되는지 다시 확인한다.

후보 행 수가 늘어나면 같은 점검 상세 범위를 반복 탐색할 수 있다. 1차 개선은 먼저 `TR_TIME` 범위와 `CHECK_TF = 'T'` 조건으로 대상 상세를 제한한 뒤, 존재 여부와 사용자별 배정 여부를 조인 가능한 집합으로 만들었다. 따라서 `TR_CHK_LIST`에서는 상관 서브쿼리 대신 외부 조인 결과로 기존 규칙을 판정한다.

| 집합 | 역할 | 최종 조건에서의 의미 |
| --- | --- | --- |
| `TR_SCRTCHK_DTLI_CNT` | 항목·일자별 지정 당직자 존재 여부 | `e.USER_CNT Is Null`이면 지정 당직자가 없으므로 모든 당직자에게 항목을 배정 |
| `TR_SCRTCHK_DTLI_USER` | 항목·일자·사용자별 지정 여부 | `f.USER_NO Is Not Null`이면 지정된 현재 당직자에게만 항목을 배정 |

`And (e.USER_CNT Is Null Or f.USER_NO Is Not Null)`은 위 두 경우를 합쳐 원래 `Not Exists Or Exists`와 같은 업무 규칙을 유지한다.

### 외부 조인과 최종 조건의 역할

`TR_CHK_LIST`는 점검항목과 해당 일자의 당직자 조합마다 "이 항목을 이 사용자에게 배정할지"를 판단한다. `e`, `f`는 모두 외부 조인(`(+)`)이므로 조인 자체는 행을 제거하지 않고, 마지막 조건에서 배정 여부를 결정한다.

- `e`는 항목·일자까지만 조인한다. 즉, 해당 항목에 지정 당직자가 한 명이라도 있는지를 확인한다.
- `f`는 항목·일자와 현재 당직자(`c.USER_NO`)까지 조인한다. 즉, 지정 당직자 중 현재 사용자가 포함되는지를 확인한다.

| 항목의 지정 당직자 상태 | `e.USER_CNT` | `f.USER_NO` | `e.USER_CNT Is Null Or f.USER_NO Is Not Null` | 처리 |
| --- | --- | --- | --- | --- |
| 지정 당직자 없음 | `Null` | `Null` | 참 | 해당 일자의 모든 당직자에게 배정 |
| 현재 사용자가 지정 당직자임 | 값 있음 | 값 있음 | 참 | 현재 사용자에게 배정 |
| 지정 당직자는 있으나 현재 사용자가 아님 | 값 있음 | `Null` | 거짓 | 현재 사용자를 제외 |

`f.USER_NO Is Not Null`만 사용하면 "지정 당직자가 아예 없는 경우"와 "지정 당직자는 있지만 현재 사용자가 아닌 경우"가 모두 `Null`이 되어 구분되지 않는다. 그래서 `e.USER_CNT Is Null`로 지정 당직자 자체가 없는 경우를 따로 허용한다. 현재 `USER_CNT`는 수치가 아니라 존재 여부만 판별하므로, 논리상으로는 `HAS_ASSIGNEE` 역할에 가깝다.

### CTE 사용에 대한 판단

`With` 절의 CTE가 항상 임시 테이블로 물리화되는 것은 아니다. Oracle 옵티마이저가 인라인 처리하거나 내부 임시 결과를 사용할 수 있으므로, "CTE로 빼면 항상 비용이 줄어든다"고 설명하면 안 된다.

이 경우의 비용 감소 근거는 `TR_SCRTCHK_DTLI_CNT`와 `TR_SCRTCHK_DTLI_USER`가 각각 한 번만 조인되는 재사용 자체가 아니라, 후보 행별로 반복되던 두 상관 서브쿼리를 사전에 범위 제한·집계한 집합 조인으로 바꾼 데 있다. 실제 적용 효과는 실행계획과 측정값으로만 판단한다.

### 측정으로 확인된 효과와 한계

- 동일 파라미터·동일 데이터 시점에서 반환 건수는 `26`건으로 같았다.
- 총 Buffers는 `12,094`에서 `10,685`로 `1,409` 감소했다(약 `11.7%`).
- 실행시간은 `0.06초`에서 `0.05초`로 줄었다. 이 차이만으로 일반적인 성능 우위를 단정하지 않는다.
- 데이터가 작거나 기존 `Exists` 탐색이 충분히 선택적이면 원본 방식이 같거나 더 유리할 수 있다. 배포·추가 변경 전에는 대표 조건으로 반환 건수, Buffers, Starts, 실행시간을 다시 비교한다.

### 후속 검토 항목

현재 `USER_CNT`는 수치 자체가 아니라 `Is Null` 여부만 사용한다. 따라서 이후 측정 시 `Count(*)` 대신 존재 플래그 또는 `Distinct` 키 집합으로 표현했을 때의 실행계획·Buffers도 비교할 수 있다. 단, 결과 규칙을 바꾸지 않는 것이 우선이며, 측정 근거 없이 힌트로 물리화를 강제하지 않는다.

## 2. 2차 수정 쿼리

### 변경

다음 병목인 `TSF_SERCURITYCHECK_DETAIL`을 회차 범위로 먼저 한정하여 `TR_PIC`과 중복점검 사용 분기에서 재사용하도록 변경했다.

```sql
TR_SERCURITYCHECK_DETAIL As (
    Select
        a.COMPANY_ID,  -- 회사 ID
        a.PROJ_CODE,  -- 현장 코드
        a.ASSMNT_SEQ,  -- 회차 번호
        a.CHECK_DATE,  -- 점검 일자
        a.USER_NO,  -- 당직자 번호
        a.DLY_SUB_TF  -- 일일 제출 여부
    From
        TSF_SERCURITYCHECK_DETAIL a,
        TR_TIME b
    Where 1 = 1
    And a.CHECK_TF = 'T'
    And a.COMPANY_ID = b.COMPANY_ID
    And a.PROJ_CODE = b.PROJ_CODE
    And a.ASSMNT_SEQ = b.ITEM_SN
)
```

`TR_PIC` 및 중복점검 사용 분기의 `TSF_SERCURITYCHECK_DETAIL e`를 위 CTE로 대체했다. `DLY_SUB_TF`는 중복점검 분기의 `Nvl(e.DLY_SUB_TF, 'F')`에 필요하므로 반드시 포함해야 한다.

### 검증 상태

- 최초 실행 오류: `ORA-00904: "E"."DLY_SUB_TF": invalid identifier`
- 원인: CTE에 `DLY_SUB_TF`가 누락됨
- 조치: CTE 선택 컬럼에 `a.DLY_SUB_TF` 추가
- 현재 상태: XML 문법 검증 완료, 실제 DB 재측정 대기

## 힌트 정책

1차·2차 초안에서 `/*+ Materialize */`를 사용해 CTE 물리화를 유도했으나, 현재 표준에 따라 해당 힌트는 제거했다. 향후에는 논리적 재작성으로 개선하고, 힌트는 재현 가능한 측정 근거와 회귀 위험 기록이 있는 예외 상황에서만 사용한다.

## 재측정 체크

1. 원본과 같은 파라미터 및 데이터 시점으로 실행한다.
2. 결과 행 수와 주요 조치율 값이 1차 수정 결과와 같은지 비교한다.
3. 총 `Buffers`, 실행 시간, `TSF_SERCURITYCHECK_DETAIL`의 Starts·Buffers를 비교한다.
4. 실행계획은 [실행계획 분석 템플릿](query-execution-plan-analysis-template.md) 절차로 남긴다.
