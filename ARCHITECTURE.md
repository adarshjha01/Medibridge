# Architecture

## Overview
MedBridge is a **Next.js 16 App Router** web application for AI-assisted drug interaction triage. A signed-in user uploads a prescription image, optionally records audio symptoms or types symptom text, and the app sends that multimodal payload to a protected server route. The server verifies the Firebase ID token, calls **Google Gemini** with a strict JSON response schema, validates the result with **Zod**, stores the triage result in **Firestore**, and returns structured data back to the UI.

The repo is primarily a TypeScript frontend with one server API route. Runtime responsibilities are split between:
- **App shell and routing** in `src/app`
- **Interactive UI components** in `src/components`
- **Reusable client hooks** in `src/hooks`
- **Schema, Firebase setup, and persistence** in `src/lib`
- **Unit/component tests** in `src/lib/*.test.ts`, `src/components/*.test.tsx`, and `src/test/setup.ts`

## High-level architecture

```text
Browser UI
  ├─ Authentication state via Firebase Auth client SDK
  ├─ Prescription upload via file input / drag-drop
  ├─ Symptom capture via microphone or textarea
  └─ POST /api/analyze-triage with Firebase ID token + FormData

Next.js App Router
  ├─ src/app/page.tsx
  │   ├─ uses useAuth()
  │   ├─ renders Header
  │   └─ renders TriageDashboard for signed-in users
  └─ src/app/api/analyze-triage/route.ts
      ├─ verifies Firebase token with firebase-admin
      ├─ prepares multimodal Gemini input
      ├─ enforces structured JSON output
      ├─ validates response with Zod
      ├─ saves result to Firestore
      └─ returns JSON to client

Firebase
  ├─ Client SDK: auth + firestore in src/lib/firebase.config.ts
  └─ Admin SDK: token verification + secure writes in src/lib/firebase-admin.ts

Persistence
  └─ Firestore collection: users/{uid}/triageHistory
```

## Top-level repository structure

```text
AGENTS.md               Agent-specific repo instructions
CLAUDE.md               Points to AGENTS.md
README.md               Product overview and setup guide
eslint.config.mjs       ESLint configuration for Next.js + TypeScript
next.config.ts          Next.js runtime config
package.json            Scripts and dependencies
postcss.config.mjs      Tailwind PostCSS plugin setup
public/                 Default static SVG assets
src/                    Application source code
tsconfig.json           TypeScript compiler config with @/* alias
vitest.config.ts        Vitest + jsdom + alias setup
```

## Source tree breakdown

```text
src/
  app/
    api/analyze-triage/route.ts   Protected server endpoint for AI analysis + persistence
    globals.css                   Design tokens, utility classes, animations, visual system
    layout.tsx                    Root HTML shell, fonts, metadata, hydration suppression
    page.tsx                      Entry page; auth gate and top-level composition
  components/
    Header.tsx                    Brand/header and auth controls
    TriageDashboard.tsx           Main orchestrator for upload, symptoms, analysis, and history tabs
    TriageHistory.tsx             Loads and renders prior saved triage records
    TriageInputForm.tsx           Prescription upload + symptom capture + submit UI
    TriageResultCard.tsx          Structured result presentation and reset action
    TriageResultCard.test.tsx     Component tests for result rendering states
  hooks/
    useAuth.ts                    Firebase auth subscription + sign-in/sign-out actions
    useAudioRecorder.ts           MediaRecorder lifecycle and microphone state
    useFileHandler.ts             Image validation, preview, drag-drop state
  lib/
    firebase-admin.ts             Server-side Firebase Admin init, auth, firestore
    firebase.config.ts            Client-side Firebase app, auth, firestore, analytics
    firestore.service.ts          Client-side history read helper and legacy save helper
    schema.ts                     Zod schema and shared TriageResult type
    schema.test.ts                Schema validation tests
  test/
    setup.ts                      Testing Library setup
```

