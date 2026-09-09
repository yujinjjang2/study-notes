# 근로자 연결해제 후 사용자 ID와 근로자 ID가 같은 경우 분석

- 작성일: 2026-09-07
- 소스: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas`
- 대상 화면
  - 기업 사용자: `BaseRegCompanyProjectUser`
  - 협력사 사용자: `BaseRegCompanyCooperationProjectUser`
- 목적: 근로자 연결을 해제한 뒤에도 `TCC_EMPLOYE.USER_ID`와 `TWTLB_WK.WK_ID`가 같은 값으로 남는 원인을 확인한다.

## 결론

두 화면 모두 **연결 설정 시 근로자 ID에 사용자 ID를 복사**하는 구조이다. 연결해제 시에는 연결 행을 해제하고, 해제 직전 해당 사용자에게 연결된 `TWTLB_PWK_COOP` 행이 **정확히 1건인 경우에만** 근로자 ID를 `WK-<WK_NO>` 형태로 복원한다.

따라서 해제 직전 사용자 기준 연결 건수가 2건 이상이면, 현재 해제 대상 근로자의 연결이 끊겨도 `WK_ID`는 기존 사용자 `USER_ID`와 같은 값으로 남는다.

## 데이터 구조

| 구분 | 테이블 / 컬럼 | 의미 |
| --- | --- | --- |
| 사용자 계정 | `TCC_EMPLOYE.USER_ID` | 기업 또는 협력사 사용자의 로그인 ID |
| 근로자 계정 | `TWTLB_WK.WK_ID` | 근로자의 로그인 ID |
| 사용자-근로자 연결 | `TWTLB_PWK_COOP.CONN_COMPANY_ID`, `CONN_USER_NO` | 현장 근로자-협력사 소속 행에 연결된 사용자 식별값 |
| 연결 대상 근로자 | `TWTLB_PWK_COOP.WK_NO` | `TWTLB_WK.WK_NO`를 참조 |

## ID가 같아지는 시점: 공통 연결 설정 팝업

두 사용자 화면은 모두 `BasePopRegCompanyConnWk.xml`을 통해 근로자 연결을 설정한다.

1. 팝업 초기화 시 `WK_ID`에 사용자 `USER_ID`를 설정한다.
   ```javascript
   wdmTrscConnWk.set("WK_ID", scwin.para.USER_ID);
   ```
2. 저장 직전에도 `WK_ID`를 사용자 `USER_ID`로 다시 설정한다.
   ```javascript
   wdmTrscConnWk.set("WK_ID", scwin.para.USER_ID);
   ```
3. `trscConnWk`는 이 값을 `TWTLB_WK.WK_ID`에 INSERT 또는 UPDATE한다.

그러므로 연결된 사용자와 근로자의 ID가 같은 것은 연결 설정에 따른 정상 데이터 상태이다.

## 기업 사용자: `BaseRegCompanyProjectUser`

### 연결해제 처리

`trscWkConnEnd`는 해제 전에 사용자 기준 연결 건수를 조회한다.

```sql
Select
    Count(*)
Into
    nConnWkCnt
From
    TWTLB_PWK_COOP a
Where 1 = 1
And a.CONN_COMPANY_ID = #{COMPANY_ID}
And a.CONN_USER_NO = #{USER_NO};
```

이후 현재 화면에서 선택한 현장과 근로자(`COMPANY_ID`, `PROJ_CODE`, `WK_CONN_NO`)의 연결 컬럼만 NULL 처리한다.

```sql
Update TWTLB_PWK_COOP
Set
    CONN_COMPANY_ID = Null,
    CONN_USER_NO = Null
Where 1 = 1
And COMPANY_ID = #{COMPANY_ID}
And PROJ_CODE = #{PROJ_CODE}
And WK_NO = #{WK_CONN_NO}
And COOP_COMPANY_ID = #{COMPANY_ID};
```

`nConnWkCnt = 1`일 때만 해당 근로자의 ID를 복원한다.

```sql
If nConnWkCnt = 1 Then
    Update TWTLB_WK
    Set
        WK_ID = 'WK-' || WK_NO
    Where 1 = 1
    And WK_NO = #{WK_CONN_NO};
