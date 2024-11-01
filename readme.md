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

## GitHub Actions: Automated Upgrades to Next.js Internal React versions

When using [Renovate Bot](https://github.com/apps/renovate) or [Dependabot](https://github.blog/news-insights/product-news/keep-all-your-packages-up-to-date-with-dependabot/) to automate package upgrades, automatic Next.js version upgrades may fail because of mismatching `react` and `react-dom` package versions.

A GitHub Actions workflow can be used to automatically upgrade the `react` and `react-dom` versions in all of a project's `package.json` files to the required version in the Next.js `peerDependencies` (requires [setup of a GitHub fine-grained access token](https://github.com/karlhorky/github-tricks#:~:text=create%20a%20GitHub%20fine%2Dgrained%20personal%20access%20token)):

`.github/workflows/upgrade-to-next-internal-react.yml`

```yaml
name: Upgrade to Next.js internal React package version

on: push

jobs:
  upgrade-to-next-internal-react:
    name: Upgrade to Next.js internal React package version
    runs-on: ubuntu-latest
    # Skip workflow run if last committer was not Renovate Bot
    if: github.event.head_commit.author.name == 'renovate[bot]'
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - uses: pnpm/action-setup@v4

      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 'lts/*'
          cache: 'pnpm'

      - run: pnpm install

      - name: Upgrade to Next.js internal React package version
        run: |
          set -e

          matching_pattern="19.0.0-rc-.*-.*"

          # Find all package.json files, omitting node_modules directories
          packages=$(find . -name 'package.json' -not -path '*/node_modules/*')

          # Check React version in Next.js, if installed
          if [ -f "node_modules/next/package.json" ]; then
            # Extract the React version from peerDependencies
            react_peer_dependency=$(jq -r '.peerDependencies.react // empty' node_modules/next/package.json)
            if [[ -z "$react_peer_dependency" ]]; then
              echo "React peerDependency not found in Next.js package.json"
              exit 0
            fi

            # Extract version after '||' and trim spaces
            next_react_version=$(echo "$react_peer_dependency" | awk -F '\\|\\|' '{gsub(/^[ \t]+|[ \t]+$/, "", $2); print $2}')

            if [[ "$next_react_version" =~ $matching_pattern ]]; then
              echo "React version in Next.js matches: $next_react_version"
            else
              echo "React version in Next.js does not match, aborting."
              exit 0
            fi
          else
            echo "Next.js not installed, skipping Next.js React version check."
            exit 0
          fi

          # For each package.json, check if react/react-dom versions match the pattern, then update only the ones that do
          declare -A lockfiles

          for package in $packages; do
            dir=$(dirname "$package")

            # Initialize the jq script dynamically based on matching dependencies
            jq_script='.'

            # Check React versions in package.json
            version_react=$(jq -r '.dependencies.react // empty' "$package" || true)
            version_react_dom=$(jq -r '.dependencies["react-dom"] // empty' "$package" || true)
            version_dev_react=$(jq -r '.devDependencies.react // empty' "$package" || true)
            version_dev_react_dom=$(jq -r '.devDependencies["react-dom"] // empty' "$package" || true)

            # Update only the versions that match the pattern
            if [[ "$version_react" =~ $matching_pattern ]]; then
              jq_script="$jq_script | .dependencies.react = \$react_version"
            fi

            if [[ "$version_react_dom" =~ $matching_pattern ]]; then
              jq_script="$jq_script | .dependencies[\"react-dom\"] = \$react_version"
            fi

            if [[ "$version_dev_react" =~ $matching_pattern ]]; then
              jq_script="$jq_script | .devDependencies.react = \$react_version"
            fi

            if [[ "$version_dev_react_dom" =~ $matching_pattern ]]; then
              jq_script="$jq_script | .devDependencies[\"react-dom\"] = \$react_version"
            fi

            # Update the package.json if any dependency matched the pattern
            if [ "$jq_script" != '.' ]; then
              echo "Updating $package"
              jq --arg react_version "$next_react_version" "$jq_script" "$package" > tmp.$$.json && mv tmp.$$.json "$package"
            else
              echo "No matching React versions in $package, skipping update."
            fi

            # Run package manager install if package.json was updated or if it is in the root directory
            if [ "$jq_script" != '.' ] || [ "$dir" = "." ]; then
              if [ -f "$dir/package-lock.json" ]; then
                lockfiles["$dir"]="npm"
              elif [ -f "$dir/yarn.lock" ]; then
                lockfiles["$dir"]="yarn"
              elif [ -f "$dir/pnpm-lock.yaml" ]; then
                lockfiles["$dir"]="pnpm"
              fi
            fi
          done

          if [ ${#lockfiles[@]} -eq 0 ]; then
            echo "No packages updated, exiting."
            exit 0
          fi

          # Install dependencies in all locations where package.json was updated
          for dir in "${!lockfiles[@]}"; do
            manager=${lockfiles[$dir]}
            echo "Running $manager install in $dir"
            case "$manager" in
              npm)
                (cd "$dir" && npm install)
                ;;
              yarn)
                (cd "$dir" && yarn install)
                ;;
              pnpm)
                (cd "$dir" && pnpm install --no-frozen-lockfile)
                ;;
            esac
          done

          # Commit changes, if any
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git config user.name "github-actions[bot]"
          git add .

          if git diff-index --quiet HEAD --; then
            echo "No changes to commit, exiting."
            exit 0
          fi

          # Use the React version from Next.js for the commit message
          git commit -m "Upgrade React packages to $next_react_version"

          # Push changes using OAuth2 token
          git push https://oauth2:${{ secrets.UPGRADE_TO_NEXT_INTERNAL_REACT_GITHUB_TOKEN }}@github.com/${{ github.repository }}.git HEAD:${{ github.ref }}
```

This will create commits like this in the repo, which will also trigger any further checks to be run on the commit:

![Screenshot 2024-10-23 at 12 39 33](https://github.com/user-attachments/assets/03eca81f-f746-4930-9c98-52ebc767878b)

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

[Security headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security) are HTTP headers that servers can send to configure security policies in the browser. [securityheaders.com](https://securityheaders.com/) can be used to show the security headers configured on a domain.

To respond with security headers in a Next.js app, one method is to use [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) to set headers on the request and response. This will work for routes using [Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#dynamic-rendering) for both the [App Router](https://nextjs.org/docs/app) and the [Pages Router](https://nextjs.org/docs/pages) (caveat: this cannot be used for routes which use [Static Rendering](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#static-rendering-default)).

Below is a recommended configuration for security headers in a Next.js app, if it were using an API, Google reCAPTCHA and Cloudinary as third party services:

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

### References

- [Next.js CSP example app (App Router) (`Sprokets/nextjs-csp-report-only`)](https://github.com/Sprokets/nextjs-csp-report-only)
- [Securing your Next.js 13 application - Engineering with Yagiz](https://www.yagiz.co/securing-your-nextjs-13-application)
- [Understanding and implementing a proper Content Security Policy with Next.js and Styled Components](https://reesmorris.co.uk/blog/implementing-proper-csp-nextjs-styled-components)

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
