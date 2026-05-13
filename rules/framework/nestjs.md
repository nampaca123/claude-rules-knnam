# Nest.js

## 1. ValidationPipe를 전역 등록

`main.ts`에서:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

- `whitelist: true`: DTO에 없는 속성 자동 제거
- `forbidNonWhitelisted: true`: 알려지지 않은 속성이 오면 400 반환
- `transform: true`: 페이로드를 DTO 클래스 인스턴스로 변환. 이게 안 켜져 있으면 `class-validator`가 실제로 안 돈다.

## 2. 한 모듈은 한 기능/도메인

```
src/
├── users/
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── dto/
│   └── entities/
├── orders/
│   └── ...
└── common/
```

`UsersModule`, `OrdersModule` 각각이 자기 컨트롤러·서비스·DTO·엔티티를 소유한다. 공통 코드만 `common/` 또는 `shared/`.

## 3. 컨트롤러는 얇게

컨트롤러는 HTTP를 파싱하고 서비스에 위임한다. 비즈니스 로직이 컨트롤러에 들어가면 안 된다. 비즈니스 로직은 `@Injectable()` 프로바이더에.

```ts
@Controller("users")
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.users.create(dto);
  }
}
```

## 4. 생성자 주입 + TypeScript 접근 제어자

`constructor(private readonly users: UsersService) {}`. property injection은 프레임워크 한계 상황에서만.

## 5. 요청·응답마다 DTO

- 요청 DTO에 `class-validator` 데코레이터 (`@IsEmail()`, `@IsNotEmpty()` 등).
- 응답 DTO를 사용하고 ORM 엔티티를 직접 반환하지 않는다.
- 응답 직렬화는 `ClassSerializerInterceptor` + `@Expose` / `@Exclude` 조합.

**함정**: `ClassSerializerInterceptor`는 반환값이 **클래스 인스턴스**일 때만 동작한다. 페이지네이션 래퍼 같은 plain object를 반환하면 직렬화가 적용되지 않는다. 래퍼도 클래스로 만들거나 인터셉터에서 명시적으로 변환.

## 6. 생명주기 훅의 역할을 혼동하지 않는다

| 종류 | 역할 |
|------|------|
| **Middleware** | 요청 전처리. Express/Fastify 수준. `route` 파라미터 모름. |
| **Guard** (`canActivate`) | 인증·인가. true/false 반환. |
| **Interceptor** | 요청 전후 wrap. 응답 변형, 타이밍, 캐싱. |
| **Pipe** | 입력 검증·변환. `ValidationPipe`가 대표. |
| **Exception Filter** | 에러 응답 포맷팅. |

이 다섯을 섞어 쓰는 게 가장 흔한 실수다.

- 인증을 Pipe에 두지 않는다 → Guard
- 응답 모양 변경을 Filter에 두지 않는다 → Interceptor
- 검증을 Middleware에 두지 않는다 → Pipe

## 7. 전역 Exception Filter

```ts
app.useGlobalFilters(new AllExceptionsFilter());
```

`@Catch()` 또는 `@Catch(HttpException)`. 클라이언트에 raw stack trace가 절대 노출되지 않게 한다. 일관된 에러 응답 포맷:

```json
{ "error": { "code": "...", "message": "..." } }
```

## 8. ConfigModule은 isGlobal + validationSchema

```ts
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    PORT: Joi.number().default(3000),
    DATABASE_URL: Joi.string().required(),
    JWT_SECRET: Joi.string().required(),
  }),
});
```

시작 시 fail-fast. 환경 변수가 모자라면 앱이 시작도 못 한다.

코드 안에서 `process.env`를 직접 읽지 않는다. `ConfigService.get('DATABASE_URL')`을 통해 읽는다.

## 9. forwardRef()는 마지막 수단

순환 의존성이 생기면 `forwardRef()`로 우회할 수는 있지만, 그 자체로 설계가 잘못됐다는 신호다.

- 공통 의존성을 별도 모듈로 추출
- 중간 계층 도입
- 이벤트 기반으로 분리 (`EventEmitter2`)

순환 의존성은 request-scope 주입과 만나면 race condition을 만든다.

## 10. Test.createTestingModule으로 테스트

```ts
const module = await Test.createTestingModule({
  providers: [
    UsersService,
    {
      provide: UsersRepository,
      useValue: { findById: jest.fn(), save: jest.fn() },
    },
  ],
}).compile();

const service = module.get(UsersService);
```

TypeORM 리포지토리 mocking은 `getRepositoryToken(Entity)`:

```ts
{
  provide: getRepositoryToken(User),
  useValue: createMockRepository(),
}
```

v8+에서는 `useMocker`로 자동 mocking 가능. 다만 명시적 `useValue`/`useFactory`가 더 안전하다.

## 흔한 안티패턴

- 모든 서비스를 `AppModule`에 등록 → 모듈 격리 잃음
- DTO 없이 `@Body() body: any` → 검증·타입 모두 사라짐
- `@Injectable({ scope: Scope.REQUEST })` 남용 → 성능 저하, 의존성 그래프 복잡화
- Controller 안에서 `@Inject('CONFIG')`로 직접 환경 변수 꺼내기 → `ConfigService` 통해서 가져온다
- 에러를 그냥 `throw new Error("...")` → 적절한 `HttpException` 서브클래스 사용
