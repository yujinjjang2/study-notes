# 업무 분석 노트

회사 소스를 분석하면서 이해한 흐름, 변경 판단, 검증 방법을 기록한다. 원본 소스는 분석에만 참고하고, 이 저장소에는 공개해도 안전한 분석 결과만 남긴다.

## 폴더 구조

```text
work/
├── README.md
├── 00-guides/                 # 작성 규칙, 용어, 문서 템플릿
├── 10-features/               # 업무·기능 주제별 분석
├── 20-platforms/              # 플랫폼에만 해당하는 공통 분석
│   ├── web/
│   │   ├── worker-connection/ # 사용자·근로자 연결
│   │   ├── data-validation/   # 입력값 검증과 데이터 확인
│   │   └── issues/            # WEB 이슈 원인과 조치
│   ├── mobile/
│   │   ├── android/
│   │   └── ios/
│   ├── api/
│   └── report/
├── 30-external-services/      # 외부 시스템 연동
│   ├── external-api/
│   ├── authentication/
│   └── webhook/
├── 40-client-maintenance/     # 고객사 운영 유지보수
│   ├── organization-registration/ # 조직·회사 등록 및 연결
│   ├── incidents/             # 고객사 운영 장애
│   └── operations/            # 운영 절차와 데이터 설정
└── 90-archive/                # 종료·이관된 참고 문서
```

기존 WEB 문서인 `worker-connection`, `data-validation`, `issues`는 `20-platforms/web/`에 정리했다. 새 문서는 아래 기준에 따라 작성한다.

## 작성 및 분류 규칙

- **업무 기능이 중심인 문서**는 `10-features/`에 둔다. 하나의 기능에서 WEB·모바일·API가 함께 동작하면 문서를 플랫폼별로 나누지 않는다.
- **플랫폼 공통 주제**(화면 구성, 모바일 권한·딥링크, API 규약·오류 형식, 리포트 출력)는 `20-platforms/`에 둔다.
- **외부 제공자와의 계약·인증·요청/응답·장애 대응**은 `30-external-services/`에 둔다. 제공자별 문서는 `external-api/<provider>/`로 나눈다.
- **고객사 운영 문의·장애·설정 요청**은 `40-client-maintenance/`에 둔다. 고객사별 폴더 대신 업무 주제별로 분류하고, 고객·담당자·계정·운영 식별값은 일반화한다.
- 문서가 여러 영역에 걸치면 한 곳만 원본으로 정하고, 다른 위치의 README에서 링크한다. 같은 내용을 복사하지 않는다.
- 파일명은 영문 kebab-case로 주제를 먼저 쓴다. 예: `mobile-deep-link-routing.md`, `external-api-provider-authentication.md`.
- 날짜·상태·관련 플랫폼은 문서 상단 메타정보에 기록한다.
- 실제 사용자 정보, 회사 식별값, 인증 정보, 운영 데이터, 원본 소스 전문은 기록하지 않는다. 처리 흐름과 필요한 조건만 일반화해 요약한다.

## 소스 참고 범위

분석할 때는 `C:\Cloudlab-WT IDE-A v2\workspace` 아래의 다음 소스 구분을 참고한다. 이 경로의 파일은 읽기 전용 참고 자료이며, 정리 결과는 반드시 이 저장소에 작성한다.

| 분석 영역 | 참고 위치 |
| --- | --- |
| 백엔드·공통 모듈 | `foundation-module-backend/` |
| 외부·내부 API | `saas-api/` |
| 모바일 | `saas-mobile/` |
| WEB·리포트 | `saas-report/` |

## WEB 기존 문서

- [worker-connection](20-platforms/web/worker-connection/README.md): 사용자-근로자 연결 흐름과 개선 분석
- [data-validation](20-platforms/web/data-validation/README.md): 입력값 검증과 데이터 무결성 분석
- [issues](20-platforms/web/issues/README.md): 발생한 오류의 원인·조치 기록
