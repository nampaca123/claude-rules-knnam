# FastAPI + Python 데이터/ML

두 부분으로 나뉜다. 위쪽은 FastAPI 웹 API, 아래쪽은 데이터/ML 스크립트.

---

## FastAPI

### 1. Pydantic v2 문법만 쓴다

- `class Config:` → `model_config = ConfigDict(...)`
- `orm_mode` → `from_attributes`
- `schema_extra` → `json_schema_extra`
- `@validator` → `@field_validator`
- `.dict()` → `.model_dump()`
- `.parse_obj()` → `.model_validate()`

v1 문법이 동작하는 것처럼 보이는 코드도 v2에서는 미묘하게 깨진다.

### 2. async def vs def는 안에서 무엇을 하는지로 결정

`async def`는 안에서 `await`할 수 있는 비차단 I/O가 있을 때만. 만약 동기 I/O(`requests`, 동기 SQLAlchemy)나 CPU 작업이라면 그냥 `def`로 선언한다. FastAPI가 알아서 threadpool에서 실행한다.

`async def` 안에서 동기 차단 호출을 하면 이벤트 루프가 멈춘다. 모든 요청이 함께 느려진다.

### 3. response_model을 항상 선언한다

함수 반환 타입 어노테이션 또는 `@router.get(..., response_model=UserOut)`. 출력을 필터링·검증하기 때문에 보안상 중요하다.

같은 도메인에 `UserIn`과 `UserOut`을 분리한다. 입력에는 password가 있지만 출력에는 없는 식.

### 4. APIRouter는 기능별로

```
app/
├── main.py
├── dependencies.py
├── routers/
│   ├── users.py
│   ├── orders.py
│   └── ...
```

`main.py`에서 `app.include_router(users.router, prefix="/users", tags=["users"], dependencies=[Depends(...)])`로 묶는다.

### 5. 의존성은 Depends()로, 모듈 레벨 전역 금지

DB 세션, 설정, 인증은 의존성으로. 테스트에서 `app.dependency_overrides`로 갈아끼울 수 있다.

path operation은 얇은 어댑터. 비즈니스 로직은 서비스/리포지토리 함수.

에러는 `HTTPException` (적절한 상태 코드 함께). `return {"error": ...}` 같은 raw 응답 금지.

### 6. 설정은 pydantic-settings BaseSettings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    database_url: str
    secret_key: SecretStr
```

`@lru_cache`로 한 번만 로드, `Depends`로 주입한다. 핸들러 안에서 `os.environ`을 직접 읽지 않는다.

### 7. BackgroundTasks와 Celery/Arq를 구분한다

- `BackgroundTasks`: 짧고 실패해도 괜찮은 fire-and-forget. 응답 직후 같은 워커에서 실행.
- 무거운 작업, 재시도 필요한 작업, 배포가 죽어도 살아남아야 하는 작업: Celery / Arq / Dramatiq 등 진짜 큐.

이메일 전송에 BackgroundTasks를 썼다가 워커가 죽어 메일이 사라지는 사례가 흔하다.

### 8. ruff + 타입 체커가 기본 도구

- **ruff**가 lint와 format을 모두 한다. flake8, black, isort, pyupgrade를 대체한다. pre-commit과 CI에서 `ruff check --fix`와 `ruff format`.
- 타입 체크는 **mypy 또는 pyright**. ruff는 타입 체커가 아니다.

이 둘 모두 CI에서 머지를 막는다.

### 9. 비동기 테스트는 httpx + ASGITransport

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.mark.anyio
async def test_root():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        response = await ac.get("/")
    assert response.status_code == 200
```

`TestClient`(동기)는 동기 핸들러용. 진짜 async 동작을 검증하려면 위 패턴.

---

## Python 데이터/ML 스크립트

### 1. if __name__ == "__main__": 가드는 필수

`multiprocessing`, PyTorch `DataLoader` 워커, joblib가 모듈을 재실행한다. 가드가 없으면 fork-bomb처럼 무한 실행.

```python
def main() -> None:
    ...

if __name__ == "__main__":
    main()
```

### 2. print 금지, logging 쓴다

```python
import logging
logger = logging.getLogger(__name__)
```

모듈 최상단에서 한 번. 핸들러와 레벨 설정은 entrypoint(`main.py`, `cli.py`) 한 곳에서만.

`print`는 디버깅 흔적으로만, 커밋 직전에 지운다.

### 3. 모든 RNG를 시드한다

재현성이 필요한 모든 실험에서 다음을 한 번에 설정한다:

```python
import os, random
import numpy as np
import torch

SEED = 42
os.environ["PYTHONHASHSEED"] = str(SEED)
random.seed(SEED)
np.random.seed(SEED)
torch.manual_seed(SEED)
torch.cuda.manual_seed_all(SEED)
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False
```

완전한 결정성은 하드웨어가 바뀌면 보장되지 않는다. 결정적 모드는 학습을 느리게 만들 수 있다. README나 ADR에 seed 값을 기록한다.

### 4. 하드코딩된 경로 금지

`pathlib.Path`를 쓰고 프로젝트 루트 기준 상대 경로로 다룬다. 또는 `pydantic-settings`로 외부 설정.

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parent.parent
DATA_DIR = ROOT / "data"
```

`/Users/myname/...` 류 절대 경로가 코드에 박히는 순간 협업이 깨진다.

### 5. 관심사 분리

전형적 ML 프로젝트 구조:

```
project/
├── data/         (raw, processed, interim) — 보통 .gitignore
├── notebooks/    01-eda.ipynb, 02-feature-explore.ipynb (탐색용)
├── src/<project_name>/
│   ├── data/        데이터 로딩
│   ├── features/    전처리
│   ├── models/      훈련
│   ├── evaluation/  메트릭
│   └── cli.py       entrypoint
├── tests/
├── models/       (.gitignore — DVC 또는 S3로)
└── pyproject.toml
```

탐색은 notebook, 안정화되면 `.py`로 승격. notebook 안에 핵심 로직이 머무르지 않게 한다.

### 6. 의존성·데이터·모델 관리

- 의존성은 락 파일로 핀: `uv`, `poetry`, 또는 `pip-tools`. `requirements.txt`만 있고 락 없으면 재현 불가.
- 데이터는 절대 커밋하지 않는다: `data/`, `models/*.pt`, `*.ckpt`, `*.parquet`을 `.gitignore`.
- 큰 산출물은 DVC, S3/GCS, 또는 Git LFS. 콘텐츠 해시 기반 파일명으로 버전 관리.
