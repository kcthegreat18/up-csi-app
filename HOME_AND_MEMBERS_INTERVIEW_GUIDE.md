# Home And Members Page Interview Guide

This guide explains the home page and members page from an interview perspective. It focuses on what each page does, how data moves through the SvelteKit app, and the technical decisions you can confidently talk about.

## Quick Summary

The app is a SvelteKit application for UP CSI applicants. The home page acts as the authenticated applicant dashboard. It shows signature sheet progress by committee, constitution quiz progress, and the quiz availability window. The members page supports the signature sheet workflow: applicants browse members by committee, open a modal for a selected member or co-applicant, submit a question and answer, upload a selfie, and save the signature record.

Key technologies involved:

- SvelteKit routing and layouts
- Svelte 5 runes such as `$state`, `$derived`, `$effect`, and `$props`
- Supabase authentication and database reads/writes
- Shared Svelte stores for user, members, applicants, and signature progress
- Google Drive API for storing uploaded signature photos
- Tailwind CSS utility classes and custom CSI color tokens

## Route Map

- `src/routes/+page.svelte`: home page / dashboard
- `src/routes/sigsheet/+page.svelte`: members page shell
- `src/routes/sigsheet/MemberGrid.svelte`: committee filtering, member grid, modal state
- `src/routes/sigsheet/MemberCard.svelte`: individual member card
- `src/routes/sigsheet/Modal.svelte`: signature submission form
- `src/routes/+layout.ts`: client/server universal layout load, Supabase client setup, shared data loading
- `src/routes/+layout.svelte`: shared authenticated layout and store hydration
- `src/routes/+layout.server.ts`: session and role data from server locals
- `src/hooks.server.ts`: Supabase server client and auth guard
- `src/routes/api/upload/+server.js`: photo upload and signature record creation
- `src/routes/api/get_gdrive_folder/+server.js`: Google Drive folder lookup/creation
- `src/lib/shared.ts`: shared writable stores used across pages

## Home Page

### Purpose

The home page has two states:

- If the user is authenticated, it displays the applicant dashboard.
- If the user is not authenticated, it shows a compact welcome/login screen.

The dashboard answers two applicant questions:

- How much of my signature sheet have I completed?
- How much of the constitution quiz have I answered, and is the quiz currently open?

### Main Responsibilities

`src/routes/+page.svelte` is responsible for:

- Reading session and role data passed from layout loading.
- Fetching the current applicant's signature records from Supabase.
- Grouping completed signatures by committee.
- Fetching total member counts by committee to compute progress bars.
- Fetching quiz answers and question count to compute quiz progress.
- Calculating and updating quiz closing/opening text.
- Rendering progress cards and conditional quiz navigation.

### Important State

The home page uses Svelte 5 state primitives:

- `signatureSheet = $state([])`: stores committee progress entries for the signature sheet card.
- `quizProgress = $state('')`: stores quiz completion as a string like `5/20`.
- `quizClosingString = $state('')`: stores human-readable quiz availability text.
- `$effect(...)`: starts an interval that updates the quiz timing string every second.

It also reads shared stores:

- `$uuid`: applicant/user ID.
- `$username`: displayed in the dashboard greeting.

### Signature Sheet Progress Flow

The dashboard loads signature sheet progress in `onMount`.

Flow:

1. Read the applicant ID from `$uuid`.
2. Query the `sigsheet` table for rows matching `applicant_id`.
3. Select `member_id` and the related member committee through the `members:member_id (member_committee)` relationship.
4. Count completed signatures per committee.
5. Query the `members` table to count the total members in each committee.
6. Add a fixed total for `Co-Applicants`.
7. Build a `signatureSheet` array containing the committee name, progress string, and progress bar color.

The progress bar width is computed by `calculatePercentage(progress)`, which parses a `completed/total` string and converts it to a percentage.

Interview talking point:

> I treated the dashboard as an aggregation layer. The raw signature rows are stored per applicant, but the page transforms them into committee-level progress so applicants can quickly see what they still need to complete.

### Constitution Quiz Progress Flow

The quiz progress also loads in `onMount`.

Flow:

1. Query `constiquiz-answers` for the current user.
2. Count answers that are not empty and have either answer text or an option ID.
3. Query `constiquiz-questions` with `count: 'exact'` and `head: true` to get the total number of questions without fetching every row.
4. Set `quizProgress` to a string like `12/30`.
5. Use the same `calculatePercentage` helper for the quiz progress bar.

Interview talking point:

> For the total quiz question count, the page uses Supabase's exact count with a head request. That is more efficient than loading all question rows just to count them.

