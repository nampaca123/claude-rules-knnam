# Next.js (App Router 기준)

13+ App Router를 기준으로 한다. Pages Router 프로젝트라면 일부 규칙이 다르게 적용된다.

## 1. 기본은 Server Component, "use client"는 필요할 때만

기본은 모든 컴포넌트가 Server Component. 다음 중 하나가 필요할 때만 파일 최상단에 `"use client"`:

- React state (`useState`, `useReducer`)
- React effects (`useEffect`)
- 이벤트 핸들러 (`onClick`, `onChange`)
- 브라우저 API (`window`, `localStorage`, `navigator`)

Client Component는 **잎(leaf)**에 둔다. 서버 컴포넌트의 자식으로 client를 받는 식. 큰 서브트리를 통째로 client로 만들지 않는다. 서버에서 가져온 데이터를 props로 내려준다.

## 2. 데이터 페칭은 Server Component에서

```tsx
async function Page() {
  const data = await fetch("...", { next: { revalidate: 60 } });
  ...
}
```

`useEffect` + `fetch` 패턴은 client 부담만 늘리고 SEO도 잃는다.

ORM이나 DB 직접 호출 시 React `cache()`로 같은 렌더 내 중복 호출을 deduplicate.

## 3. 비밀은 서버에 머문다

- `NEXT_PUBLIC_` 접두사 환경 변수만 클라이언트 번들에 들어간다. 그 외는 서버 전용.
- 서버 전용 모듈 최상단에 `import 'server-only'`. 실수로 client에서 import하면 빌드 시점에 에러로 잡힌다.
- Server Component가 fetch한 객체 전체를 Client Component에 props로 통째로 넘기지 않는다. 필요한 필드만 골라서 넘긴다.

```tsx
// 좋음
<ClientCard name={user.name} avatar={user.avatar} />

// 나쁨 - password hash 같은 게 client 번들에 노출
<ClientCard user={user} />
```

## 4. 변경은 Server Action으로

```tsx
async function createOrder(formData: FormData) {
  "use server";
  // 인증 확인, 검증, DB 쓰기
  revalidatePath("/orders");
  redirect("/orders");
}
```

- `<form action={createOrder}>`로 연결.
- Server Action은 본질적으로 POST 엔드포인트. 직접 호출 가능하므로 **매 호출마다 입력 검증(Zod) + 인증/인가 확인**.
- `revalidatePath` / `revalidateTag`로 캐시 무효화 후 필요시 `redirect`.

## 5. 내부 이동은 `<Link>`

`<a href>` 대신 `next/link`. 프리페치, 클라이언트 사이드 전환, partial rendering이 켜진다.

**주의**: `<Link>`의 프리페치는 GET을 보낸다. GET 핸들러에 부수효과(쿠키 변경 등)를 두면 사용자가 링크를 보기만 해도 트리거된다. `/logout` 같은 것은 GET이 아니라 POST + Server Action / Route Handler로.

## 6. 이미지는 `<Image>`

`next/image`만 쓰고 `<img>` 안 쓴다. AVIF/WebP 자동 변환, 반응형 크기, lazy loading, CLS 방지.

원격 이미지는 `next.config.js`의 `images.remotePatterns`에 등록. 와일드카드(`*`)는 가능하면 피한다.

## 7. Route Handler가 pages/api를 대체

App Router에서는 `app/api/.../route.ts`:

```ts
export async function GET(request: Request) { ... }
export async function POST(request: Request) { ... }
```

기본적으로 `GET`은 캐싱이 안 된다. 캐싱을 원하면 `export const dynamic = 'force-static'` 또는 `fetch({ cache: 'force-cache' })`.

## 8. Suspense + Metadata API

- 페이지 옆에 `loading.tsx`를 두면 자동으로 `<Suspense>`가 감싼다.
- 부분 페이지를 따로 스트리밍하려면 명시적 `<Suspense>` 경계.
- SEO 메타데이터는 `export const metadata` 또는 `generateMetadata()`. **Server Component 전용**. Client Component에서는 export할 수 없다.

## 9. 미들웨어는 보안 경계가 아니다

`middleware.ts`는 Edge runtime에서 돈다. 전체 Node API가 없다.

미들웨어를 "인증 차단막"으로만 쓰면 위험하다. 우회 가능한 경우가 있다(예: CVE-2025-29927). 실제 인증·인가는 다음에 있어야 한다:

- Data Access Layer (DB 호출 직전)
- Route Handler 안
- Server Action 안

미들웨어는 가벼운 라우팅 결정(리다이렉트, 헤더 조작)에 쓴다.

## 흔한 안티패턴

- Server Component에서 client-only 라이브러리 import → 빌드 에러
- 한 페이지가 통째로 `"use client"` → 서버 렌더링 이점 손실
- 큰 객체를 Client Component에 props로 전체 전달 → 번들 비대 + 정보 누출
- `cookies()`나 `headers()`를 호출하는 Server Component를 캐싱 가정으로 작성 → 런타임에 dynamic이 되며 의도치 않은 성능 변화
- `next/dynamic` 남용 → 정말 동적 로딩이 필요한 곳에만

## 디렉터리 구조

```
app/
├── (marketing)/         경로 그룹 — URL에 영향 없음
│   ├── page.tsx
│   └── layout.tsx
├── (app)/
│   ├── dashboard/
│   │   ├── page.tsx
│   │   ├── loading.tsx
│   │   └── error.tsx
│   └── ...
├── api/
│   └── webhooks/
│       └── stripe/
│           └── route.ts
└── layout.tsx           루트 레이아웃
```

기능 단위로 라우트 그룹 `(...)`을 활용해 URL을 깨지 않고 코드를 묶는다.
