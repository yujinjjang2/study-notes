# 모바일 근로자 연결·연결 해제 흐름

## 확인 범위

- 분석 모듈: `C:\Cloudlab-WT IDE-A v2\workspace\saas-mobile`
- 주요 도메인: 현장근로자(`wtlb`), 현장사용자(`sfas`)
- 기준 테이블: `TWTLB_PWK_COOP`
- 연결 식별 컬럼: `CONN_COMPANY_ID`, `CONN_USER_NO`

`saas-mobile`은 Android/iOS 앱 화면 프로젝트가 아니라 모바일용 Spring API 모듈이다. 따라서 이 문서는 앱 화면 동작이 아닌 API·Mapper 기준의 동작을 정리한다.

## 결론

근로자와 현장사용자가 이미 연결된 상태를 조회하고, 관련 데이터를 동기화하거나 연결을 해제하는 로직은 있다. 반면 이 모듈 안에서는 연결 대상을 선택해 `CONN_COMPANY_ID`, `CONN_USER_NO`에 최초 값을 설정하는 **독립된 근로자 연결 API/Mapper 로직은 확인하지 못했다.** 최초 연결 설정은 다른 모듈 또는 별도 화면 흐름에서 담당할 가능성이 있다.

## 연결 상태 조회와 데이터 동기화

`WtlbWorkerSql.xml`은 `TWTLB_PWK_COOP`의 연결 회사·사용자 번호를 조회하여 `IS_USER_CONN`을 구성한다.

- `CONN_USER_NO`가 있으면 사용자 연결 상태로 판단한다.
- 근로자 등록·수정 시 연결 정보가 존재하면 연결된 `TCC_EMPLOYE`의 아이디, 성명, 휴대폰 번호, 생년월일, 이메일을 근로자 정보에 맞춰 갱신한다.
- 현장사용자 수정 시에도 연결된 근로자를 찾아 같은 업무 필드를 동기화한다.

관련 소스:

- `src/main/resources/mapper/oracle/base/moda/wtlb/WtlbWorkerSql.xml`
  - 근로자 등록 후 동기화: `postRestWtlbWorkerItem` 내 근로자연결설정 처리
  - 근로자 수정 후 동기화: `patchRestWtlbWorkerItem` 내 근로자연결설정 처리
- `src/main/resources/mapper/oracle/base/moda/sfas/SfasStandardUserSql.xml`
  - 현장사용자 수정 시 연결 근로자 정보 동기화

## 연결 해제

연결 해제는 현장사용자 수정 및 삭제 흐름에 구현돼 있다.

### 현장사용자 수정 중 현장 이동

`patchRestSfasStandardUserItem`에서 `updSectCode = 'M'`인 경우, 해당 현장사용자에 연결된 근로자 번호를 찾는다. 연결이 있으면 다음 처리를 수행한다.

1. `TWTLB_PWK_COOP.CONN_COMPANY_ID`, `CONN_USER_NO`를 `Null`로 변경한다.
2. 근로자 아이디 `TWTLB_WK.WK_ID`를 `WK-<WK_NO>` 형식으로 복원한다.

아이디 중복 검증에서 연결 전용 예외가 있었던 근로자를 일반 근로자 상태로 되돌리는 처리다.

### 현장사용자 삭제

`deleteRestSfasStandardUserItem`은 연결된 근로자와 연결 현장 수를 확인한다. 삭제 대상 현장에 대한 연결 정보를 해제하고, 연결 현장이 마지막 하나인 경우에만 근로자 아이디를 `WK-<WK_NO>`로 복원한다.

따라서 한 근로자가 여러 현장 연결을 가진 경우, 특정 현장 사용자를 삭제해도 다른 연결을 고려하도록 구현되어 있다.

## 관련 API

현장사용자 API가 연결 해제 흐름을 호출한다.

- `PATCH /v1/rest/su/sfas-standard/user/060/u/item`: 현장사용자 수정
- `DELETE /v1/rest/su/sfas-standard/user/070/u/item`: 현장사용자 삭제

Controller: `src/main/java/com/cloudlab/wt/saas/mobile/moda/sfas/controller/SfasStandardUserController.java`

## 작업 시 확인할 점

- 최초 연결 설정 기능을 변경하거나 재현해야 한다면 `saas-mobile` 밖의 연결 설정 화면/API를 추가로 찾아야 한다.
- 사용자 또는 근로자 정보 수정은 연결된 반대편 계정 정보에 영향을 줄 수 있으므로, 동기화 대상 필드와 아이디 중복 검증을 함께 확인한다.
- 연결 해제 시에는 단일 현장 연결인지 복수 현장 연결인지에 따라 `WK_ID` 복원 여부가 달라진다.