## How the application works

### 1. App boot and layout
- `src/app/layout.tsx` defines global metadata, loads `Geist` fonts, imports `globals.css`, and sets `suppressHydrationWarning` on `<body>`.
- `src/app/globals.css` defines the visual language: CSS custom properties, Tailwind v4 theme tokens, glass-card styles, badges, animations, focus rules, and waveform styles.

### 2. Authentication flow
- `src/app/page.tsx` is a **client component** and acts as the route entry point.
- It calls `useAuth()` from `src/hooks/useAuth.ts`.
- `useAuth()` subscribes to Firebase auth state via `onAuthStateChanged(auth, ...)`, tracks `user` and `loading`, and exposes:
  - `signIn()` using `signInWithPopup` and `GoogleAuthProvider`
  - `logOut()` using `signOut`
- `page.tsx` renders three states:
  1. **Loading** while Firebase auth initializes
  2. **Signed out** welcome/login card
  3. **Signed in** dashboard via `<TriageDashboard user={user} />`
- `Header.tsx` also consumes `useAuth()` separately to show live auth controls, avatar, sign-in, and sign-out actions.

### 3. Dashboard orchestration
`src/components/TriageDashboard.tsx` is the central client orchestrator.

It composes:
- `useFileHandler()` for image file state and drag-drop UX
- `useAudioRecorder()` for audio capture state
- `TriageInputForm` for data entry
- `TriageResultCard` for result visualization
- `TriageHistory` for previously saved records

Dashboard state includes:
- `activeTab` (`new` or `history`)
- `symptomsText`
- `isLoading`
- `apiError`
- `triageResult`
- `saveStatus`

It also dynamically imports `TriageResultCard` with `ssr: false`, so the heavy result panel is client-only and shows a loading skeleton while resolving.

### 4. Prescription image handling
`src/hooks/useFileHandler.ts` manages prescription upload behavior:
- accepts one file
- validates that the MIME type starts with `image/`
- rejects files larger than **10 MB**
- creates an object URL for preview
- exposes drag handlers (`onDrag`, `onDrop`)
- exposes `onFileChange()` for file input
- exposes `clearImage()` and revokes the object URL to avoid leaks

`TriageInputForm.tsx` renders this as a drag-and-drop upload card with hidden file input and preview state.

### 5. Symptom capture
`src/components/TriageInputForm.tsx` supports two symptom input modes:
- **Audio mode** using the microphone
- **Text mode** using a textarea

Audio behavior is implemented in `src/hooks/useAudioRecorder.ts`:
- requests microphone access via `navigator.mediaDevices.getUserMedia({ audio: true })`
- records with `MediaRecorder`
- prefers `audio/webm`, falls back to `audio/mp4` for Safari-like compatibility
- accumulates chunks and produces a final `Blob`
- tracks `isRecording`, `recordingTime`, and `audioError`
- stops tracks and intervals during cleanup to avoid dangling resources

### 6. Submission path from browser to server
When the user clicks **Process Triage**, `TriageDashboard.analyzeData()`:
1. Ensures an image exists
2. Creates `FormData`
3. Appends:
   - `image`
   - optional `audio`
   - optional `symptoms_text`
4. Retrieves a Firebase ID token with `user.getIdToken()`
5. Sends a `POST` request to `/api/analyze-triage` with:
   - `Authorization: Bearer <idToken>`
   - multipart `FormData` body
6. Parses the JSON response into shared `TriageResult` shape
7. Updates the UI and save status

### 7. Protected analysis API route
`src/app/api/analyze-triage/route.ts` contains the backend workflow.

#### Step A: Verify the caller
- Reads the `Authorization` header
- Requires a bearer token
- Uses `adminAuth.verifyIdToken(idToken)` from `src/lib/firebase-admin.ts`
- Rejects unauthenticated or invalid tokens with `401`

