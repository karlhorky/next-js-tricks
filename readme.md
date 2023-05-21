# Next.js Tricks

A collection of useful Next.js tricks

## Avoid PostgreSQL `connection slots` Error with Development Server

TODO: https://github.com/vercel/next.js/discussions/26427#discussioncomment-898067

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