### Quiz Time Window

The home page defines quiz start and end dates:

- Start: October 27, 2025, 6:00 PM
- End: November 3, 2025, 11:59:59 PM

`updateTimeLeft()` compares the current time to those dates and updates `quizClosingString`.

Cases handled:

- Quiz has not opened yet.
- Quiz has already closed.
- Quiz closes in less than an hour.
- Quiz closes in days and/or hours.

The "Continue" button only renders when the current time is within the start/end window.

Interview talking point:

> The page keeps the quiz route discoverable only during the valid time window. The UI message and the button are both derived from the same start and end dates, which keeps the behavior consistent.

### Authenticated Vs Unauthenticated Rendering

The page checks `data.session`.

If `data.session` exists:

- Render dashboard content.
- Show `Hello, {$username}`.
- Show the user's role if available.

If no session exists:

- Render a welcome panel with the CSI logo.
- Link the user to `/login`.

Note that `hooks.server.ts` redirects unauthenticated users away from non-public routes. The unauthenticated home view is still useful as a fallback and for routes that can be reached before auth state fully resolves.

### UI Structure

The dashboard uses:

- A dark background matching the app theme.
- Two main dashboard cards:
  - Signature Sheet
  - Constitution Quiz
- Committee-specific progress bar colors.
- Responsive layout using a column layout on small screens and side-by-side cards on large screens.

## Members Page

### Purpose

The members page is the applicant-facing signature sheet interface. It lets applicants:

- Browse members by committee.
- See which members they have already submitted signatures for.
- Open a signature modal for unsigned members.
- Submit a question, answer, and image proof.
- Submit signatures for co-applicants through a separate dropdown flow.

### Page Composition

`src/routes/sigsheet/+page.svelte` is intentionally small. It renders a dark page wrapper and delegates the actual UI to `MemberGrid`.

This separation keeps the route file simple and lets `MemberGrid`, `MemberCard`, and `Modal` handle the interactive behavior.

### Shared Data Setup

The members page relies on shared stores from `src/lib/shared.ts`:

- `members`: list of all members loaded from Supabase.
- `filledSigsheet`: `Set` of member IDs already signed by the applicant.
- `applicant_names_list`: names used for the co-applicant dropdown.
- `uuid`: current applicant ID.
- `username`: current applicant display name.
- `gdrive_folder_id`: Google Drive folder for the applicant's uploaded images.

These stores are hydrated in `src/routes/+layout.svelte`.

The layout:

1. Receives data from `+layout.ts`.
2. Sets `uuid`, `username`, `filledSigsheet`, and `gdrive_folder_id` synchronously so child components can read them.
3. Fetches all profile names from `profiles` for the co-applicant dropdown.
4. Fetches all members from `members` for the member grid.

Interview talking point:

> I used shared Svelte stores because the members page and modal need the same user and member data without prop-drilling through every route/component boundary.

### MemberGrid Responsibilities

`MemberGrid.svelte` owns the main page interaction state:

- `activeCategory`: currently selected committee tab/category.
- `selectedMember`: member shown in the modal.
- `showModal`: whether the modal is open.

It also defines:

- `categories`: committee list shown as filter buttons.
- `categoryColors`: color mapping for category indicators and active states.
- `categoryHeaders`: display labels for the page heading.

When a committee is active, the grid filters the shared `members` store:

```svelte
$members.filter(member => member.member_committee === activeCategory)
```

Each member is rendered as a button containing `MemberCard`.

### Filled Signature Behavior

`filledSigsheet` is a `Set`, so checking whether a member is already signed is fast:

```svelte
$filledSigsheet.has(member.member_id)
```

That value is passed to `MemberCard` as `filled`.

Current behavior:

- If a member has not been signed yet, clicking opens the modal.
- If a member has already been signed, clicking does not open the modal.
- Unsigned member photos are grayscale.
- Filled members are shown in color.

Interview talking point:

> A `Set` is a good fit here because the page frequently checks membership by `member_id`. It also makes the UI logic very readable: `filledSigsheet.has(member.member_id)` directly expresses the business rule.

### Committee Filter UI

The committee buttons are generated from the `categories` array. Each button:

- Updates `activeCategory`.
- Displays a colored dot.
- Uses the category color as the active border.
- Has responsive sizing classes.

The Co-App button is a special case. It sets `activeCategory` to `CoApp` and immediately opens the modal, because co-applicant signatures are selected from a dropdown rather than from the member grid.

### MemberCard Responsibilities

`MemberCard.svelte` is a presentational component.

It receives:

- `member`
- `filled`