#### Step B: Validate inputs
- Requires `GEMINI_API_KEY`
- Parses `FormData`
- Requires `image`
- rejects image payloads over **10 MB**
- rejects audio payloads over **5 MB**

#### Step C: Build Gemini input
- Converts uploaded `File` objects into inline base64 payloads using `fileToInlineData()`
- Starts `geminiInput` with the prescription image
- Optionally adds audio file inline data
- Optionally adds plain text symptoms as `Patient Symptoms: ...`

#### Step D: Constrain model output
The route uses `@google/generative-ai` with:
- a system prompt focused on home triage
- a declared JSON `responseSchema`
- `responseMimeType: "application/json"`
- harm category safety thresholds

The expected fields are:
- `symptoms`
- `identified_medications`
- `risk_level`
- `potential_interactions`
- `action_plan`

#### Step E: Validate model output again with Zod
- The raw model text is stripped of accidental markdown fences
- Parsed with `JSON.parse`
- Validated with `TriageResponseSchema.parse(...)`

This second validation layer ensures the server only returns strongly shaped data to the client, even if the model output drifts.

#### Step F: Save to Firestore
On successful validation, the route writes to:
- `users/{uid}/triageHistory`

Stored record shape:
- `result: validated`
- `createdAt: new Date()`

#### Step G: Return result
- Returns the validated JSON body to the frontend
- Handles `ZodError` specially for malformed AI output
- Handles generic failures with `500`

## Data model

### Shared triage result contract
Defined in `src/lib/schema.ts`:

```ts
{
  symptoms: string[];
  identified_medications: string[];
  risk_level: "Low" | "Medium" | "High" | "Critical";
  potential_interactions: string;
  action_plan: string[];
}
```

This schema is important because it is used in three places:
1. As the frontend TypeScript type (`TriageResult`)
2. As runtime validation on the server (Zod)
3. As the logical contract for persisted result payloads in Firestore

## Firebase architecture

### Client Firebase (`src/lib/firebase.config.ts`)
Used in browser code for:
- `getAuth(app)`
- `getFirestore(app)`
- optional Analytics initialization in supported browser environments

