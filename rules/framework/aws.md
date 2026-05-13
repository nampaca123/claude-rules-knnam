# AWS 클라우드 네이티브

Lambda, IAM, VPC, S3, SQS 등을 다룰 때 따른다. 비용 사고와 보안 사고가 가장 흔히 일어나는 곳이라 보수적으로 잡았다.

## 1. IAM에 와일드카드 금지

프로덕션 정책에 `"Action": "*"` 또는 `"Resource": "*"`를 쓰지 않는다. IAM Access Analyzer로 CloudTrail 기반 최소 권한 정책을 생성한다.

`"Action": "s3:*"` 같은 서비스 단위 와일드카드도 좋지 않다. 필요한 작업만 명시한다.

## 2. 장기 자격증명 금지

워크로드는 IAM Role + STS 임시 자격증명을 쓴다. 액세스 키는 코드, `.env`, git, Slack에 절대 금지.

사람은 IAM Identity Center로 SSO를 통해 접근. 루트 계정은 봉인하고 MFA 걸어 잠근다.

## 3. 비밀은 Lambda 환경 변수에 평문으로 두지 않는다

Lambda 환경 변수는 콘솔, CLI, CloudFormation 출력에 노출된다. AWS 자체 권고: **Secrets Manager 또는 SSM Parameter Store SecureString**을 쓴다.

Parameters and Secrets Lambda Extension을 쓰면 호출마다 외부 API를 안 가도 캐싱된다.

## 4. SQS visibility timeout ≥ Lambda 타임아웃 × 6

AWS 권고 그대로다. Lambda 함수 타임아웃이 30초라면 SQS visibility timeout은 최소 180초.

AWS는 이를 API에서 강제한다: 함수 타임아웃은 큐의 visibility timeout 이하여야 한다. 안 그러면 처리 중 메시지가 다른 워커에 다시 보이고 중복 처리된다.

## 5. DLQ는 모든 큐와 비동기 호출에 둔다

DLQ가 없으면 독약 메시지가 영원히 순환한다. SQS, SNS, 비동기 Lambda 호출 모두에 설정.

CloudWatch alarm을 DLQ의 `ApproximateNumberOfMessagesVisible`에 건다. 메시지가 들어오면 누군가 봐야 한다.

## 6. 멱등성 처리

SQS, SNS, EventBridge는 at-least-once 전달이다. 같은 메시지가 두 번 올 수 있다는 전제로 핸들러를 짠다.

AWS Lambda Powertools의 `@idempotent` 데코레이터를 쓴다. DynamoDB에 멱등성 키를 저장 + TTL. 키는 안정적인 비즈니스 필드(JMESPath)로 추출. 이벤트 객체 전체를 키로 쓰면 같은 의미의 이벤트가 다른 메타데이터 때문에 새 호출로 인식된다.

## 7. NAT Gateway 비용 함정: S3/DynamoDB는 VPC Gateway Endpoint

VPC 안에서 NAT Gateway를 통해 S3나 DynamoDB로 트래픽이 흐르면 NAT 처리 비용($0.045/GB) + 데이터 송신 비용($0.09/GB)이 쌓인다.

**Gateway Endpoint는 무료**다. 모든 VPC + private subnet 환경에서 S3와 DynamoDB는 Gateway Endpoint를 만든다. 사례에 따라 NAT 비용의 35~45%가 사라진다.

## 8. Lambda 기본 아키텍처는 arm64

Graviton2는 동급 x86 대비 약 19% 빠르고 20% 저렴, 가성비로 최대 34% 우위. 순수 Python/Node 함수는 한 줄 바꾸면 끝.

레거시 native binary 의존성이 있을 때만 x86. ARM 호환 확인 후 가능하면 옮긴다.

## 9. 구조화 JSON 로깅

`print` / `console.log` 금지. AWS Lambda Powertools Logger를 쓴다.

자동으로 `cold_start`, `function_arn`, `request_id`를 주입한다. 상관관계 ID를 지원한다. `POWERTOOLS_SERVICE_NAME` 환경 변수만 설정하면 된다.

CloudWatch Logs Insights에서 JSON 키로 검색·집계 가능. plain text 로그는 운영 중 도움이 안 된다.

## 10. Infrastructure as Code 강제, 태그 정책

콘솔에서 프로덕션 리소스를 만들지 않는다. CDK / Terraform / SAM / CloudFormation 중 하나로.

모든 리소스에 표준 태그 세트:

- `Environment` (prod / staging / dev)
- `Owner` (팀 또는 사람)
- `CostCenter`
- `Application` 또는 `Project`

SCP나 Tag Policy로 강제한다. 비용 분석과 사고 대응에 필수.

## 11. 리전 하드코딩 금지

코드 안에 `"us-east-1"`을 박지 않는다.

- Lambda는 `AWS_REGION` 환경 변수가 자동 세팅됨. 그것을 읽는다.
- CDK/Terraform은 입력으로 받는다.
- 로컬 개발에서는 `AWS_DEFAULT_REGION`.

리전이 박혀 있으면 DR과 로컬 테스트가 깨진다.

## 12. Multi-AZ + S3 Block Public Access

- 프로덕션 DB, ALB, ECS 서비스는 최소 2 AZ에 분산.
- RDS는 Multi-AZ.
- S3 버킷은 Block Public Access의 네 가지 옵션을 모두 켠다. (2023년 4월부터 신규 버킷은 기본 ON. 기존 버킷도 SCP로 강제.)
- 공개해야 하는 정적 자산은 CloudFront + OAI를 통해 노출하고 버킷 자체는 private.

## 추가 권장

- **암호화 at rest**는 모든 곳에서 기본 ON. RDS, EBS, S3, DynamoDB.
- **계정 분리**: AWS Organizations로 dev / staging / prod를 별도 계정. SCP로 보호.
- **CloudFormation Outputs에 비밀 금지.** 필요하면 `NoEcho: true`.
- **API Gateway는 throttling + WAF**. 공개 엔드포인트에는 기본.
- **CloudWatch alarm**을 Lambda errors, throttles, duration p95, DLQ depth에 건다.
- **X-Ray** 또는 ADOT으로 분산 추적 켠다. 운영 중 디버깅이 다른 차원으로 쉬워진다.

## 흔한 사고

- private subnet에서 NAT 없이 외부 API 호출 → 그냥 안 됨
- S3 presigned URL 만료 시간 너무 김 → 의도치 않은 공유
- Lambda concurrent execution 한계로 throttling → reserved concurrency 설정
- CloudWatch Logs 보존 기간 무한 → 비용 누적, 30일~90일이 일반적