It derives the member image path from the `photo` field:

```ts
const memberimg = $derived(`/assets/members/${member.photo}.webp`);
```

It renders:

- Member photo
- Member name
- Member role

The photo is grayscale when `filled` is false.

Interview talking point:

> The card component stays intentionally small. It does not know about Supabase, modals, or categories. It just renders the data it receives, which makes it easier to reuse and reason about.

### Modal Responsibilities

`Modal.svelte` handles the signature submission form.

For regular members, the modal displays:

- Member name
- Member role
- Hidden `member_id`
- Hidden `member_name`

For co-applicants, it displays:

- A custom dropdown populated from `$applicant_names_list`
- A hidden `member_name` containing the selected co-applicant

For both flows, the modal collects:

- Question asked by the applicant
- Answer from the member/co-applicant
- Uploaded image proof
- Hidden applicant and Google Drive metadata

### Upload Preview

When the user selects an image, `handleFileChange` checks that the file is an image and creates a temporary object URL:

```ts
imageURL = URL.createObjectURL(file);
```

The modal then previews the selected image before submission.

Interview talking point:

> The image preview is client-side only. It gives immediate feedback without uploading anything until the user submits the full form.

### Submission Flow

`handleSubmit` controls the modal submission.

Flow:

1. Prevent the default form submission.
2. Guard against duplicate submissions with `submitting`.
3. Build a `FormData` object from the form.
4. POST the form to `/api/upload`.
5. If the API returns an error, show an error status message.
6. If the API succeeds:
   - Add the member ID to `filledSigsheet`.
   - Show a success message.
   - Close the modal after a short delay.
7. Always reset `submitting` in `finally`.

Interview talking point:

> I added a `submitting` guard so users cannot accidentally double-submit the same signature while the network request is in flight.

### Backend Upload Flow

`src/routes/api/upload/+server.js` receives the modal form submission.

It:

1. Verifies the user session through `locals.safeGetSession()`.
2. Reads the submitted form fields.
3. Validates the question, answer, and image file.
4. Converts the image file to a Node readable stream.
5. Authenticates with Google Drive using service account credentials.
6. Uploads the image to the applicant's Google Drive folder.
7. Builds a Google Drive file URL.
8. Inserts a signature record into Supabase's `sigsheet` table.
9. Returns JSON success or error responses.

Important security point:

- The server gets the trusted authenticated user from `locals.safeGetSession()`.
- Even though the form includes a hidden `uuid`, the upload endpoint uses `user.id` from the server session for `applicant_id`.

Interview talking point:

> Hidden form fields are useful for passing UI context, but they should not be trusted for authorization. The upload API uses the authenticated server-side user ID as the source of truth.

### Google Drive Folder Flow

`src/routes/api/get_gdrive_folder/+server.js` ensures each applicant has a Google Drive folder for uploads.

Flow:

1. Verify the user session.
2. Look for an existing folder in the `pic-folders` Supabase table.
3. If found, return the existing folder ID.
4. If not found, create a folder in Google Drive under the configured root folder.
5. Save the folder ID in Supabase.
6. Return the folder ID to layout loading.

The resulting folder ID is placed in the `gdrive_folder_id` shared store and sent with future upload form submissions.

## Auth And Layout Data Flow

### Server Hook

`hooks.server.ts` creates the Supabase server client and defines `safeGetSession`.

`safeGetSession`:

- Reads the session from Supabase.
- Verifies the user with `auth.getUser()`.
- Returns `{ session, user }`.
- Treats invalid JWTs as unauthenticated.

The auth guard:

- Loads the user's role from `profiles`.
- Stores `session`, `user`, and `userRole` in `locals`.
- Redirects unauthenticated users to `/login`, except public login routes and API routes.

### Layout Server Load

`+layout.server.ts` returns:

- `session`
- serialized cookies
- `userRole`

### Universal Layout Load

`+layout.ts` creates a Supabase client appropriate for browser or server execution.

It also loads:

- Current user
- Current applicant UUID and username
- Completed signature member IDs
- Google Drive folder ID
- Constitution quiz sections, questions, and answers

Then `+layout.svelte` uses that data to populate shared stores and render the common authenticated navigation shell.

## Likely Interview Questions And Strong Answers

### What did you contribute to the home page?

You can say:

> I worked on the applicant dashboard. It shows personalized progress for the signature sheet and constitution quiz. The page fetches the applicant's signature records from Supabase, groups them by member committee, compares that against total committee counts, and renders progress bars. It also computes quiz completion from saved answers and displays the current quiz time-window status.

### How does the signature progress calculation work?

