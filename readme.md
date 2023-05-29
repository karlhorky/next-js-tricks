# Next.js Tricks

A collection of useful Next.js tricks

## Avoid PostgreSQL `connection slots` Error with Development Server

The Next.js dev server uses Fast Refresh to update the app in the browser when app code is changed, re-running the code to achieve this. If the code has stateful operations in it, including creation of a connection to a database (eg. PostgreSQL, MongoDB), then these stateful operations will be re-run.

In the case of code which connects to a database, what this means is that every time the code is changed (or even every time the file is saved without changes), new database connections will be created and the old connections will not be closed:

`util/database.ts`

```ts
import postgres from 'postgres';

export const sql = postgres(); // 💥 Will lead to PostgreSQL connection slots errors
```

Since PostgreSQL has a maximum of 100 connections by default, these connections accumulating over time will lead to exhaustion of the available connection slots and the following error:

```
PostgresError: remaining connection slots are reserved for non-replication superuser connections
```

This has been documented to happen in both the `pages/` directory and the App Router:

- `pages/` directory: [`vercel/next.js#26427` (discussion)](https://github.com/vercel/next.js/discussions/26427)
- App Router: [`vercel/next.js#45483`](https://github.com/vercel/next.js/issues/45483)

The current workaround to avoid the issue is to store the first database connection in a property of the Node.js `globalThis` object and prevent creation of any further connections if the property already exists:

`util/database.ts`

```ts
import postgres from 'postgres';

declare module globalThis {
  let postgresSqlClient: ReturnType<typeof postgres> | undefined;
}

// Connect only once to the database
// https://github.com/vercel/next.js/discussions/26427#discussioncomment-898067
function connectOnceToDatabase() {
  if (!globalThis.postgresSqlClient) {
    globalThis.postgresSqlClient = postgres();
  }

  return globalThis.postgresSqlClient;
}

export const sql = connectOnceToDatabase();
```

## Improve 3rd-Party Service Performance with Resource Hints

[Resource Hints](https://www.keycdn.com/blog/resource-hints) are instructions contained in HTTP headers or HTML `<link>` tags which direct the browser to connect to required resources such as domains, images, etc. earlier than it would normally, improving performance.

Next.js configures some resource hints by default, but may not add resource hints for everything that you need - eg. Next.js doesn't currently add resource hints for third party services such as Google reCAPTCHA.

For example, to improve the performance of external domain names, pass a `Link` HTTP header with [`preconnect`](https://andydavies.me/blog/2019/03/22/improving-perceived-performance-with-a-link-rel-equals-preconnect-http-header/) or [`dns-prefetch`](https://developer.mozilla.org/en-US/docs/Web/Performance/dns-prefetch) values:

`middleware.ts`

```ts
import { NextResponse } from 'next/server.js';

export function middleware() {
  const response = NextResponse.next();

  // Configure `preconnect` to connect early to Cloudinary,
  // improving performance of images on the first page load
  response.headers.append(
    'Link',
    '<https://res.cloudinary.com>; rel=preconnect;',
  );

  // Configure `preconnect` to connect early to Google
  // reCAPTCHA domains, improving performance of reCAPTCHA
  // widget on the first page load
  //
  // This is configured only for the pathnames that use
  // Google reCAPTCHA, to avoid unnecessary connections
  if (
    [
      '/login',
      '/reset-password-request',
      '/reset-password',
      '/signup',
    ].includes(request.nextUrl.pathname)
  ) {
    response.headers.append(
      'Link',
      '<https://www.google.com>; rel=preconnect;',
    );
    response.headers.append(
      'Link',
      '<https://www.gstatic.com>; rel=preconnect; crossorigin=anonymous',
    );
  }

  // Configure `dns-prefetch` to resolve DNS record for
  // example API, improving performance
  //
  // This is configured as `dns-prefetch` instead of
  // `preconnect because the imaginary API in this case
  // is used later, after the page has loaded
  response.headers.append(
    'Link',
    '<https://api.example.com/>; rel=dns-prefetch',
  );

  return response;
}

export const config = {
  // Run middleware above on all paths
  matcher: '/:path*',
};
```

This is also possible to configure using [the `headers` config option in `next.config.js`](https://nextjs.org/docs/app/api-reference/next-config-js/headers).

### Caveats

- Next.js performs some optimizations already on its own, so make sure you do not duplicate or interfere with these

## Security Headers including Content Security Policy (CSP)

[Security headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security) are HTTP headers that servers can send to configure security policies in the browser.

To respond with security headers in a Next.js app, one method is to use [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) to set headers on the request and response. This will work for routes using [Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#dynamic-rendering) for both the [App Router](https://nextjs.org/docs/app) and the [Pages Router](https://nextjs.org/docs/pages) (caveat: this cannot be used for routes which use [Static Rendering](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#static-rendering-default)).

Below is a recommended configuration for security headers in a Next.js app, if it were using an API and Google reCAPTCHA and Cloudinary as a third party services:

`middleware.ts`

```ts
import { NextRequest, NextResponse } from 'next/server.js';

type CspDirective = {
  name: string;
  values: string[];
};

function formatContentSecurityPolicyHeader(cspDirectives: CspDirective[]) {
  return cspDirectives
    .map((directive) => {
      return `${directive.name} ${directive.values
        .filter((value) => typeof value === 'string')
        .join(' ')}`;
    })
    .join('; ');
}

export function generateCspHeaderAndNonce() {
  // Generate random nonce on every HTTP page load
  const nonce = crypto.randomUUID();

  return {
    nonce: nonce,
    cspHeader: formatContentSecurityPolicyHeader([
      { name: 'default-src', values: ["'self'"] },
      {
        name: 'script-src',
        values: [
          "'self'",
          // Allow script nonce for:
          // - Next.js <NextScript />
          // - other scripts on pages
          `'nonce-${nonce}'`,
          "'strict-dynamic'",
        ],
      },
      {
        name: 'style-src',
        values: [
          "'self'",
          // Allow unsafe inline styles for Next.js injected inline styles
          "'unsafe-inline'",
        ],
      },
      {
        name: 'connect-src',
        values: [
          "'self'",
          // Allow connecting for API
          process.env.NODE_ENV !== 'production'
            ? 'localhost:8000/'
            : 'https://api.example.com/',
        ],
      },
      {
        name: 'img-src',
        values: [
          "'self'",
          'blob:',
          'data:',
          // Allow images for Cloudinary
          'https://res.cloudinary.com/',
        ],
      },
      {
        name: 'frame-src',
        values: [
          // Allow iframes for Google reCAPTCHA
          'https://www.google.com/',
        ],
      },
      { name: 'frame-ancestors', values: ["'none'"] },
      { name: 'worker-src', values: ["'self'", 'blob:'] },
      { name: 'object-src', values: ["'none'"] },
      { name: 'base-uri', values: ["'none'"] },
      { name: 'form-action', values: ["'self'"] },
    ]),
  };
}

export function middleware(request: NextRequest) {
  const { cspHeader, nonce } = generateCspHeaderAndNonce();

  // Clone the request headers
  const requestHeaders = new Headers(request.headers);

  // Set `x-nonce` request header to read in pages with headers()
  requestHeaders.set('x-nonce', nonce);

  // Set the Content-Security-Policy header in the request for
  // App Router routes - this request header will be read in the
  // `renderToHTMLOrFlight` function in Next.js and:
  //
  // 1. The header will be parsed to extract the nonce
  // 2. The nonce will be added to the HTML tags which are
  //    generated by Next.js (eg. the <script /> tags for the
  //    framework)
  //
  // This is required in addition to the CSP response header below
  //
  // Ref: https://github.com/vercel/next.js/blob/29c2e89bd11b644d6665652d3b5c0314fdb0cdc5/packages/next/src/server/app-render/app-render.tsx#L1222-L1227
  // Ref: https://github.com/vercel/next.js/issues/43743#issuecomment-1542712188
  requestHeaders.set('Content-Security-Policy', cspHeader);

  const response = NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  });

  // Security headers
  response.headers.set('Content-Security-Policy', cspHeader);
  response.headers.set(
    'Permissions-Policy',
    'camera=(), microphone=(), geolocation=()',
  );
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set(
    'Strict-Transport-Security',
    // 1 year HSTS (strict HTTPS), including subdomains
    // TODO: Add your domain to preload list: https://hstspreload.org/
    'max-age=31536000; includeSubDomains; preload',
  );
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('X-DNS-Prefetch-Control', 'on');
  response.headers.set('X-Frame-Options', 'DENY');

  return response;
}

export const config = {
  // Run middleware above on all paths
  matcher: '/:path*',
};
```

You can test the security headers configured on your domain using [securityheaders.com](https://securityheaders.com/).

### App Router

For the [App Router](https://nextjs.org/docs/app), the middleware header with the nonce can be accessed and the nonce used with a custom `<Script />`:

`app/page.tsx`

```tsx
import { headers } from 'next/headers';
import Script from 'next/script';

export default function Page() {
  // x-nonce header set in middleware.ts
  const nonce = headers().get('x-nonce');

  return (
    <>
      <div>Home page</div>

      <Script
        src="https://www.google.com/recaptcha/api.js?render=explicit"
        strategy="afterInteractive"
        nonce={nonce}
      />
    </>
  );
}
```

### Pages Router

If you're using the Pages Router, first modify the code from the `middleware.ts` block above:

1. For the `script-src`, `"'unsafe-eval'"` will be required in development for React Fast Refresh (hot reloading):

```ts
{
  name: 'script-src',
  values:
    process.env.NODE_ENV === 'production'
      ? [
          "'self'",
          // Allow script nonce for:
          // - Next.js <NextScript />
          // - other scripts on pages
          `'nonce-${nonce}'`,
          "'strict-dynamic'",
        ]
      : [
          "'self'",
          // Allow eval scripts for React Fast Refresh
          // in Next.js dev server with pages/ directory
          "'unsafe-eval'",
          // Allow domains for any additional scripts such as
          // Google reCAPTCHA (cannot use nonce because of the
          // 'unsafe-eval' above)
          'https://www.google.com/',
          'https://www.gstatic.com/',
        ],
},
```

To set the nonce in the `<script />` tags generated by Next.js, create a custom `Document` component with a `getInitialProps` method to read the header from the request:

`pages/_document.tsx`

```tsx
import Document, {
  DocumentContext,
  Head,
  Html,
  Main,
  NextScript,
} from 'next/document.js';

type Props = {
  nonce?: string;
};

export default function CustomDocument({ nonce }: Props) {
  return (
    <Html>
      <Head nonce={nonce} />
      <body>
        <Main />
        <NextScript nonce={nonce} />
      </body>
    </Html>
  );
}

CustomDocument.getInitialProps = async (context: DocumentContext) => {
  const initialProps = await Document.getInitialProps(context);
  return {
    ...initialProps,
    // x-nonce header set in middleware.ts
    nonce: context.req?.headers['x-nonce'],
  };
};
```

To use the nonce in a page, define a `getServerSideProps` function to read the nonce from the request header:

`pages/index.tsx`

```tsx
import { GetServerSidePropsContext } from 'next';
import Script from 'next/script';

type Props = {
  nonce: string;
};

export default function Page({ nonce }: Props) {
  return (
    <>
      <div>Home page</div>

      <Script
        src="https://www.google.com/recaptcha/api.js?render=explicit"
        strategy="afterInteractive"
        nonce={nonce}
      />
    </>
  );
}

export async function getServerSideProps(context: GetServerSidePropsContext) {
  return {
    props: {
      // x-nonce header set in middleware.ts
      nonce: context.req.headers['x-nonce'],
    },
  };
}
```

### Alternatives

1. Set HTTP headers in Next.js using [the `headers` config option in `next.config.js`](https://nextjs.org/docs/app/api-reference/next-config-js/headers) (caveat: cannot generate a nonce for CSP)
2. Instead of configuring the values for your security headers yourself, use a package such as [`next-secure-headers`](https://www.npmjs.com/package/next-secure-headers) (caveat: non-standard configuration names)

## Type Check Responses from Route Handlers

To add TypeScript type checking of responses in your [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/router-handlers):

1. Upgrade to at least [`next@13.4.4-canary.0`](https://github.com/vercel/next.js/releases/tag/v13.4.4-canary.0) ([PR](https://github.com/vercel/next.js/pull/47526))
2. Set a return value using `NextResponse<ResponseBody>`, where `ResponseBody` is your type for the JSON response body (optionally also wrap `NextResponse` in `Promise` for `async` Route Handler functions):

   `app/api/animals/[animalId]/route.ts`

   ```ts
   import { NextRequest, NextResponse } from 'next/server';
   import { z } from 'zod';
   import { updateAnimalById } from '@/database/animals';

   const animalSchema = z.object({
     firstName: z.string(),
     type: z.string(),
   });

   type Animal = {
     id: number;
     firstName: string;
     type: string;
   };

   type AnimalResponseBodyPut = { error: string } | { animal: Animal };

   export async function PUT(
     request: NextRequest,
     { params }: { params: { animalId: string } },
   ): /* Return type set */ Promise<NextResponse<AnimalResponseBodyPut>> {
     const animalId = Number(params.animalId);

     if (!animalId) {
       return NextResponse.json(
         { error: 'Invalid animal id' }, // ✅ Response body type checked
         { status: 400 },
       );
     }

     const result = animalSchema.safeParse(await request.json());

     if (!result.success) {
       return NextResponse.json(
         // ✅ Response body type checked
         {
           error:
             'Invalid request body: firstName or type missing from request body',
         },
         { status: 400 },
       );
     }

     const updatedAnimal = await updateAnimalById(
       animalId,
       result.data.firstName,
       result.data.type,
     );

     if (!updatedAnimal) {
       return NextResponse.json(
         { error: 'Update of animal failed' }, // ✅ Response body type checked
         { status: 404 },
       );
     }

     return NextResponse.json({ animal: updatedAnimal }); // ✅ Response body type checked
   }
   ```
