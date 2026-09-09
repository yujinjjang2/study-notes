# 사용자-근로자 연결 흐름 및 개선 분석

- 작성일: 2026-09-02
- 소스: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas`
- 대상: `BasePopRegCompanyConnWkSql.xml`의 `trscConnWk`
- 목적: 기존 근로자를 새 협력사 사용자에게 연결할 때, 이전 사용자 연결을 해제하고 새 연결만 유지한다.

## 결론

`trscConnWk`는 하나의 PL/SQL 블록에서 근로자, 현장근로자, 현장근로자별 소속업체 정보를 차례로 저장한다. 이전 연결 해제 SQL은 **`TWTLB_PWK_COOP`의 `MERGE` 바로 앞**에 둔다.

```text
근로자 번호 확정(nWkNo)
  → TWTLB_WK 저장
  → TWTLB_PWK 저장
  → 이전 사용자 연결 해제   ← 추가 위치
  → TWTLB_PWK_COOP MERGE로 새 사용자 연결 저장
```

이 순서면 같은 트랜잭션에서 이전 연결을 비운 뒤 현재 선택한 사용자 연결을 저장한다.

## 실제 `trscConnWk` 흐름

1. 전달된 `WK_NO`가 있으면 그 값을 쓰고, 없으면 시퀀스로 새 `nWkNo`를 만든다.
2. 아이디 및 휴대전화번호·생년월일 중복을 검사한다.
3. `TWTLB_WK`와 `TWTLB_PWK`를 `MERGE`한다.
4. `TWTLB_PWK_COOP`를 `MERGE`하며 현재 연결을 저장한다.
   - `CONN_COMPANY_ID = #{COOP_COMPANY_ID}`
   - `CONN_USER_NO = #{USER_NO}`

기존 `MERGE`의 일치 조건은 `COMPANY_ID + PROJ_CODE + WK_NO + COOP_COMPANY_ID`이다. 따라서 **같은 회사·같은 현장·같은 협력사**에서 사용자만 바꾸면 이미 존재하는 한 행이 갱신된다. 예를 들어 `lyjlyjtest1 → 근로자 A` 연결 후 `lyjlyjtest2 → 근로자 A`를 저장하면, 동일 행의 `CONN_USER_NO`가 `test1`에서 `test2`로 바뀐다. 이 경우에는 별도 연결 해제 `UPDATE`가 없어도 첫 번째 사용자가 끊긴 것처럼 보인다.

반면 협력사 소속 행(`COOP_COMPANY_ID`)이 다르면 `MERGE` 대상이 서로 다른 행이므로, 기존 `MERGE`만으로는 이전 행의 연결 값을 비우지 않는다. 아래 `UPDATE`는 이 경우를 포함해 동일 근로자의 다른 연결을 정리하려는 보완이다.

## 적용할 이전 연결 해제 SQL

아래는 공유받은 DB 공통 표준 및 파라미터를 적용한 SQL이다.

```sql
Update TWTLB_PWK_COOP
Set
    MODUTYCD = 'U',
    MODUSERNO = #{SESSION_USER_NO},
    MODDATE = Sysdate,
    CONN_COMPANY_ID = Null,
    CONN_USER_NO = Null
Where 1 = 1
And CONN_COMPANY_ID Is Not Null
And CONN_USER_NO Is Not Null
And WK_NO = nWkNo
And CONN_USER_NO != #{SESSION_USER_NO};
```

### 조건절 의미

| 조건 | 의미 |
| --- | --- |
| `WK_NO = nWkNo` | 저장 로직에서 확정한 현재 근로자만 대상으로 한다. 신규·기존 연결 모두 같은 기준으로 처리하며, 다른 근로자는 변경하지 않는다. |
| `CONN_COMPANY_ID Is Not Null`, `CONN_USER_NO Is Not Null` | 회사와 사용자 모두 실제로 연결된 행만 대상으로 한다. |
| `CONN_USER_NO != #{SESSION_USER_NO}` | 현재 로그인한 사용자와 다른 사용자 연결을 해제한다. |

> 주의: SQL의 비교값은 새로 연결할 대상인 `#{USER_NO}`가 아니라 로그인 사용자 `#{SESSION_USER_NO}`이다. 둘이 항상 같다면 “새 사용자 외 기존 연결 해제”와 동일하게 동작한다. 관리자가 다른 사용자를 대신 연결하는 등 두 값이 다를 수 있다면, 로그인 사용자의 연결은 남을 수 있으므로 의도를 다시 확인해야 한다.

이 조건은 `COMPANY_ID`, `PROJ_CODE`, `COOP_COMPANY_ID` 제한 없이 `WK_NO`만으로 대상을 찾는다. 따라서 협력사·회사·현장이 달라도 같은 근로자에 연결된 다른 사용자 행을 모두 해제할 수 있다. 근로자 한 명의 연결 계정을 사용자 기준으로 하나만 유지하려는 규칙에는 맞지만, 현장별 연결을 유지해야 한다면 범위가 넓다.

