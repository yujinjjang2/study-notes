# 기존 사용자 연결 해제 사전 안내 설계

## 목적

`BasePopRegCompanyConnWk`의 근로자 연결 설정에서 한 근로자에게 이미 다른 사용자가 연결되어 있으면, 새 연결 저장 시 기존 연결이 해제된다. 저장 전에 이를 사용자에게 안내하고, 확인한 경우에만 저장을 진행한다.

대상 저장 SQL은 `BasePopRegCompanyConnWkSql.xml`의 `trscConnWk`에 있는 아래 처리다.

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
And CONN_USER_NO != #{USER_NO};
```

## 안내 문구

> 해당 근로자는 이미 다른 사용자와 연결되어 있습니다.<br/>계속 진행하면 기존 사용자와의 연결이 해제됩니다.<br/>계속하시겠습니까?

`연결이 끊어집니다`보다 실제 DB 처리와 일치하는 `연결이 해제됩니다`를 사용한다.

## 화면 처리 흐름

```text
팝업 확인
  → 기존 휴대전화번호·생년월일 및 ID 중복 검증
  → 동일 근로자의 다른 사용자 연결 여부 조회
      → OTHER_USER_CONN_TF = F: trscConnWk 저장
      → OTHER_USER_CONN_TF = T: 확인 팝업 표시
          → 취소: 저장하지 않음
          → 확인: trscConnWk 저장 → 기존 사용자 연결 해제
```

경고는 항상 표시하지 않는다. 실제로 기존 연결 해제 UPDATE의 대상이 존재할 때만 표시한다.

## 사전조회 SQL 예시

아래 조회는 현재 연결 해제 UPDATE의 대상 조건과 같은 기준으로 `OTHER_USER_CONN_TF`를 반환한다.

```sql
Select
    Decode(Count(*), 0, 'F', 'T') OTHER_USER_CONN_TF  -- 다른사용자연결여부
From
    TWTLB_PWK_COOP x
