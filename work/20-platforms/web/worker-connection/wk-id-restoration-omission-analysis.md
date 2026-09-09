# 협력업체 해지 시 WK_ID 복원 누락 분석

- 작성일: 2026-09-08
- 근거 문서: `WK_ID_복원누락_분석_20260908.html`
- 소스 확인: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas\src\main`
- 핵심 경로: `SfasRegCompanySql.xml`의 `updateDelReq` → `LSPWT_TWTLB_PWK_COOP_CONN_D`
- 분석 범위: 정적 코드 분석. 운영 DB 조회와 실행 재현은 미수행.

## 결론

협력사등록의 **협력업체해지** 경로는 연결된 `TWTLB_PWK_COOP` 행을 삭제하지만, 해당 근로자의 `TWTLB_WK.WK_ID`를 `WK-<WK_NO>`로 복원하지 않는다.

근로자 연결 시에는 `WK_ID`가 사용자 `TCC_EMPLOYE.USER_ID`로 설정된다. 따라서 마지막 연결이 사라진 뒤에도 이 값을 그대로 두면, 연결은 없지만 사용자 ID가 근로자 ID에 점유된 고아 상태가 된다. 이후 같은 ID를 다른 근로자에 연결하거나 등록할 때 통합 중복검증에 걸릴 수 있다.

## 데이터 및 정상 규칙

| 역할 | 테이블/컬럼 | 설명 |
| --- | --- | --- |
| 근로자 마스터 | `TWTLB_WK.WK_NO`, `WK_ID` | 연결 시 `WK_ID`를 사용자 ID로 동기화 |
| 현장·근로자·협력사 관계 | `TWTLB_PWK_COOP.WK_NO`, `COOP_COMPANY_ID` | 근로자의 소속 협력사 행 |
| 사용자 연결 | `TWTLB_PWK_COOP.CONN_COMPANY_ID`, `CONN_USER_NO` | 사용자 계정 식별값. `CONN_COMPANY_ID`에는 협력사 ID가 저장됨 |
| 사용자 계정 | `TCC_EMPLOYE.COMPANY_ID`, `USER_NO`, `USER_ID` | `WK_ID`와 같은 ID 네임스페이스에서 중복검증됨 |

정상 규칙은 다음과 같다.

1. 연결 설정: `CONN_*` 값을 저장하고 `WK_ID = USER_ID`로 동기화한다.
2. 연결 해제: `CONN_*`를 비우거나 행을 삭제한다.
3. 해당 `WK_NO`에 연결된 행(`CONN_USER_NO Is Not Null`)이 더 이상 없을 때만 `WK_ID = 'WK-' || WK_NO`로 복원한다.

기존 일부 경로는 해제 **전** `Count(*) = 1`을 검사한다. 다건 연결이 가능한 현재 구조에서는 해제 후 남은 연결 건수로 판정하는 방식이 더 직접적이고 안전하다.

## 유력 원인: 협력업체해지 경로

`SfasRegCompanySql.xml:updateDelReq`는 해지 대상 협력사(`COOP_COMPANY_ID = idx.LINK_COMPANY_ID`)의 연결 행을 골라 `LSPWT_TWTLB_PWK_COOP_CONN_D`로 백업 후 삭제한다. 그 뒤 현장권한 등 관련 정보를 직접 삭제하지만 `WK_ID` 갱신은 없다.

- `SfasRegCompanyServiceImpl.java:168`: `CANCEL` 처리에서 `updateDelReq` 호출
- `SfasRegCompanySql.xml:3352~3787`: `updateDelReq`
- `SfasRegCompanySql.xml:3659`: `LSPWT_TWTLB_PWK_COOP_CONN_D` 호출
- `SfasRegCompanySql.xml`: `WK_ID` 참조 없음

반면 공통 프로시저 `LSPCC_USER_PROJ_CODE_D`에는 마지막 연결을 해제할 때 `TWTLB_WK.WK_ID = 'WK-' || WK_NO`로 되돌리는 로직이 있다(`TCC_USER_PROJ_CODE.xml:341`). `updateDelReq`는 이 프로시저를 호출하지 않고 `TCC_USER_PROJ_CODE` 등을 직접 삭제하므로, 공통 복원 로직을 우회한다.

## 영향 및 재현

재현 절차는 다음과 같다.

1. 협력사 근로자에 현장사용자를 연결한다.
2. 협력사등록 메뉴에서 해당 협력사를 해지한다.
3. `TWTLB_PWK_COOP`의 연결 행은 삭제됐지만 `TWTLB_WK.WK_ID`가 사용자 `USER_ID`와 같은지 확인한다.
4. 해당 사용자 ID를 다른 근로자에 연결/등록할 때 ID 중복 오류가 나는지 확인한다.

## 운영 데이터 검증 SQL

`LSPWT_TWTLB_PWK_COOP_CONN_D`는 삭제 전 데이터를 `DTDB_TWTLB_PWK_COOP_CONN`에 남긴다. 아래 조회에서 결과가 나오면 협력업체해지 경로로 인한 고아 `WK_ID` 가능성이 높다. Oracle 조인 문법으로 작성했다.

```sql
Select
    w.WK_NO WK_NO,  -- 근로자번호
    w.WK_ID WK_ID,  -- 근로자ID
    e.USER_ID USER_ID,  -- 사용자ID
    e.USE_TAG USE_TAG,  -- 사용여부
    d.DTDB_DTTM DTDB_DTTM,  -- 삭제일시
    d.COMPANY_ID COMPANY_ID,  -- 회사ID
    d.PROJ_CODE PROJ_CODE,  -- 현장코드
    d.COOP_COMPANY_ID COOP_COMPANY_ID,  -- 협력회사ID
    d.CONN_USER_NO CONN_USER_NO  -- 연결사용자번호
