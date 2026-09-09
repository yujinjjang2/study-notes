# 현장근로자 휴대전화번호 필수값 검증 분석

- 작성일: 2026-09-01
- 대상 메뉴: 전체근로자조회 / 현장근로자 등록(기업·협력사)
- 대상 소스: `WtlbRegCompanyProjectWorker`, `WtlbRegCooperationProjectWorker`
- 결론: 화면에서는 필수값으로 제어하지만, 서버 및 DB 단계에서 `MOBILE_NO`의 null·공백을 차단하지 않아 저장 가능성이 있다.

## 화면단 검증

두 화면의 `wdlWk` 데이터셋 설정에는 다음과 같이 `MOBILE_NO`가 필수 컬럼으로 포함되어 있다.

```javascript
oNotNullColumns: ["WK_NAME", "WK_ID", "MOBILE_NO", "BD", ...]
```

따라서 일반적인 화면 조작으로 신규·수정 후 저장하면 공통 화면 로직이 휴대전화번호 미입력을 막는 것이 의도된 동작이다.

## 서버 저장 경로

두 Controller는 요청 본문을 별도의 필수값 검증 없이 Service로 전달한다.

| 구분 | 저장 API |
| --- | --- |
| 기업 | `/wtlb/regCompanyProjectWorker/save.do` |
| 협력사 | `/wtlb/regCooperationProjectWorker/save.do` |

각 `ServiceImpl.save()`는 `MOBILE_NO`를 암호화하여 `encMobileNo`에 넣을 뿐, null·빈 문자열·공백 문자열을 거절하지 않는다.

```java
String rawMobileNo = (String) mWdlWk.get("MOBILE_NO");
mWdlWk.put("encMobileNo", (rawMobileNo == null || rawMobileNo.isEmpty()) ? null
        : CoreCipherUtils.encryptOracleAes256(rawMobileNo.replace("-", "")));
```

즉 `null` 또는 `""`은 `encMobileNo = null`이 되어 Mapper로 전달된다. 공백만 입력된 값은 `isEmpty()`가 false이므로 공백 자체가 암호화되어 화면에서는 빈 값처럼 보일 수도 있다.

## SQL 저장 동작

두 SQL Mapper의 신규 등록·수정 SQL 모두 `TWTLB_WK.MOBILE_NO`에 `#{encMobileNo}`를 직접 대입한다.

```text
-- INSERT
MOBILE_NO, ...
#{encMobileNo}, ...

-- UPDATE
MOBILE_NO = #{encMobileNo}
```

`MOBILE_NO + 생년월일` 중복검증도 `a.MOBILE_NO = #{encMobileNo}` 조건이다. Oracle에서 `NULL = NULL`은 참이 아니므로 휴대전화번호가 null인 경우 중복검증으로 차단되지 않는다.

로컬 테스트 스키마의 `TWTLB_WK.MOBILE_NO`는 `VARCHAR2(1000)`이고 `NOT NULL` 제약이 없다. 운영 DB도 같은 제약 상태라면 DB에서도 null 저장을 허용한다. 운영 DB의 실제 컬럼 제약은 확인이 필요하다.

## 왜 화면 필수값이 있어도 저장되는가

화면 필수검증은 브라우저의 JavaScript 실행 결과일 뿐, 서버 API 자체의 규칙이 아니다. 정상 화면에서는 저장 요청 전에 차단되지만 아래 경로는 서버까지 도달할 수 있다.

1. 브라우저 개발자도구·외부 프로그램·자동화가 `save.do`를 직접 호출한 경우
2. 화면 공통 저장 로직이 특정 상황에서 실행되지 않거나, 클라이언트 오류·구버전 화면이 사용된 경우
3. 다른 기능 또는 배치가 동일 테이블을 갱신하면서 null을 전달한 경우
4. 공백 문자열이 화면 검증을 통과하고 암호화되어 저장된 경우

현재 조회 Service는 `MOBILE_NO`가 null 또는 빈 문자열이면 그대로 빈 값으로 반환한다. 암호문 복호화가 실패한 경우에는 빈 값으로 대체하지 않고 예외가 발생하므로, 목록의 빈 칸은 DB에 null·빈 값·공백 값이 저장된 가능성이 높다.

## 권장 조치

1. 두 `ServiceImpl.save()`에서 `rowStatus`가 `C` 또는 `U`일 때 `MOBILE_NO`를 `trim()` 기준으로 검증하고, 비어 있으면 저장 전 예외 처리한다.
2. 기존 빈 데이터를 정리한 뒤 운영 DB의 `TWTLB_WK.MOBILE_NO`에 `NOT NULL` 제약을 추가한다.
3. 수정 저장에서도 null로 덮어쓰지 못하도록 같은 검증을 적용한다.
4. 정상 입력·null·빈 문자열·공백 문자열의 신규/수정 저장을 검증하는 자동 테스트를 추가한다.

## 기존 데이터 확인 SQL

```sql
Select
    a.WK_NO WK_NO,  -- 근로자번호
    a.WK_NM WK_NM,  -- 근로자명
    a.WK_ID WK_ID,  -- 근로자ID
    a.CRTDATE CRT_DATE,  -- 등록일시
    a.MODDATE MOD_DATE,  -- 수정일시
    a.CRTUSERNO CRT_USER_NO,  -- 등록사용자번호
    a.MODUSERNO MOD_USER_NO  -- 수정사용자번호
From
    TWTLB_WK a
Where 1 = 1
And a.MOBILE_NO Is Null
Or Trim(a.MOBILE_NO) Is Null;
```

운영 DB에서 Oracle의 빈 문자열은 `NULL`로 취급된다. 암호화 저장 컬럼인 점을 고려하면, 공백 문자열은 암호문으로 저장될 수 있으므로 필요 시 복호화 기준의 추가 점검도 수행한다.
