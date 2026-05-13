# rules/quality

코드 컨벤션, 파일 구조, 계층화, 문서 동기화 규칙.

## 단일 책임 원칙 (SRP) — 정량 기준

다음은 경고 임계치다. 넘으면 분리를 시도한다.

- 함수: **30줄 도달** (목표 20줄)
- 파일: **300줄 도달** (경고), 300줄 이상 500줄 이하 (강한 경고)
- 함수 인자: **3개 도달**
- Cyclomatic complexity: **10 도달**

파일이 300줄을 넘는다면 다음 축으로 분리할 수 있는지 본다: (a) 도메인이 두 개인가, (b) 액션 종류가 너무 많은가, (c) 외부 의존성이 여러 개 섞여 있는가.

함수가 30줄을 넘으면 대체로 하위 단계를 추출할 수 있다. 추출했을 때 이름을 못 짓겠다면 그건 단계가 아니라 흐름의 일부니까 두는 게 낫다.

## 계층화 (Hexagonal / Ports & Adapters)

```
src/
├── modules/<domain>/
│   ├── domain/          외부 의존 0. 엔티티, 값 객체, 도메인 에러
│   ├── application/     유스케이스. 포트 인터페이스 정의
│   │   └── ports/
│   └── infrastructure/
│       ├── inbound/     HTTP 컨트롤러, 큐 컨슈머
│       └── outbound/    Prisma 리포지토리, HTTP 클라이언트 — 포트 구현
├── shared/              2개 이상 모듈에서 쓰는 것만
└── main.ts              의존성 조립
```

**의존 방향** (위반 시 머지 불가):

- `domain` → 누구도 import 안 함. 외부 라이브러리 import 금지.
- `application` → `domain`만 import.
- `infrastructure` → `application`의 포트를 구현. `domain`도 import 가능.
- `application`은 `infrastructure`를 절대 import하지 않는다. 필요하면 포트를 추가한다.

단순 CRUD에 헥사고날을 강제하지는 않는다. 도메인 규칙이 거의 없는 곳에 유스케이스 계층을 만드는 것은 비용만 든다.

## Package-by-feature

폴더의 최상위는 **기능**으로 나눈다. 그 안에서 얕은 계층을 갖는다.

좋은 예:
```
modules/order/
modules/user/
modules/payment/
```

나쁜 예:
```
controllers/
services/
repositories/
```

기능을 통째로 지울 수 있는지가 모듈성의 시험대다. 한 폴더만 지우면 그 기능이 사라지는 구조가 목표.

## 파일·디렉터리 평탄화 금지

같은 종류 파일이 한 폴더에 10~15개를 넘으면 하위 폴더로 나눈다.

테스트 디렉터리도 마찬가지로 도메인별로 나눈다:

```
test/
├── fx/
├── tax/
├── auth/
└── ...
```

`test/` 아래에 수백 개 파일이 평탄하게 놓이는 일은 막는다.

## 네이밍

### 파일명

- **kebab-case**, 역할 suffix를 붙인다: `<domain>-<subject>.<role>.ts`
- 역할 suffix 예: `.controller.ts`, `.service.ts`, `.use-case.ts`, `.repository.ts`, `.client.ts`, `.mapper.ts`, `.schema.ts`, `.types.ts`, `.errors.ts`, `.config.ts`, `.port.ts`, `.test.ts`, `.integration.test.ts`

### 금지 파일명

- 한 단어 일반 명칭: `llm.ts`, `auth.ts`, `db.ts`
- 덤프 그라운드: `utils.ts`, `helpers.ts`, `common.ts`, `misc.ts`, `shared.ts`
- 같은 이름을 폴더로만 구분: `service/llm.ts` + `controller/llm.ts` 금지. 항상 role suffix로 구분.

유틸이 정말 필요하면 무엇을 위한 것인지 이름에 드러낸다: `string-formatter.ts`, `date-range.ts`.

### 식별자

