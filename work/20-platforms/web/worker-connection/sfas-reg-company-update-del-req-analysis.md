# `SfasRegCompanySql.xml` — `updateDelReq` 분석

- 작성일: 2026-09-08
- 소스: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas\src\main\resources\sqlmap\mappers\sfas\SfasRegCompanySql.xml`
- 대상: `updateDelReq` (3352~3812행)
- 호출 흐름: `SfasRegCompanyServiceImpl.regRequest()` → `GB = 'CANCEL'` → `sfasRegCompanyMapper.updateDelReq(amWdmRequest)`
- 분석 범위: 현재 소스의 정적 분석. 실행·운영 DB 검증은 포함하지 않음.

## 한 줄 요약

`updateDelReq`는 원청 현장과 연결된 협력사 현장을 **해지**하면서, 협력사 사용자 권한·조직·현장 정보·근로자 연결·프로젝트 연결 정보를 정리하고 이력을 남기는 단일 PL/SQL 블록이다.

현재 소스에는 연결 행 삭제 뒤 해당 근로자의 남은 사용자 연결이 없으면 `TWTLB_WK.WK_ID`를 `WK-<WK_NO>`로 복원하는 로직도 포함되어 있다(3671~3693행).

## 입력값과 주요 식별자

| 값 | 용도 |
| --- | --- |
| `#{COMPANY_ID}` | 원청 회사 ID |
| `#{PROJ_CODE}` | 원청 현장 코드 |
| `#{R_COMPANY_ID}` | 해지할 협력사 ID |
| `#{REG_SEQ}` | `TSF_CO_REG` 해지 대상 등록 순번 |
| `#{SESSION_COMPANY_ID}`, `#{SESSION_USER_NO}` | 백업 및 수정 이력의 수행자 |
| `idx.LINK_COMPANY_ID`, `idx.LINK_PROJ_CODE` | `TCC_PROJ_LINK`에서 찾은 협력사 회사/현장 |
| `jdx.WK_NO` | 해지 대상 협력사에 연결된 근로자 번호 |

## 전체 처리 흐름

```text
협력사 해지 요청
  → 출입장비 연계 근로자 존재 여부 검증
  → 원청 현장명 조회
  → 해지 대상 협력사 현장(idx) 조회
      → 협력사 조직·사용자(Idy) 및 권한 현장(Idz) 정리
      → 협력사 현장/조직을 해지 상태로 변경
      → 연결된 근로자 행(jdx) 백업·삭제
          → 해당 근로자의 잔여 연결이 0건이면 WK_ID 복원
          → 현장 근로자 대표자 플래그 해제 및 잔여 소속 행 확인
      → 사용자 권한 이력 백업·삭제
      → 현장·회사 연결 정보 삭제
  → 협력사 등록(TSF_CO_REG)을 해지 상태로 변경
```

## 단계별 분석

### 1. 출입장비 연계 근로자 해지 차단 (3368~3388)

`TWTLB_PWK_COOP`와 `TCC_PROJ_CODE`를 조회해 다음 조건의 근로자가 하나라도 있으면 해지를 막는다.

- 해지 대상 원청 현장 및 협력사에 속함
- `TWTLB_PWK_COOP.EQUIP_KEY Is Not Null`
- `TCC_PROJ_CODE.EQ_TF = 'T'`

건수가 0이면 `sDelPsblTf = 'T'`, 아니면 `F`로 판정하고 `F`일 때 `Raise_Application_Error(-20009, ...)`를 발생시킨다. 즉, 출입장비와 연계된 근로자가 있으면 이후의 모든 해지 처리는 실행되지 않는다.

### 2. 해지 대상 협력사 현장 결정 (3390~3425)

원청 현장명은 `sProjNm`에 보관한다. 이후 `TCC_PROJ_LINK`에서 다음 조건의 링크를 찾고, 연결된 협력사 현장 정보를 `idx` 루프로 처리한다.

- 요청 원청: `COMPANY_ID`, `PROJ_CODE`
- 해지 협력사: `LINK_COMPANY_ID = #{R_COMPANY_ID}`
- 하위 링크 여부: `LINK_SUB_TAG = 'F'`
- 협력사 현장 `TCC_PROJ_CODE.USE_TAG = 'T'`

따라서 실제 정리 범위는 단순히 협력사 회사 ID 하나가 아니라, 이 링크 조건으로 확인되는 협력사 현장(`idx.LINK_PROJ_CODE`)이다.

