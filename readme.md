# Next.js Tricks

A collection of useful Next.js tricks

## Avoid PostgreSQL `connection slots` Error with Development Server

The Next.js dev server uses Fast Refresh to update the app in the browser when app code is changed, re-running the code to achieve this. If the code has stateful operations in it, including creation of a connection to a database (eg. PostgreSQL, MongoDB), then these stateful operations will be re-run.

In the case of code which connects to a database, what this means is that every time the code is changed (or even every time the file is saved without changes), new database connections will be created and the old connections will not be closed:

```ts
// util/database.ts

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

```ts
// util/database.ts

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

## Content Security Policy (CSP)

- TODO: CSP example for pages/ dir with middleware and reading x-nonce header in pages/\_document.tsx
- TODO: CSP example for App Router with middleware and reading x-nonce header in app/page.tsx

## Type Check Responses from Route Handlers

To add TypeScript type checking of responses in your [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/router-handlers):

1. Upgrade to at least [`next@13.4.4-canary.0`](https://github.com/vercel/next.js/releases/tag/v13.4.4-canary.0) ([PR](https://github.com/vercel/next.js/pull/47526))
2. Set a return value using `NextResponse<ResponseBody>`, where `ResponseBody` is your type for the JSON response body (optionally also wrap `NextResponse` in `Promise` for `async` Route Handler functions):

   ```ts
   // app/api/animals/[animalId]/route.ts

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