- 클래스 / 타입 / 인터페이스 / enum: `PascalCase`. `I-` 접두사는 쓰지 않는다.
- 변수 / 함수: `camelCase`
- 모듈 상수: `SCREAMING_SNAKE_CASE`
- 한 파일의 주요 export는 하나. 보조 타입은 함께 export 가능.
- `export default` 금지. 이름 있는 export만.

### 언어별 관례

| 언어 | 파일 | 변수·함수 | 클래스 | 상수 |
|------|------|-----------|--------|------|
| TypeScript / JavaScript | `kebab-case.ts` | `camelCase` | `PascalCase` | `UPPER_SNAKE` |
| Python | `snake_case.py` | `snake_case` | `PascalCase` | `UPPER_SNAKE` |
| Java | `PascalCase.java` | `camelCase` | `PascalCase` | `UPPER_SNAKE` |
| SQL | `snake_case.sql` | `snake_case` | — | `UPPER_SNAKE` |

## 타입 분리

타입은 기능 폴더 안의 별도 파일에 둔다. 한 군데에 거대한 `types/` 폴더를 만들지 않는다. 단, 정말 여러 기능에서 공유하는 타입만 `shared/types/`에 둔다.

TypeScript: 객체 모양은 `interface`, 합집합·유틸리티는 `type`.

## 절대 금지

- `any` 타입. `unknown` 또는 구체 타입을 쓴다.
- `console.*`. 구조화 로거(pino, structlog 등)만 쓴다.
- `require()` / CommonJS. ES Modules만 쓴다.
- `export default`
- 매직 넘버. 모듈 상수로 정의한다.
- 중첩 삼항 연산자
- `catch (e) {}` 식의 swallow. 무시할 거면 명시적으로 로깅한다.
- 비밀(API 키, 토큰, DB 비번)을 코드/저장소에 하드코딩

## 피해야 할 패턴

- God Service / God Class. 파일이 300줄을 넘으면 도메인·액션·외부 의존성 축으로 분리한다.
- 상대 경로 3단계 이상 (`../../../`). barrel `index.ts` 또는 path alias(`@`) 사용.
- 동일 파일에서 5개 이상의 다른 도메인을 import.

## 주석

기본 원칙: **코드가 스스로 말하게 한다.** 좋은 이름이 주석을 대체한다.

### 허용

- 파일 최상단 한 줄: 그 파일의 역할 설명. 형식: `// <Role>: <one-line description>`
- WHY를 적는 주석 (의도, 비즈니스 규칙, 트레이드오프)
- 비자명한 알고리즘에 대한 참조 (논문 링크, 위키 항목)
- 공개 API 문서 (JSDoc, Javadoc, docstring)
- `TODO` / `FIXME`와 이슈 URL
- 법적 헤더

### 금지

- WHAT을 적는 주석 (`i++; // i를 1 증가`)
- 변수 이름이나 타입을 재진술하는 주석
- 주석 처리된 코드 (git을 쓴다)
- 코드와 모순된 낡은 주석
- ASCII 배너, 장식용 구분선

LLM이 생성한 코드는 줄마다 자명한 주석을 다는 경향이 있다. 이건 명시적으로 금지한다.

## 입력 검증

- 모든 외부 입력은 inbound 어댑터 **경계에서** 스키마(zod, Pydantic, class-validator 등)로 검증한다.
- 검증 실패는 422로 변환한다.
- 검증된 타입을 `application`으로 넘기고, 그 안에서는 다시 검증하지 않는다.

## 비밀 관리

- 비밀은 환경변수에서만 읽는다. 시작 시 한 번 검증한 뒤 타입이 부여된 `config` 객체로 export한다.
- 비밀, 토큰, 카드/주민번호 등 민감 정보는 절대 로깅하지 않는다.
- 프로덕션 비밀은 Secrets Manager / SSM Parameter Store / Vault에서 읽는다.

## 로깅

- 구조화 로거만 쓴다. JSON 한 줄.
- 모든 요청에 request id를 부여하고 로그에 포함시킨다.
- 4xx 에러는 `warn`, 5xx는 `error`로 기록한다.
- 메시지에 비밀, PII를 넣지 않는다.