### 3. 협력사 사용자 권한과 조직 정리 (3426~3620)

`Idy`는 협력사에서 원청 현장명으로 시작하는 활성 부서와 그 부서의 구성원을 조회한다. 각 사용자에 대해 `Idz` 루프가 권한을 삭제해도 되는 협력사 현장을 찾는다.

`Idz`의 제외 조건은 다음과 같다.

- 다른 부서 소속 관계로 유지되어야 하는 현장
- `TCC_GROUP_EMPLOYE` / `TCC_GROUP_PROJ` 그룹 권한으로 유지되어야 하는 현장

삭제 및 보정 순서는 다음과 같다.

| 순서 | 대상 | 처리 |
| --- | --- | --- |
| 1 | `TCC_USER_PROJ_DR` | 사용자-현장 DR 권한 삭제 |
| 2 | `TCC_USER_PROJ_CODE` | 사용자-현장 권한 삭제 |
| 3 | `TCC_EMPLOYE.PROJ_CODE` | 삭제한 현장이 현재값이면 `NULL` 처리 |
| 4 | `TCC_COMPANY_MEMBER` | 부서 구성원 관계 삭제 |
| 5 | `TCC_EMPLOYE.DEPT_NO` | 삭제한 부서가 현재값이면 `NULL` 처리 |
| 6 | `TCC_EMPLOYE.PROJ_CODE`, `DEPT_NO` | 다른 권한 현장이 남아 있으면 그 현장/부서로 재설정 |

`nProjCnt > 1`인 경우 `Max(PROJ_CODE)`와 해당 현장의 `Max(DEPT_NO)`를 다음 기본 현장/부서로 선택한다. 이것은 정렬 기준이 아닌 `MAX`에 따른 선택이므로, 업무적으로 어떤 현장을 기본값으로 삼아야 하는지 별도 확인이 필요하다.

### 4. 협력사 현장·부서 해지 처리 (3622~3638)

- `TCC_PROJ_CODE`: 현장명 뒤에 `[해지]`를 붙이고, `USE_TAG = 'F'`, `OK_TAG = 'T'`로 갱신한다.
- `TCC_COMPANY_DEPT`: 원청 현장명으로 시작하는 협력사 부서를 `USE_TAG = 'F'`로 갱신한다.

물리 삭제가 아니라 비활성화로 남기는 처리다.

### 5. 연결 근로자 백업·삭제와 `WK_ID` 복원 (3640~3726)

먼저 `SCC_DTDB.Nextval`을 `nDelBakSn`으로 확보한다. 이후 원청 현장·해지 협력사에 속하면서 사용자 연결이 존재하는 행만 `jdx` 루프로 조회한다.

```sql
And a.CONN_COMPANY_ID Is Not Null
And a.CONN_USER_NO Is Not Null
And a.COMPANY_ID = #{COMPANY_ID}
And a.PROJ_CODE = #{PROJ_CODE}
And a.COOP_COMPANY_ID = idx.LINK_COMPANY_ID
```

각 행의 처리 순서:

1. `LSPWT_TWTLB_PWK_COOP_CONN_D` 호출
   - `DTDB_TWTLB_PWK_COOP_CONN`에 원본을 백업한다.
   - `TWTLB_PWK_COOP`의 해당 행을 물리 삭제한다.
2. `TWTLB_PWK_COOP`에서 동일 `WK_NO`의 잔여 연결 행을 조회한다.
3. 잔여 행 중 `CONN_COMPANY_ID`, `CONN_USER_NO`가 모두 있는 행이 0건이면 `TWTLB_WK.WK_ID = 'WK-' || WK_NO`로 복원한다.
4. `TWTLB_PWK`의 `RPRST_TF`, `GNR_RPRST_TF`를 `F`로 갱신한다.
5. 같은 원청 현장·근로자의 `TWTLB_PWK_COOP` 소속 행이 아예 없으면 `LSPWT_TWTLB_PWK_D`를 호출한다.

### `WK_ID` 복원 로직의 의미 (3671~3693)

```sql
Select
    Count(*)
Into
    nWkConnCnt
From
    TWTLB_PWK_COOP a
Where 1 = 1
And a.CONN_COMPANY_ID Is Not Null
And a.CONN_USER_NO Is Not Null
And a.WK_NO = jdx.WK_NO;

If nWkConnCnt = 0 Then
    Update TWTLB_WK
    Set
        WK_ID = 'WK-' || WK_NO
    Where 1 = 1
    And WK_NO = jdx.WK_NO;
End If;
```