## 연결 흐름과 Cover Case 정리

| 구분 | 기존 연결 행과 새 연결 행의 관계 | 기존 `MERGE`만 수행했을 때 | 추가 `UPDATE`가 하는 일 | 결과/확인 포인트 |
| --- | --- | --- | --- | --- |
| 기본: 같은 회사·같은 현장·같은 협력사, 사용자만 다름 | `COMPANY_ID`, `PROJ_CODE`, `WK_NO`, `COOP_COMPANY_ID`가 모두 같음 | 같은 행을 갱신해 `CONN_USER_NO`가 첫 사용자에서 두 번째 사용자로 바뀜 | 같은 근로자의 다른 연결 행도 추가로 비울 수 있음 | `lyjlyjtest1 → A` 뒤 `lyjlyjtest2 → A`이면, 기존 `MERGE`만으로도 test1 연결은 test2로 교체됨. |
| Cover Case 1: 같은 회사의 A협력사 사용자 연결 후 B협력사 사용자 연결 | 기존·신규 `CONN_COMPANY_ID`는 같고 `CONN_USER_NO`와 `COOP_COMPANY_ID`는 다름 | `COOP_COMPANY_ID`가 달라 서로 다른 행이므로, A협력사 행의 연결이 남고 B협력사 행에 새 연결이 생김 | A협력사 행의 `CONN_COMPANY_ID`, `CONN_USER_NO`를 `NULL`로 비움 | 같은 근로자를 협력사 소속 행 여러 개에 연결하지 않으려는 요구를 처리함. |
| Cover Case 2: 다른 회사의 A사용자 연결 후 C사용자 연결 | 기존·신규 `CONN_COMPANY_ID`, `CONN_USER_NO`, `COOP_COMPANY_ID`가 모두 다름 | 서로 다른 협력사 소속 행이므로 이전 A사용자 연결이 남음 | 이전 행의 연결회사·연결사용자를 `NULL`로 비움 | 회사가 달라도 근로자 기준으로 연결을 하나만 유지하려는 요구를 처리함. |

```text
기존 MERGE의 범위: 같은 회사 + 같은 현장 + 같은 협력사 소속 행 1건을 갱신
추가 UPDATE의 범위: 같은 WK_NO의 다른 사용자 연결 행을 해제
```

따라서 이번 보완의 핵심 대상은 **기본 사례가 아니라 Cover Case 1·2처럼 `COOP_COMPANY_ID`가 달라 기존 `MERGE`가 다른 행으로 처리하는 경우**다.

## 확인 시나리오

1. 근로자 A를 회사 1·사용자 1에 연결한다.
2. 같은 근로자 A를 회사 2·사용자 2에 연결한다.
3. `TWTLB_PWK_COOP`에서 근로자 A의 이전 연결 컬럼이 `NULL`인지, 새 연결 컬럼이 회사 2·사용자 2인지 확인한다.
4. 회사·사용자가 같은 상태로 다시 저장해도 현재 연결이 해제되지 않는지 확인한다.
5. 필요하면 이전 사용자의 현장 권한도 함께 해제할지 업무 규칙을 확인한다. 이 SQL은 연결 컬럼만 해제한다.
## 근로자연결설정 전체 시나리오

### 연결 구조

| 구분 | 식별값 | 역할 |
| --- | --- | --- |
| 협력사 사용자 | `TCC_EMPLOYE.COMPANY_ID`, `USER_NO` | 로그인 계정 |
| 근로자 마스터 | `TWTLB_WK.WK_NO`, `WK_ID` | 실제 근로자 개인 정보 |
| 현장 근로자 | `TWTLB_PWK.COMPANY_ID`, `PROJ_CODE`, `WK_NO` | 원청 현장 참여 정보 |
| 협력사 소속/사용자 연결 | `TWTLB_PWK_COOP` | 현장·협력사 소속 및 연결된 사용자 계정 |

```text
협력사 사용자 (TCC_EMPLOYE)
  COMPANY_ID + USER_NO
          │  CONN_COMPANY_ID + CONN_USER_NO
          ▼
현장 근로자 소속 (TWTLB_PWK_COOP) ── WK_NO ──▶ 근로자 마스터 (TWTLB_WK)
```

사용자와 근로자를 직접 연결하는 것이 아니라 원청 현장의 협력사 소속 행에 다음 값을 기록한다.

```text
CONN_COMPANY_ID = COOP_COMPANY_ID
CONN_USER_NO    = USER_NO
```

### 1. 연결 팝업을 연다

`BaseRegCompanyCooperationProjectUser`에서 연결되지 않은 사용자(`WK_CONN_TF = 'F'`)를 선택해 `BasePopRegCompanyConnWk` 팝업을 연다. 원청 현장·협력사 현장·사용자 번호와 사용자명, ID, 휴대전화번호, 이메일, 생년월일이 팝업에 전달되고 근로자 정보의 초기값으로 사용된다.

