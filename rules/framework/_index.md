# rules/framework

프레임워크별 필수 원칙. 작업 중인 코드의 스택에 해당하는 파일을 함께 읽는다. 여기에 담는 것은 "널리 받아들여졌지만 많이 놓치는" 원칙들. 세부 전략·튜토리얼·세팅은 별도 Claude Skill로 분리한다.

## 라우팅

작업 중인 파일 트리에 따라 다음을 참조한다:

- @rules/framework/spring-boot.md — Java + Spring Boot
- @rules/framework/fastapi.md — Python + FastAPI (웹 API), 데이터/ML 스크립트
- @rules/framework/aws.md — AWS 클라우드 네이티브 (Lambda, IAM, VPC, S3, SQS 등)
- @rules/framework/nextjs.md — Next.js (App Router 기준)
- @rules/framework/nestjs.md — Nest.js

여러 프레임워크가 한 프로젝트에 섞여 있으면 해당하는 모든 파일을 함께 따른다. 모노레포에서는 하위 디렉터리에 별도 `CLAUDE.md`를 두어 lazy load 시키는 것이 가장 깔끔하다.

## 공통 원칙 (모든 프레임워크)

프레임워크별 규칙으로 들어가기 전에 적용되는 것들:

- 입력 검증은 경계(adapter)에서 한 번만. 검증된 타입을 도메인으로 넘긴 뒤에는 다시 검증하지 않는다.
- 비밀은 환경 변수 → 타입화된 config 객체로. 코드에 하드코딩 금지.
- 외부 라이브러리 에러는 outbound 어댑터에서 도메인 에러로 변환.
- 컨트롤러는 얇게. 비즈니스 로직은 서비스/유스케이스에.
- DTO와 도메인 객체를 섞지 않는다. 응답에 ORM 엔티티를 직접 노출하지 않는다.

세부는 각 프레임워크 파일에서.