From
    TWTLB_WK w,
    DTDB_TWTLB_PWK_COOP_CONN d,
    TCC_EMPLOYE e
Where 1 = 1
And w.WK_NO = d.WK_NO
And e.USER_ID(+) = w.WK_ID
And w.USE_TF = 'T'
And w.WK_ID != 'WK-' || w.WK_NO
And Not Exists (
    Select 1 EXISTS_TF  -- 연결존재여부
    From
        TWTLB_PWK_COOP x
    Where 1 = 1
    And x.WK_NO = w.WK_NO
    And x.CONN_USER_NO Is Not Null
)
Order By
    d.DTDB_DTTM Desc;
```

결과 해석:

| 결과 | 해석 |
| --- | --- |
| `DTDB_TWTLB_PWK_COOP_CONN` 이력 존재 | 협력업체해지 경로 가설을 강하게 지지 |
| 다른 백업 이력만 존재 | 다른 근로자/소속업체 삭제 경로를 추가 조사 |
| 이력 없음 | `UPDATE` 기반 해제 경로의 건수 판정 문제 등 다른 원인 조사 |

## 권장 조치

### 수정 위치

`SfasRegCompanySql.xml`의 `updateDelReq` 안에서, **`LSPWT_TWTLB_PWK_COOP_CONN_D(...)` 호출 직후**에 넣는다.

- 현재 위치: 3659~3668행의 연결 행 백업·삭제 호출 다음
- 삽입 위치: `TWTLB_PWK` 대표자 플래그 갱신(3670행) 전
- 선언 추가: `Declare` 구문에 `nWkConnCnt Number;` 추가

이 시점에는 `jdx`가 가리키던 `TWTLB_PWK_COOP` 행이 이미 삭제됐으므로, 동일 근로자(`jdx.WK_NO`)의 **삭제 후 잔여 연결 수**를 정확히 판정할 수 있다.

### 적용 예시

아래는 삽입할 로직의 예시다. 실제 반영 시 기존 들여쓰기와 감사 컬럼 규칙을 따른다.

다음은 삭제 후 복원 분기를 설명하기 위한 원본 절차 코드 발췌다. 실행용 SQL 예시가 아니므로 원문 형태를 유지한다.

```text
-- Declare 영역
nWkConnCnt Number;

...