연결 후에는 목록 조회가 `TWTLB_PWK_COOP`의 연결 키를 찾아 `WK_CONN_TF = 'T'`, `WK_CONN_NO = WK_NO`로 표시한다. 연결된 사용자는 휴대전화번호·생년월일·현장직책을 사용자 화면에서 수정할 수 없다. 근로자 정보와 동기화되는 항목이기 때문이다.

### 2. 동일 근로자가 없으면 신규 생성하여 연결한다

1. 팝업 확인 시 `MOBILE_NO + BD`로 활성 `TWTLB_WK`를 확인한다.
2. 동일 근로자가 없으면 `WK_ID`가 다른 근로자 또는 다른 사용자에게 사용 중인지 확인한다.
3. 통과하면 새 `WK_NO`를 발급하고 `TWTLB_WK`를 생성한다.
4. `TWTLB_PWK`를 `MERGE`하여 원청 현장 참여 정보를 만든다. 근로자대표이면 같은 협력사 소속의 기존 대표는 해제한다.
5. `TWTLB_PWK_COOP`를 `MERGE`하여 협력사 소속과 사용자 연결 키를 저장한다.
6. `TCC_EMPLOYE`의 휴대전화번호·생년월일·이메일을 확정 값으로 갱신하고 사용자 현장직책(`DR_CD = 'U'`)을 등록한다.

### 3. 동일 근로자가 있으면 기존 `WK_NO`를 재사용한다

휴대전화번호 또는 생년월일 변경 시 동일 근로자 조회가 실행된다. 같은 근로자가 있으면 사용자가 동일인 여부를 확인한다. **예**를 선택하면 해당 기존 `WK_NO`가 팝업에 설정되며, 다음 확인 시 신규 근로자 생성 없이 위 2번의 현장 참여·협력사 소속·사용자 연결 처리만 수행한다.

하나의 `WK_NO`는 여러 원청 현장 또는 여러 협력사 소속 행에서 재사용될 수 있다. 다만 한 사용자(`CONN_COMPANY_ID`, `CONN_USER_NO`)는 논리적으로 하나의 `WK_NO`에만 연결되어야 한다.

### 4. 연결된 사용자 정보를 저장한다

`BaseRegCompanyCooperationProjectUserSql.xml`의 `updateUser`는 아래 조건으로 연결 근로자를 찾고, 1건이면 사용자명·ID·휴대전화번호·생년월일·이메일을 `TWTLB_WK`에도 동기화한다.

```sql
Select
    a.WK_NO
Into
    nConnWkNo
From
    TWTLB_PWK_COOP a
Where 1 = 1
And a.CONN_COMPANY_ID = :COMPANY_ID
And a.CONN_USER_NO = :USER_NO
Group By
    a.WK_NO;
```

서로 다른 `WK_NO`가 2건 이상이면 단일 행을 기대하는 `Select ... Into`가 실패하여 `ORA-01422`가 발생한다. 같은 `WK_NO`가 여러 현장/소속에 있는 경우는 `Group By WK_NO` 결과가 1행이므로 이 오류의 원인이 아니다.

### 5. 연결을 해제한다

연결된 사용자(`WK_CONN_TF = 'T'`)만 해제할 수 있다. `trscWkConnEnd`는 선택한 `TWTLB_PWK_COOP` 행의 `CONN_COMPANY_ID`, `CONN_USER_NO`를 `NULL`로 바꾸고, 연결 수가 1건이었으면 `TWTLB_WK.WK_ID`를 `WK-<WK_NO>`로 변경한다. 다른 현장/소속에서 계속 연결 중이면 `WK_ID`는 유지한다. 또한 사용자 현장직책 중 `DR_CD = 'U'`를 삭제한다.

## `scwin.linkTag`의 역할

`BasePopRegCompanyConnWk.xml`의 `scwin.linkTag`는 DB 컬럼 `TCC_EMPLOYE.LINK_TAG`와 다른 **팝업 내부 JavaScript 변수**다. 동일 근로자 중복조회가 어떤 목적에서 실행됐는지 구분한다.

| 값 | 설정 시점 | 조회 결과의 처리 |
| --- | --- | --- |
| `null` | 팝업 최초 진입 | 아직 중복조회 목적이 없는 상태 |
| `'A'` | 휴대전화번호 또는 생년월일 변경 | 동일 근로자가 있으면 동일인 확인 후 기존 `WK_NO`를 선택할 수 있다. |
| `'B'` | `WK_NO`가 없는 상태에서 팝업 확인 | 동일 근로자가 있으면 신규 등록을 중단한다. 없으면 `WK_ID` 중복검증 후 신규 연결을 저장한다. |

`'A'`는 **입력 중 기존 근로자 선택을 위한 조회**, `'B'`는 **저장 직전 신규 근로자 생성 가능 여부 검증**이다. 같은 조회를 재사용하면서도 결과 처리를 다르게 하기 위한 구분값이다.