End If;
```

### ID가 동일하게 남는 조건

해제 전 동일 사용자에 대한 `TWTLB_PWK_COOP` 연결 행이 2건 이상이면 `nConnWkCnt > 1`이므로 ID 복원 분기를 타지 않는다. 따라서 `WK_ID = USER_ID`가 유지된다.

## 협력사 사용자: `BaseRegCompanyCooperationProjectUser`

### 연결해제 처리

처리 방식은 기업 사용자 화면과 동일하다. 차이는 사용자 식별 회사 값으로 `COOP_COMPANY_ID`를 사용한다는 점이다.

```sql
Select
    Count(*)
Into
    nWkConnCnt
From
    TWTLB_PWK_COOP a
Where 1 = 1
And a.CONN_COMPANY_ID = #{COOP_COMPANY_ID}
And a.CONN_USER_NO = #{USER_NO};
```

선택한 현장·근로자의 연결만 NULL 처리한 후, `nWkConnCnt = 1`일 때만 `WK_ID = 'WK-' || WK_NO`로 변경한다.

### ID가 동일하게 남는 조건

기업 사용자 화면과 같다. 해제 전 동일 협력사 사용자에 대한 연결 행이 2건 이상이면 근로자 ID를 복원하지 않는다.

## 해제 후에도 ID가 같은 구체적 상황

| 해제 전 상태 | 해제 후 결과 | 판단 |
| --- | --- | --- |
| 동일 근로자가 다른 현장에도 같은 사용자로 연결됨 | 현재 현장 연결은 해제되지만 `WK_ID` 유지 | 의도된 동작. 다른 현장 연결이 남음 |
| 사용자가 다른 근로자와도 연결됨 | 현재 해제한 근로자의 `WK_ID`도 유지 가능 | 카운트가 `WK_NO`별이 아니라 사용자별이기 때문 |
| 연결 행이 중복되었거나 과거 연결 행이 남아 있음 | `WK_ID` 유지 가능 | 사용자 기준 카운트가 2 이상이 됨 |
| 해제 전 연결이 정확히 1건 | `WK_ID`를 `WK-<WK_NO>`로 복원 | 사용자 ID와 달라짐 |

> 주의: 두 해제 쿼리는 `COUNT(*)`에서 `WK_NO`, `COMPANY_ID`, `PROJ_CODE`를 제한하지 않는다. 반면 실제 NULL 처리는 선택한 한 현장·근로자 행에만 수행한다. 따라서 사용자의 다른 근로자 연결 또는 잔존 연결 행도 ID 복원 여부에 영향을 준다.

## 운영 데이터 확인 SQL

특정 사용자에게 현재 남아 있는 연결 행을 확인한다. 아래 SQL은 Oracle 조인 문법을 사용한다.

```sql
Select
    p.COMPANY_ID,
    p.PROJ_CODE,
    p.COOP_COMPANY_ID,
    p.WK_NO,
    w.WK_ID,
    w.WK_NM,
    e.USER_ID,
    e.USER_NM,
    p.CONN_COMPANY_ID,
    p.CONN_USER_NO
From
    TWTLB_PWK_COOP p,
    TWTLB_WK w,
    TCC_EMPLOYE e
Where 1 = 1
And p.WK_NO = w.WK_NO
And p.CONN_COMPANY_ID = e.COMPANY_ID
And p.CONN_USER_NO = e.USER_NO
And p.CONN_COMPANY_ID = :CONN_COMPANY_ID
And p.CONN_USER_NO = :CONN_USER_NO
Order By
    p.WK_NO,
    p.COMPANY_ID,
    p.PROJ_CODE;
```

해제 직전 이 결과가 2건 이상이면, 현 코드에서는 선택한 근로자 연결을 해제해도 `WK_ID` 복원 쿼리가 실행되지 않는다.

## 관련 소스

| 구분 | 파일 | 주요 위치 |
| --- | --- | --- |
| 공통 연결 설정 팝업 | `BasePopRegCompanyConnWk.xml` | 158, 248행: `WK_ID`에 사용자 ID 설정 |
| 공통 연결 저장 SQL | `BasePopRegCompanyConnWkSql.xml` | 189, 241행: `TWTLB_WK.WK_ID` 저장 |
| 기업 사용자 해제 SQL | `BaseRegCompanyProjectUserSql.xml` | 2319~2366행 |
| 협력사 사용자 해제 SQL | `BaseRegCompanyCooperationProjectUserSql.xml` | 2087~2140행 |