LSPWT_TWTLB_PWK_COOP_CONN_D
(
    jdx.COMPANY_ID,
    jdx.PROJ_CODE,
    jdx.WK_NO,
    jdx.COOP_COMPANY_ID,
    nDelBakSn,
    #{SESSION_COMPANY_ID},
    #{SESSION_USER_NO}
);

-- 삭제 후 해당 근로자의 잔여 사용자 연결을 확인
Select
    Count(*)
Into
    nWkConnCnt
From
    TWTLB_PWK_COOP a
And a.CONN_COMPANY_ID Is Not Null
And a.CONN_USER_NO Is Not Null
Where a.WK_NO = jdx.WK_NO;

If nWkConnCnt = 0 Then
    Update TWTLB_WK
    Set
        MODUTYCD = 'U',
        MODUSERNO = #{SESSION_USER_NO},
        MODDATE = Sysdate,
        WK_ID = 'WK-' || WK_NO
    Where 1 = 1
    And WK_NO = jdx.WK_NO;
End If;
```

### 이 위치와 조건을 사용하는 이유

1. **삭제 후 판정이 필요하다.** 기존 일부 해제 경로의 해제 전 `Count(*) = 1` 방식은 다건 연결 구조에서 실제 잔여 연결 상태와 어긋날 수 있다. 삭제 후 `CONN_USER_NO Is Not Null`이 0건인지 확인하면 “마지막 연결 해제 시에만 복원”이라는 업무 규칙을 그대로 표현한다.
2. **근로자 기준으로 판정해야 한다.** 복원 대상은 `jdx.WK_NO`의 `TWTLB_WK.WK_ID`이므로, 사용자나 협력사 전체 건수가 아니라 해당 근로자에게 남은 연결만 센다.
3. **같은 트랜잭션 안에서 처리된다.** 삭제와 `WK_ID` 복원이 `updateDelReq`의 단일 PL/SQL 블록에 포함돼, 중간 상태가 확정되는 것을 막을 수 있다.
4. **공통 삭제 프로시저에는 넣지 않는다.** `LSPWT_TWTLB_PWK_COOP_CONN_D`의 책임은 현재처럼 백업 후 행 삭제로 유지한다. 여러 호출처를 가진 공통 모듈에 `TWTLB_WK` 변경을 넣으면, 다른 업무 흐름에도 의도하지 않은 ID 복원이 발생할 수 있다. 이 협력업체해지 흐름의 호출부가 복원 책임을 갖는 편이 안전하다.

### 후속 권장 사항

향후 16개 연결 해제 경로의 규칙을 통일해야 한다면, 위의 “`WK_NO`별 잔여 연결 0건 확인 후 복원” 로직을 전용 공통 프로시저(예: `LSPWT_WK_ID_SYNC(WK_NO)`)로 분리하고 각 호출부에서 실행한다. 배포 전에는 기존 고아 데이터를 보정하고, 협력업체해지·다건 연결·사용자 재연결 시나리오로 회귀 테스트한다.

## 함께 확인할 별개 위험

- 일부 근로자 삭제 SQL은 `COOP_COMPANY_ID` 없이 `Select ... Into ... Group By`를 수행하고 `TOO_MANY_ROWS`를 처리하지 않는다. 같은 현장·근로자에 소속업체 행이 여러 개면 삭제가 실패할 수 있다.
- `trscMoveUser`의 `CONN_COMPANY_ID` 비교 기준이 세션 회사 ID와 맞지 않을 수 있다. 실제 저장값은 협력사 ID(`COOP_COMPANY_ID`)인지 확인해야 한다.
- 일부 삭제 화면의 “연결 근로자는 삭제 불가” 제약은 JavaScript에만 있어 API 직접 호출이나 동시성 상황에서 서버가 보장하지 못한다.

## 미확인 사항

- 실제 운영 데이터에서 고아 `WK_ID`가 존재하는지
- 문제 사례가 협력업체해지 경로로 발생했는지
- 모바일 등 다른 채널이 `CONN_*` 또는 `WK_ID`를 변경하는지