It reads public env vars:
- `NEXT_PUBLIC_FIREBASE_API_KEY`
- `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`
- `NEXT_PUBLIC_FIREBASE_PROJECT_ID`
- `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`
- `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- `NEXT_PUBLIC_FIREBASE_APP_ID`
- optional `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID`

### Admin Firebase (`src/lib/firebase-admin.ts`)
Used only on the server for:
- verifying ID tokens
- secure Firestore writes through admin privileges

It reads server env vars:
- `FIREBASE_PROJECT_ID`
- `FIREBASE_CLIENT_EMAIL`
- `FIREBASE_PRIVATE_KEY`

This split is important:
- browser code handles interactive auth/session state
- server code enforces trust boundaries and secure persistence

## History and persistence

There are **two persistence paths** in the repo:

### Actual active path
- `src/app/api/analyze-triage/route.ts` writes records to `users/{uid}/triageHistory`
- `src/components/TriageHistory.tsx` reads history via `getUserTriageHistory()`
- `getUserTriageHistory()` in `src/lib/firestore.service.ts` queries `users/{uid}/triageHistory`

### Secondary / likely older path
- `saveTriageResult()` in `src/lib/firestore.service.ts` writes to `users/{uid}/triage_logs`

That helper is not used by the current API route, which means `triage_logs` and `triageHistory` are inconsistent collection names. The live app behavior currently centers on `triageHistory`.

## Component responsibilities

### `Header.tsx`
- persistent top bar
- brand identity
- system status badge
- auth controls and avatar

### `TriageDashboard.tsx`
- central state container
- API calling logic
- tab switching between new analysis and history
- wires hooks to presentational components

### `TriageInputForm.tsx`
- input-only UI
- upload surface
- audio/text toggle
- record button
- submit button
- inline error presentation

### `TriageResultCard.tsx`
- empty state before analysis
- risk badge rendering
- symptoms list
- identified medications list
- interaction narrative
- ordered action plan
- reset button for a fresh workflow

### `TriageHistory.tsx`
- fetches user-scoped saved records on mount
- renders expandable history cards
- shows date, risk badge, medications, symptoms, and action plan preview

## Runtime request lifecycle

```text
1. User signs in with Google
2. Firebase client SDK resolves auth state
3. User uploads prescription image
4. User records audio symptoms or types symptoms text
5. TriageDashboard builds FormData
6. TriageDashboard fetches Firebase ID token
7. Browser POSTs to /api/analyze-triage
8. API route verifies token with Firebase Admin
9. API route sends multimodal prompt to Gemini
10. Gemini returns structured JSON
11. Zod validates the JSON
12. API route stores result in Firestore under the signed-in user
13. API route returns validated result JSON
14. TriageResultCard renders structured findings
15. History tab later re-reads stored results from Firestore
```

## Testing architecture

### Test setup
- `vitest.config.ts` configures:
  - `jsdom` environment
  - global test APIs
  - alias `@ -> ./src`
  - setup file `./src/test/setup.ts`
- `src/test/setup.ts` imports `@testing-library/jest-dom`

### Existing tests
- `src/lib/schema.test.ts`
  - validates correct schema acceptance
  - verifies all risk enum values
  - verifies malformed payload rejection
  - checks required-field enforcement
- `src/components/TriageResultCard.test.tsx`
  - covers rendered symptoms, medications, warning text, action plan, empty state, and saved badge

## Tooling and build system

### Framework/runtime
- **Next.js 16.2.1** with App Router
- **React 19.2.4**
- **TypeScript 5**

### Styling
- **Tailwind CSS 4** via `@import "tailwindcss"` in `globals.css`
- custom CSS variables and utility classes layered on top

### Validation and AI
- **Zod** for runtime schema enforcement
- **@google/generative-ai** for Gemini integration

### Motion/UI
- **framer-motion** for tab animation and transitions
- **lucide-react** for icons

### Quality
- **ESLint** with Next core-web-vitals and TypeScript presets
- **Vitest** + **React Testing Library** + **jsdom** for tests

### Package scripts
From `package.json`:
- `npm run dev` → start dev server
- `npm run build` → production build
- `npm run start` → run built app
- `npm run lint` → lint source
- `npm test` → run Vitest once

## Environment variables

### Required for AI
- `GEMINI_API_KEY`

### Required for Firebase client
- `NEXT_PUBLIC_FIREBASE_API_KEY`
- `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`
- `NEXT_PUBLIC_FIREBASE_PROJECT_ID`
- `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`
- `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- `NEXT_PUBLIC_FIREBASE_APP_ID`
- optional `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID`

### Required for Firebase Admin server verification
- `FIREBASE_PROJECT_ID`
- `FIREBASE_CLIENT_EMAIL`
- `FIREBASE_PRIVATE_KEY`

## Important implementation notes

1. **The README is partially outdated**.
   - It says Gemini 1.5 Flash, but `route.ts` uses `gemini-2.5-flash-lite`.
   - It mentions `users/{uid}/triage_logs`, but active code stores/retrieves from `triageHistory`.
   - The clone instructions still reference `promptwars-project`, which does not match this repo.

2. **The architecture is intentionally trust-layered**.
   - Client Firebase manages login and browser state.
   - Server Firebase Admin verifies identity and writes securely.
   - Gemini output is constrained by both model schema and Zod validation.

3. **The API route is the main backend boundary**.
   Almost all business logic converges in `src/app/api/analyze-triage/route.ts`.

4. **Most domain state lives in the dashboard component**.
   The hooks are intentionally narrow and reusable, while `TriageDashboard` coordinates the full workflow.

5. **This is a single-app repo, not a monorepo**.
   There are no separate packages, services, or shared workspaces.