- **삭제 후** 조회하므로 마지막 연결 해제 여부를 실제 상태로 판단한다.
- 사용자별 또는 협력사별이 아니라 **근로자 번호(`WK_NO`)별**로 판정하므로, 복원 대상인 `TWTLB_WK`와 기준이 일치한다.
- 다른 현장/협력사에 연결이 남아 있으면 복원하지 않아 연결된 사용자 ID를 유지한다.
- 이 로직은 공통 삭제 프로시저가 아닌 협력업체해지 호출부에 있어, 다른 삭제 흐름에 의도치 않은 ID 복원을 전파하지 않는다.

### 6. DR 권한 백업 및 현장 연결 삭제 (3728~3800)

`TCC_USER_PROJ_DR` 중 `DR_CD = 'U'`인 협력사 현장 권한을 `DTDB_TCC_USER_PROJ_DR_CONN`으로 백업한 뒤 삭제한다. 같은 `nDelBakSn`을 사용하므로 근로자 연결 삭제 이력과 해지 단위를 연계해 추적할 수 있다.

마지막으로 아래 관계를 물리 삭제한다.

| 테이블 | 삭제 대상 |
| --- | --- |
| `TCC_PROJ_LINK` | 원청 요청 → 협력사 연결 |
| `TCC_PROJ_LINK` | 협력사 요청 → 원청 역방향 연결 |
| `TCC_PROJ_LINK` | 협력사 현장에 속한 나머지 연결 |
| `TCC_CPN_LK` | 원청 현장과 협력사 회사의 확인 연결 |

### 7. 협력사 등록 상태 변경 (3802~3809)

`TSF_CO_REG`의 요청 `COMPANY_ID`, `PROJ_CODE`, `REG_SEQ`에 대해 `STATUS_CD = 'F'`, `REG_DATE = NULL`로 갱신하며 해지를 확정한다.

## 영향 테이블 정리

| 구분 | 테이블 | 처리 방식 |
| --- | --- | --- |
| 사용자 현장 권한 | `TCC_USER_PROJ_DR`, `TCC_USER_PROJ_CODE` | 물리 삭제, 일부 이력 백업 |
| 사용자 기본 소속 | `TCC_EMPLOYE` | `PROJ_CODE`, `DEPT_NO` 초기화 또는 다른 현장으로 변경 |
| 부서 구성 | `TCC_COMPANY_MEMBER` | 물리 삭제 |
| 협력사 현장·부서 | `TCC_PROJ_CODE`, `TCC_COMPANY_DEPT` | 해지/비활성화 |
| 근로자-협력사 연결 | `TWTLB_PWK_COOP` | 연결된 행을 백업 후 물리 삭제 |
| 근로자 ID | `TWTLB_WK` | 마지막 사용자 연결 해제 시 `WK-<WK_NO>`로 복원 |
| 현장 근로자 | `TWTLB_PWK` | 대표자 플래그 해제, 소속 행이 없으면 공통 삭제 프로시저 호출 |
| 연결 관계 | `TCC_PROJ_LINK`, `TCC_CPN_LK` | 물리 삭제 |
| 협력사 등록 | `TSF_CO_REG` | 해지 상태로 변경 |

## 확인할 위험 및 테스트 포인트

1. **서버 검증 범위**: 현재 서버단 차단은 출입장비(`EQUIP_KEY`) 조건이다. 사용자 연결 근로자의 해지 제한이 별도로 필요한지 업무 규칙을 확인한다.
2. **`MAX`로 기본 현장 선택**: 사용자에게 다른 현장 권한이 남은 경우 `MAX(PROJ_CODE)`를 기본 현장으로 선택하는 것이 의도된 정책인지 확인한다.
3. **다건 근로자 연결**: 동일 `WK_NO`가 여러 현장·협력사에 연결된 경우, 마지막 연결을 해지할 때만 `WK_ID`가 복원되는지 검증한다.
4. **중간 오류 롤백**: `Raise_Application_Error` 또는 프로시저 오류 발생 시 ServiceImpl 트랜잭션 단위로 전체 변경이 롤백되는지 통합 테스트한다.
5. **백업 이력 확인**: `DTDB_TWTLB_PWK_COOP_CONN`과 `DTDB_TCC_USER_PROJ_DR_CONN`이 같은 `nDelBakSn`으로 남는지 확인한다.