## 비동기

- callback 스타일 금지. `async/await` 사용.
- 독립적인 I/O는 `Promise.all`로 병렬화. 단 외부 호출은 동시성 한계를 둔다.
- 다중 쓰기는 트랜잭션으로 묶는다.

## 멱등성

- `PUT`, `DELETE`는 멱등하게 구현한다.
- 결제 등 부수효과가 큰 `POST`는 idempotency key를 받는다.

## 에러 처리

### HTTP 상태 코드

| 상황 | Status |
|------|--------|
| 요청 본문/쿼리 형식이 망가짐 | 400 |
| 토큰 없음 / 만료 / 위조 | 401 |
| 인증됐지만 권한 부족 | 403 |
| 리소스 없음 | 404 |
| 중복, 버전 충돌, 상태 전이 불가 | 409 |
| 스키마 / 유효성 검증 실패 | 422 |
| 요청 횟수 초과 | 429 |
| 예상 못한 내부 오류 | 500 |
| 외부 서비스 장애 | 502 / 503 / 504 |

401은 "credential 자체가 없거나 무효", 403은 "credential은 유효하지만 권한 없음"으로 명확히 구분한다. 입력 검증 실패는 422다. 400이 아니다.

### 응답 포맷

모든 에러 응답은 아래 형식만 쓴다. stack trace, 내부 객체 노출 금지.

```json
{ "error": { "code": "UNAUTHORIZED", "message": "Authentication required." } }
```

`userMessage`는 클라이언트에 노출. 내부 ID, 경로, 스택 정보는 포함하지 않는다. `logMessage`는 디버깅용 상세 정보로 따로 둔다.

### 던지기 규칙

- `domain` / `application`은 도메인 에러만 던진다.
- 외부 라이브러리(ORM, HTTP client) 에러는 outbound 어댑터에서 잡아 도메인 에러로 변환한다.
  - Prisma `P2002` (unique) → `ConflictError`
  - Prisma `P2025` (not found) → `NotFoundError`
  - axios `ECONNREFUSED` → `ServiceUnavailableError(503)`
- 컨트롤러에서 try-catch로 응답을 만들지 않는다. 던지기만 하고 어댑터에 위임한다.

## 문서 동기화

코드와 다음 문서가 어긋난 상태로 머지되는 PR은 통과시키지 않는다.

### README.md

다음 항목을 코드와 동기화 상태로 유지한다:

- 프로젝트 개요와 핵심 기능
- 필요한 환경 변수 (key만 나열. 실제 값은 노출 금지)
- 설치 / 실행 / 테스트 명령어
- 폴더 구조 요약과 도메인 모듈 목록
- 외부 의존성 (DB, 큐, 외부 API 등)

기능 추가/제거, 환경 변수 변경, 실행 명령 변경, 의존성 추가/제거가 있는 PR은 README.md 갱신을 포함한다.

### schema.sql

- 현재 DB 스키마의 **single source of truth**로 다룬다.
- 마이그레이션 적용 후 schema.sql을 재생성해 같은 PR에 커밋한다.
- 테이블 / 컬럼 / 인덱스 / 제약조건 / 외래키가 ORM 정의와 일치해야 한다.
- 스키마 변경이 ORM 정의에는 반영됐는데 schema.sql에는 빠져 있으면, 그 PR은 미완으로 본다.

### 그 외

- `API_LIST.md` 등은 연관 로직 변경과 함께 갱신한다.
- 비자명한 설계 의사결정은 README의 별도 섹션이나 `docs/test/adr/`에 ADR로 남긴다. 주석으로 길게 설명하지 않는다.

## PR / 커밋

- 커밋 메시지: 날짜 + 주요 작업. 예: `260514 add-fx-rate-fetcher`
- 모든 변경된 라인이 사용자의 요청에 직접 닿아 있어야 한다. 인접 코드 "개선"은 별도 PR로 분리한다.