Where 1 = 1
And x.CONN_COMPANY_ID Is Not Null
And x.CONN_USER_NO Is Not Null
And x.WK_NO = #{WK_NO}
And x.CONN_USER_NO != #{USER_NO}
```

- `T`: 현재 근로자에 다른 사용자가 연결되어 있어 저장 시 기존 연결이 해제된다.
- `F`: 해제 대상이 없어 안내 없이 저장할 수 있다.
- 기존 사용자 정보는 조회 결과에 포함하지 않는다. 경고에는 연결 해제 사실만 안내하면 된다.

> 주의: 위 조회는 현재 UPDATE와 의도적으로 조건을 맞춘 예시다. 향후 사용자 식별 범위가 `CONN_COMPANY_ID + CONN_USER_NO` 조합이라면, UPDATE와 조회 양쪽을 동일하게 조정해야 한다.

## 수정 대상

### 프런트: `BasePopRegCompanyConnWk.xml`

1. 조회조건 DataMap `wdmScCdConnWkOtherUser` 추가
   - `WK_NO`
   - `USER_NO`
2. 결과 DataList `wdlCdConnWkOtherUser` 추가
   - `OTHER_USER_CONN_TF`
3. 조회 Submission `wsmInqConnWkOtherUser` 추가
   - `/base/popRegCompanyConnWk/selectCdConnWkOtherUser.do`
4. 저장 전 처리 함수 `scwin.requestConnWkSave()` 추가
   - 저장 파라미터를 세팅한 뒤 사전조회를 호출한다.
5. 실제 저장 함수 `scwin.saveConnWk()` 추가
   - `cips.executeSubmission(wsmTrscConnWk)` 호출과 팝업 닫기 처리를 한 곳으로 모은다.
6. 사전조회 완료 이벤트 추가
   - `OTHER_USER_CONN_TF = 'T'`이면 `cips.msg.confirm(안내문구, 콜백함수명)`을 호출한다.
   - 확인 콜백은 `scwin.saveConnWk()`를 호출하고, 취소 시 아무 작업도 하지 않는다.
   - `OTHER_USER_CONN_TF = 'F'`이면 바로 `scwin.saveConnWk()`를 호출한다.

현재 `wsmTrscConnWk`를 직접 호출하는 두 경로는 모두 `scwin.requestConnWkSave()`로 변경해야 한다.

- 기존 근로자(`WK_NO` 존재) 선택 후 확인
- 휴대전화번호·생년월일 및 ID 검증을 끝낸 신규 근로자 연결

두 경로 중 하나만 변경하면 해당 경로에서는 경고가 누락된다.

### 백엔드

새 조회 endpoint를 추가하는 경우 SQL XML만으로는 호출할 수 없다. 아래 파일에 같은 조회 메서드를 추가한다.

- `BasePopRegCompanyConnWkSql.xml`: `selectCdConnWkOtherUser` SELECT
- `BasePopRegCompanyConnWkMapper.java`: Mapper 메서드
- `BasePopRegCompanyConnWkService.java`: Service 인터페이스 메서드
- `BasePopRegCompanyConnWkServiceImpl.java`: Mapper 위임 구현
- `BasePopRegCompanyConnWkController.java`: `/selectCdConnWkOtherUser.do` endpoint 및 `wdlCdConnWkOtherUser` 응답 설정

## 적용 완료 내역

### `BasePopRegCompanyConnWk.xml`

- 사전조회 조건 DataMap `wdmScCdConnWkOtherUser`를 추가했다.
  - `WK_NO`
  - `USER_NO`
- 조회 결과 DataList `wdlCdConnWkOtherUser`를 추가했다.
  - `OTHER_USER_CONN_TF`
- `/base/popRegCompanyConnWk/selectCdConnWkOtherUser.do`를 호출하는 `wsmInqCdConnWkOtherUser` submission을 추가했다.
- 저장 파라미터 설정과 사전조회를 수행하는 `scwin.requestConnWkSave()`를 추가했다.
- 실제 `trscConnWk` 저장 및 팝업 닫기를 수행하는 `scwin.saveConnWk()`를 추가했다.
- `wsmInqCdConnWkOtherUser` 완료 시 다음과 같이 처리한다.
  - `OTHER_USER_CONN_TF = 'T'`: 안내문구를 Confirm으로 표시한다.
  - Confirm 확인: `scwin.wbtnConnWkOtherUserCallback()`에서 저장한다.
  - Confirm 취소: 저장하지 않는다.
  - `OTHER_USER_CONN_TF = 'F'`: 경고 없이 저장한다.
- 기존 `wsmTrscConnWk` 직접 호출 두 곳을 모두 `scwin.requestConnWkSave()` 호출로 변경했다.

### `BasePopRegCompanyConnWkSql.xml`

- `selectCdConnWkOtherUser` 조회를 추가했다.
- `TWTLB_PWK_COOP`에서 현재 `WK_NO`에 연결된 사용자 중 현재 `USER_NO`가 아닌 연결이 하나라도 있으면 `OTHER_USER_CONN_TF = 'T'`를 반환한다.
- 조회 조건은 `trscConnWk`의 기존 사용자 연결 해제 UPDATE 조건과 일치시켰다.

### Java 계층

다음 파일에 `selectCdConnWkOtherUser` 메서드와 응답 연결을 추가했다.

- `BasePopRegCompanyConnWkMapper.java`
- `BasePopRegCompanyConnWkService.java`
- `BasePopRegCompanyConnWkServiceImpl.java`
- `BasePopRegCompanyConnWkController.java`

Controller endpoint는 `/base/popRegCompanyConnWk/selectCdConnWkOtherUser.do`이며, WebSquare 결과 데이터는 `wdlCdConnWkOtherUser`로 반환한다.

## 검증 상태

- 사전조회 submission ID, endpoint, Controller, Service, Mapper, SQL ID, 결과 컬럼 이름의 연결을 정적 검색으로 확인했다.
- Maven 및 Maven Wrapper가 실행 환경에 없어 Java 컴파일 테스트는 수행하지 못했다.
- 기존 WebSquare XML은 원본 인코딩/속성 형식 때문에 표준 XML 파서 검증이 불가했다. 추가한 화면 요소와 JavaScript 흐름은 기존 `cips.msg.confirm` 및 submission 처리 패턴에 맞춰 반영했다.

## 확인 항목

1. 다른 사용자가 연결되지 않은 근로자: 경고 없이 저장되고 새 사용자 연결이 생성된다.
2. 다른 사용자가 연결된 근로자 + 취소: `TWTLB_PWK_COOP`의 기존 연결과 신규 연결 모두 변경되지 않는다.
3. 다른 사용자가 연결된 근로자 + 확인: 기존 행의 `CONN_COMPANY_ID`, `CONN_USER_NO`가 `NULL`이 되고, 대상 행에는 새 사용자 연결이 저장된다.
4. 기존 근로자 선택 경로와 신규 근로자 연결 경로 모두에서 동일하게 동작한다.
5. 동시 저장이 발생할 수 있으므로 사전조회는 안내용으로만 사용한다. 최종 1:1 보장은 `trscConnWk`의 서버 UPDATE가 담당한다.
