# rules/test

테스트 방법론. 모든 API는 2중 게이트를 통과해야 "온전한" 것으로 본다.

## 기본 자세

- **Write tests. Not too many. Mostly integration.** 통합 테스트가 가장 큰 비중. 순수 함수에는 단위 테스트. 핵심 사용자 흐름에는 소수의 E2E.
- 테스트 없이 API가 머지되지 않는다. 코드와 테스트는 같은 PR에서 들어간다.
- 테스트가 LLM에게는 사양서이기도 하다. 어떤 동작을 원하는지 모호하면, 코드 전에 테스트부터 쓴다.

## 2중 게이트

API가 "온전함"을 보장하려면 두 관문을 모두 통과해야 한다.

### Gate 1 — 배포 전 (CI)

CI에서 다음이 모두 초록불일 때만 배포 단계로 진행한다:

- 단위 + 통합 테스트
- 린트 + 포맷
- 타입 체크
- (해당 시) 컨트랙트 테스트
- (해당 시) 보안 스캔

하나라도 실패하면 배포가 막힌다. CI를 우회한 수동 배포 금지.

### Gate 2 — 배포 후 (Smoke)

배포 직후 프로덕션 또는 스테이징 환경에 curl로 직접 친다. 실패하면 자동 롤백.

최소 스모크 스크립트:

```bash
set -euo pipefail
BASE_URL="${BASE_URL:?must be set}"
curl -fsSL --max-time 10 "$BASE_URL/health" >/dev/null
curl -fsSL --max-time 10 "$BASE_URL/api/v1/ping" >/dev/null
curl -fsSL --max-time 15 -H "Authorization: Bearer $SMOKE_TOKEN" \
     "$BASE_URL/api/v1/<critical-endpoint>" >/dev/null
echo "smoke passed"
```

GitHub Actions에서는 `deploy` 잡 다음에 `smoke` 잡을 두고, `if: failure()` 트리거로 `./scripts/rollback.sh`를 호출한다.

스모크 통과 전까지 "배포 완료"라고 보고하지 않는다.

## 하네스 엔지니어링

첫 번째 기능 테스트를 쓰기 전에 다음을 먼저 만든다. 하네스 없이 시작하면 모든 테스트가 점점 더 깨지기 쉬워진다.

- **픽스처 자동 발견**: pytest `conftest.py`, JUnit 5 `@BeforeEach`, Vitest `setupFiles`
- **팩토리 레이어**: factory_boy (Python), factory-bot (TS), Java test data builders. 같은 객체 리터럴을 두 군데 이상 쓰면 팩토리로 빼낸다.
- **실제 의존성 컨테이너화**: Testcontainers로 DB, 큐, 캐시를 띄운다. AWS는 LocalStack 또는 moto. `latest` 태그 금지, digest로 핀.
- **시간·랜덤 동결**: 테스트에서 `Date.now()`, `random()`을 직접 부르지 않는다. 주입 가능한 시간/난수 소스로 추상화한다.

## Hermetic 원칙

테스트는 완전히 봉인되어야 한다.

- 공유 DB 금지 (테스트마다 격리된 트랜잭션 또는 컨테이너)
- 공개 인터넷 호출 금지 (HTTP는 모두 mock)
- 벽시계 시간 의존 금지
- 테스트 순서 의존 금지
- 전역 상태 공유 금지

## 빠른 테스트와 느린 테스트의 분리

내부 루프(inner loop)는 **30초 이내**로 유지한다.

- 단위 테스트는 기본. 항상 빠르게 실행.
- 통합 테스트는 별도 마크: `@pytest.mark.slow`, JUnit `@Tag("slow")`, Vitest skip-by-tag.
- E2E는 별도 CI 잡으로 분리.

`npm test`는 빠른 것만, `npm run test:all`이 전체.

## 정적 검사는 비협상

가장 싼 테스트는 컴파일러가 해주는 것이다.

- **타입 체커**: mypy / pyright (Python), tsc (TypeScript), javac + ErrorProne (Java)
- **린터/포맷터**: ruff (Python), Biome 또는 ESLint+Prettier (TS), Spotless (Java)
- 이 셋 모두 CI에서 머지를 막는다.

## 무엇을 테스트하는가

- **도메인 규칙**: 값 객체의 불변식, 엔티티의 상태 전이
- **유스케이스의 입출력**: 입력 → 출력, 던지는 예외, 포트 호출
- **어댑터 경계**: 매퍼, 스키마 검증, 외부 에러의 도메인 에러 변환
- **HTTP 계층**: 컨트롤러를 통해 통합 테스트
- **구현 세부사항은 테스트하지 않는다.** private 메서드, 함수 호출 순서, 내부 자료구조에 의존하는 검증은 리팩토링을 막는다. 행위만 검증한다.

## 테스트 구조와 이름

### Arrange-Act-Assert

세 블록을 빈 줄 하나로 분리한다. 한 `it`/`test` 블록은 하나의 시나리오만 검증한다.

### 이름: should-when 패턴

형식: `should <기대 결과> when <조건>`. `describe`는 대상(클래스·유스케이스), `it`은 시나리오.

예: `should throw ConflictError when the email is already taken`

## 의존성 다루기

- `application` 레이어 테스트는 포트의 **in-memory 구현을 직접 작성**해 주입한다. 모킹 라이브러리보다 우선.
- 외부 라이브러리 모킹은 `vi.mock` / `unittest.mock` 등으로 같은 파일 안에서만 사용한다.

