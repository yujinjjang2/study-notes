# 사용자-근로자 연결설정 전체 시나리오

- 작성일: 2026-09-01
- 화면: `BasePopRegCompanyConnWk.xml`
- 연계 소스: `BasePopRegCompanyConnWkController`, `BasePopRegCompanyConnWkServiceImpl`, `BasePopRegCompanyConnWkSql.xml`
- 목적: 현장사용자를 기존 근로자에 연결하거나 신규 근로자를 생성해 연결하는 흐름을 이해한다.

## 데이터 구조

| 역할 | 테이블 | 핵심 키/값 |
| --- | --- | --- |
| 사용자 계정 | `TCC_EMPLOYE` | `COMPANY_ID`, `USER_NO`, `USER_ID` |
| 근로자 마스터 | `TWTLB_WK` | `WK_NO`, `WK_ID`, `MOBILE_NO`, `BT_DT` |
| 기업 현장 근로자 | `TWTLB_PWK` | `COMPANY_ID`, `PROJ_CODE`, `WK_NO` |
| 협력사 소속 및 사용자 연결 | `TWTLB_PWK_COOP` | 현장·근로자 키, `COOP_COMPANY_ID`, `CONN_COMPANY_ID`, `CONN_USER_NO` |

```text
TCC_EMPLOYE (협력사 사용자)
  COMPANY_ID + USER_NO
       │ CONN_COMPANY_ID + CONN_USER_NO
       ▼
TWTLB_PWK_COOP (현장/협력사 소속 근로자 행)
       │ WK_NO
       ▼
TWTLB_WK (개인 근로자 마스터)
```

`WK_NO`는 근로자 마스터 번호다. 팝업 초기에는 사용자 정보만 전달받고 기존 근로자를 선택하지 않았으므로 보통 `WK_NO`가 없다.

## 팝업 초기화

`initW()`는 부모 화면에서 전달한 기업/현장/협력사와 사용자번호·성명·ID·휴대전화번호·이메일·생년월일을 `wdmTrscConnWk`에 넣는다.

```text
초기값: MOBILE_NO, BD, WK_ID, EMAIL, WK_REP_TF, SMR_WK_REP_TF
초기 미설정: WK_NO
```

따라서 `WK_NO`가 비었다는 것은 전화번호가 없다는 뜻이 아니라, **연결할 기존 근로자 레코드가 아직 확정되지 않았다는 뜻**이다.

## 중복검사 흐름

### 입력 중 검사: `linkTag = 'A'`

휴대전화번호 또는 생년월일 변경 후 두 값이 모두 있으면 `selectCdDupVerWkInfo.do`를 호출한다.

1. Service가 휴대전화번호·생년월일을 암호화한다.
2. SQL이 사용 중인 `TWTLB_WK`에서 같은 `MOBILE_NO + BT_DT`를 찾는다.
3. 결과가 있으면 “동일한 근로자입니까?”를 묻는다.
4. 사용자가 확인하면 조회 결과의 `WK_NO`를 팝업 데이터에 넣는다.

이 단계는 기존 근로자를 선택하는 단계일 뿐, 저장하지 않는다. 이후 확인 버튼을 다시 눌러야 연결이 저장된다.

### 확인 버튼 검사: `linkTag = 'B'`

확인 버튼은 먼저 `MOBILE_NO`, `BD` 입력을 검사한다. 그 다음 `WK_NO`가 없으면 동일한 `MOBILE_NO + BD` 중복 조회를 한다.

```text
WK_NO 없음
  └ 근로자 중복 조회
       ├ 중복 있음: 안내 후 중단
       └ 중복 없음: WK_ID 중복 조회
            ├ 사용 중: 안내 후 중단
            └ 사용 가능: trscConnWk.do 호출
```

즉 `A`는 기존 근로자를 선택할 기회를 제공하고, `B`는 확인 시점에 같은 근로자를 새로 생성하지 못하게 막는다.

## 저장 흐름

### 기존 근로자 연결 (`WK_NO` 있음)

화면은 기업·현장·협력사·사용자 정보를 채워 `trscConnWk.do`를 호출한다. 서버 SQL은 `nWkNo := #{WK_NO}`로 기존 근로자 번호를 사용한다.

1. `TWTLB_WK` MERGE: 선택된 근로자 마스터를 갱신한다.
2. `TWTLB_PWK` MERGE: 대상 기업 현장에 해당 근로자를 등록/갱신한다.
3. `TWTLB_PWK_COOP` MERGE: 협력사 소속 행을 만들거나 갱신하고 아래 값으로 사용자를 연결한다.

```text
CONN_COMPANY_ID = COOP_COMPANY_ID
CONN_USER_NO    = USER_NO
```

### 신규 근로자 생성 후 연결 (`WK_NO` 없음)

중복검사를 모두 통과하면 같은 저장 API를 호출한다. SQL은 `SCC_WK.Nextval`로 `nWkNo`를 새로 발급한 뒤 다음을 수행한다.

1. `TWTLB_WK` INSERT: 새 근로자 마스터 생성. 휴대전화번호·생년월일·이메일은 Service에서 암호화한다.
2. `TWTLB_PWK` MERGE: 기업 현장 근로자 등록.
3. `TWTLB_PWK_COOP` MERGE: 협력사 소속 및 현재 사용자 연결.
4. `TWTLB_PWK_COOP_LOG` INSERT: 작업근무지 로그 기록.
5. 대표 근로자가 없으면 조건에 맞는 근로자 중 성명 순 첫 번째 근로자를 대표로 지정.
6. `TCC_EMPLOYE` UPDATE: 사용자 휴대전화번호·생년월일·이메일 갱신.
7. `TCC_USER_PROJ_DR` 재등록: 해당 협력사 현장에 `DR_CD = 'U'` 부여.
8. `TCC_USER_PROJ_CODE` UPDATE: `DLY_SAF_PIC_PSBL_TF = 'F'` 설정.

## 대표 여부

- `WK_REP_TF = 'T'`: 같은 기업·현장·협력사의 기존 대표를 해제한다.
- `SMR_WK_REP_TF = 'T'`: 같은 기업·현장의 기존 총괄대표를 해제한다.
- 저장 후 협력사 대표가 없으면, 해당 협력사의 대상 근로자 중 한 명을 자동 대표로 지정한다.

## 현재 소스의 연결 해제 범위

현재 `trscConnWk` SQL은 대상 `TWTLB_PWK_COOP` 행에 현재 사용자를 연결하지만, 같은 `WK_NO`에 연결된 다른 사용자를 먼저 해제하는 SQL은 없다.

아래는 다른 사용자 연결을 해제하려는 경우의 검토용 SQL이며, 현재 소스에 구현된 SQL은 아니다.

```sql
Update TWTLB_PWK_COOP
Set
    MODUTYCD = 'U',
    MODUSERNO = #{SESSION_USER_NO},
    MODDATE = Sysdate,
    CONN_COMPANY_ID = Null,
    CONN_USER_NO = Null
Where 1 = 1
And CONN_USER_NO Is Not Null
And WK_NO = nWkNo
And CONN_USER_NO != #{USER_NO};
```

이 조건은 같은 근로자에 연결된 다른 사용자만 해제하고 현재 `USER_NO`는 유지한다. 회사 ID 조건이 없으므로, 다른 협력사 소속 행의 다른 사용자도 해제할 수 있다. 실제 반영 전에는 “근로자 1명당 사용자 연결을 전 협력사에 걸쳐 하나만 허용하는가”를 업무 규칙으로 확정해야 한다.
