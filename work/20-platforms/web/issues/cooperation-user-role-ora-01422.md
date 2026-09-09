# 협력사 사용자 현장직책 저장 ORA-01422 분석 및 조치

- 작성일: 2026-08-31
- 관련 이슈: Redmine #36850
- 메뉴: `안전보건관리체계 > 위험성평가(M) > 기준정보 > 현장사용자등록(협력사)`
- 대상: 참조은정보통신(주) 김영우 (`73608899`)
- 상태: 운영 데이터 조치 완료

## 증상 및 원인

현장직책을 현장소장으로 선택하여 저장할 때 `ORA-01422: exact fetch returns more than requested number of rows`가 발생했다.

`BaseRegCompanyCooperationProjectUserSql.xml`의 `updateUser`는 연결된 근로자 번호를 한 개 변수에 조회한다. 그러나 동일한 협력사 사용자 연결에 서로 다른 `WK_NO`가 2건 존재해 단건 조회가 실패했다.

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
    a.WK_NO
;
```

| WK_NO | WK_ID | CONN_COMPANY_ID | CONN_USER_NO | 처리 |
| --- | --- | --- | --- | --- |
| 36514 | 73608899 | NULL | NULL | 소속 현장이 없는 중복 아이디 |
| 40323 | 73608899 | 361448 | 402342 | 유지할 사용자 연결 |
| 40103 | 01073608899-20231229 | 361448 | 402342 | 해제할 중복 사용자 연결 |

## 적용한 DB 조치

소속 현장이 없는 중복 아이디는 `WK-근로자번호`로 변경하고, 등록 대상이 아닌 근로자의 사용자 연결을 해제했다.

```sql
Update TWTLB_WK
Set
    WK_ID = 'WK-36514'
Where 1 = 1
And WK_NO = 36514
;

Update TWTLB_PWK_COOP
Set
    CONN_COMPANY_ID = Null,
    CONN_USER_NO = Null
Where 1 = 1
And WK_NO = 40103
;
```

## 확인

조치 후 연결 사용자 조회는 `WK_NO = 40323` 한 건만 반환되어야 하며, 현장직책 저장 및 재조회가 정상 처리되어야 한다.

```sql
Select
    a.PROJ_CODE,
    a.WK_NO,
    b.WK_NM,
    b.WK_ID,
    a.CONN_COMPANY_ID,
    a.CONN_USER_NO
From
    TWTLB_PWK_COOP a,
    TWTLB_WK b
Where 1 = 1
And a.WK_NO = b.WK_NO
And a.CONN_COMPANY_ID = 361448
And a.CONN_USER_NO = 402342
Order By
    a.PROJ_CODE,
    a.WK_NO
;
```

## 참고

현장소장이라는 직책 값이 원인이 아니라, 직책 저장 시 기존 사용자 전체 저장 로직이 실행되면서 중복 연결 데이터가 확인된 것이다. 임의로 `ROWNUM = 1` 또는 `MAX(WK_NO)`를 사용해 한 건을 고르는 방식은 적용하지 않는다.