## 종류별 테스트 — 언제 쓰는가

### 컨트랙트 테스트 (Pact)

내가 양쪽을 모두 통제하는 서비스 경계에는 모두 둔다. 컨슈머 테스트 실행 시 컨트랙트가 생성되고, 프로바이더의 CI에서 검증된다.

### 스냅샷 테스트

- 좁고 안정적인 출력(API 응답 모양, 작은 컴포넌트)에만 쓴다.
- 페이지 전체 같은 큰 스냅샷은 금지. 깨질 때 누구도 안 본다.
- `--update-snapshots`를 무지성으로 돌리지 않는다. diff를 사람이 읽고 커밋한다.

### Property-based 테스트

순수 함수에 명확한 불변식이 있으면 쓴다. round-trip(직렬화·역직렬화), 멱등성, 가환성, 정렬 보존 등.

- Python: Hypothesis
- TypeScript: fast-check
- Java: jqwik

### 뮤테이션 테스트

라인 커버리지보다 강한 품질 지표. 핵심 모듈에 주기적으로 돌린다.

- Java: PIT
- TypeScript: Stryker
- Python: mutmut

## 도구 매트릭스

| 프레임워크 | 단위/통합 | HTTP/Mock | E2E | 실제 의존성 |
|------------|-----------|-----------|-----|-------------|
| Spring Boot | JUnit 5, AssertJ, `@WebMvcTest`, `@DataJpaTest` | MockMvc, RestAssured, Mockito | Playwright Java | Testcontainers |
| FastAPI / Python | pytest, pytest-asyncio | `httpx.AsyncClient` + `ASGITransport`, respx | Playwright Python | Testcontainers-python, LocalStack, moto |
| Next.js / Nest.js / TS | Vitest 또는 Jest, Testing Library | Supertest (Nest), MSW, `vi.mock` | Playwright | Testcontainers-node, LocalStack |
| AWS | — | aws-sdk-client-mock (TS), moto (Py) | — | LocalStack, SAM `local invoke` |

## 성능 게이트

핵심 엔드포인트마다 k6 thresholds를 둔다.

```js
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};
```

threshold 위반은 non-zero exit로 배포를 막는다. 결과는 `docs/test/perf/`에 아카이브한다.

## 아카이브: docs/test/

모든 의미 있는 테스트 실행, 고친 버그, 성능 측정, 설계 결정은 `docs/test/`에 마크다운으로 남긴다. 작업이 끝난 뒤가 아니라 작업하면서 같이 쓴다.

### 디렉터리 구조

```
docs/test/
├── README.md                     ← 색인
├── runs/  YYYY-MM-DD-<feature>.md ← 의미 있는 테스트 실행
├── bugs/  NNNN-<short-slug>.md    ← 버그 하나당 파일 하나
├── perf/  baselines.md            ← 현재 성능 기준선
│         YYYY-MM-DD-<endpoint>.json
├── adr/   NNNN-<decision>.md      ← MADR 형식 ADR
└── postmortems/  YYYY-MM-DD-<incident>.md
```

### 버그 한 건의 템플릿

`docs/test/bugs/NNNN-<slug>.md`:

```markdown
# Bug NNNN: <짧은 제목>

- 날짜: YYYY-MM-DD
- 심각도: P0 / P1 / P2 / P3
- 영향 범위: <서비스, 엔드포인트>

## 어떤 작업이었는가
사용자 또는 시스템이 무엇을 하려 했는지.

## 무엇이 문제였는가
증상과 재현 절차.

## 왜 그랬는가
근본 원인. trace, log 링크.

## 어떻게 고쳤는가
변경 요약과 PR/커밋 링크. 회귀 테스트가 추가됐다는 점을 명시.

## API 성능 결과
- 수정 전 p50 / p95 / p99 latency
- 수정 후 p50 / p95 / p99 latency
- 에러율 변화
- k6 결과 링크

## 후속 작업
필요한 추가 작업. 연관된 ADR이 있으면 링크.
```

### ADR

비자명한 설계 결정은 MADR 형식으로 `docs/test/adr/NNNN-<decision>.md`에 남긴다. 섹션:

- Context and Problem
- Considered Options
- Decision Outcome
- Consequences

### 포스트모템

프로덕션 인시던트는 **blameless**(개인 책임 추궁 없이) 포스트모템을 `docs/test/postmortems/`에 남긴다. 타임라인, 근본 원인, 기여 요인, 액션 아이템, 회귀 테스트가 들어간다.

## 카나리 + 피처 플래그

- 배포와 릴리스를 분리한다. 피처 플래그로 새 코드를 비활성 상태로 배포.
- 5%부터 트래픽을 흘리고, 메트릭이 안 좋으면 자동 롤백.
- 합성 모니터링(synthetic check)이 `/health` + 핵심 비즈니스 경로 한 가지를 지속적으로 확인.

## "테스트를 빼먹지 않는다"는 규칙

다음 상황이 모두 충족되지 않으면 작업이 끝난 게 아니다:

1. 새 코드 경로마다 테스트가 있다.
2. CI에서 모두 통과한다.
3. 배포 후 curl 스모크가 통과한다.
4. 의미 있는 변경이면 `docs/test/`에 기록이 남았다.

이 네 가지 중 하나라도 빠지면 사용자에게 "완료"라고 보고하지 않는다.