You can say:

> The page first fetches all sigsheet rows for the current applicant. Each row is joined to the member's committee, then counted by committee. Separately, it fetches all members so it can calculate total members per committee. The UI then maps each committee to a `completed/total` string and converts that to a percentage for the progress bar.

### Why is some data loaded in the layout instead of inside the members page?

You can say:

> The layout loads data that multiple child routes and components need, such as the current user, filled signature IDs, members, applicant names, and the Google Drive folder. Hydrating shared stores in the layout avoids passing the same data through several component layers and keeps the members page focused on interaction logic.

### How does the members page prevent duplicate submissions?

You can say:

> On the client side, already completed members are stored in `filledSigsheet`, which is a `Set`. If the selected member ID already exists in the set, the modal does not open. During submission, the modal also uses a `submitting` boolean to prevent double-click duplicate requests. On the database side, there appears to be a unique constraint for applicant/signatory pairs, and the upload API handles that error for co-applicant duplicates.

### How does image upload work?

You can say:

> The modal submits a `FormData` payload to `/api/upload`. The server validates the authenticated session, reads the image file, converts it into a readable stream, authenticates with the Google Drive API using service account credentials, uploads the file to the applicant's Drive folder, and then saves the resulting file URL plus question and answer data to Supabase.

### Why use a server endpoint for upload instead of calling Google Drive directly from the browser?

You can say:

> The Google Drive service account credentials must stay private. Keeping the Drive API call on the server protects those secrets and lets the server enforce authentication before accepting uploads.

### What is the role of Supabase in these pages?

You can say:

> Supabase handles authentication and database storage. The dashboard reads sigsheet records, member records, quiz answers, and quiz question counts. The members page reads members and applicant names. The upload API writes completed signature records to the `sigsheet` table.

### What is the role of Svelte stores here?

You can say:

> Stores hold cross-route state such as the current user's UUID, username, member list, filled sigsheet IDs, applicant names, and Google Drive folder ID. They are hydrated once in the layout and consumed by the dashboard, member grid, cards, and modal.

### What are Svelte 5 runes doing in these pages?

You can say:

> `$state` is used for local reactive component state, such as modal visibility and active category. `$derived` is used for values derived from props or state, like member image URLs. `$effect` is used for side effects, such as the timer that refreshes the quiz countdown.

### What would you improve if you had more time?

Good answers:

- Move more dashboard aggregation into server-side load functions or database RPCs to reduce client-side queries.
- Add stronger loading and empty states for member lists and progress cards.
- Add error states visible to the user when Supabase requests fail.
- Revoke image preview object URLs to avoid holding browser memory after repeated uploads.
- Make the quiz open and close dates configurable from the database instead of hard-coded in the component.
- Strengthen TypeScript types for Supabase responses and shared stores.
- Add tests for progress calculation and upload validation.

## Code Walkthrough Script

A good walkthrough order in an interview:

1. Start with `hooks.server.ts` to explain authentication and server locals.
2. Move to `+layout.server.ts` and `+layout.ts` to show how session and app data are loaded.
3. Show `+layout.svelte` to explain shared store hydration and the authenticated shell.
4. Open `src/routes/+page.svelte` and explain dashboard progress calculation.
5. Open `src/routes/sigsheet/+page.svelte` to show the route-level wrapper.
6. Open `MemberGrid.svelte` to explain committee filtering and modal state.
7. Open `MemberCard.svelte` to show the small presentational component.
8. Open `Modal.svelte` to explain form submission and image preview.
9. End with `/api/upload` and `/api/get_gdrive_folder` to explain server-side upload and persistence.

## Risks To Be Ready To Discuss

- Some data is fetched on the client after mount, so users may briefly see empty progress values before requests finish.
- The quiz schedule is hard-coded, which means future quiz windows require a code change.
- The home page has repeated Supabase calls that could potentially be consolidated.
- The members store starts empty before the layout fetch completes, so robust loading states would improve resilience.
- The upload form includes hidden fields for context, but the server correctly relies on the authenticated session for the applicant ID.

## Short Interview Pitch

You can summarize your contribution like this:

> I contributed to the applicant-facing dashboard and members/signature-sheet experience. On the home page, I helped build the progress-tracking dashboard by querying Supabase, aggregating signature and quiz progress, and rendering responsive progress cards. On the members page, I worked on the member browsing and signature submission workflow: filtering members by committee, showing completed signatures through shared state, opening a modal for new signatures, previewing uploaded images, and posting the completed signature to a secure server endpoint that uploads to Google Drive and stores the record in Supabase.
