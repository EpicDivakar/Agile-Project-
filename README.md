# Student Setu

An academic assistant for Indian college students: live attendance with early
warnings, a handbook-grounded chatbot (Setu AI), a searchable notices feed, a
"Where do I go?" office guide, and a faculty flow for marking attendance and
spotting likely defaulters before the exam form deadline.

Mobile-first React app with a small Node API. The visual language is the Toran
design system (tokens, type and components recreated in `src/styles.css` and
`src/ui.tsx`).

## Run it

Needs Node 22 or newer.

```bash
npm install
npm run dev
```

Open http://localhost:5173. The API runs on port 8787 and Vite proxies `/api` to it.

Demo accounts, all with the password `setu123`:

| Role | Sign in with | What you see |
| --- | --- | --- |
| Student | `21CS001` | Aditi Sharma, below 75% in Database Systems |
| Student | `21CS011` | Kavya Menon, above 94% in every subject |
| Student | `21CS006` | Farhan Khan, below 65% in Computer Networks after 4 straight absences; flagged by two teachers |
| Student | `21CS009` | Ishita Banerjee, above 80% today but missing half her recent classes; the risk model flags her |
| Student | `21CS020` | Uday Yadav, short early in the semester, has attended every class since |
| Student | `21CS023` | Zoya Ansari, right on 75% with no classes to spare |
| Faculty | `FAC102` | Prof. Mehta, who teaches Database Systems |
| Faculty (HOD) | `FAC101` | Dr. Nair, who sees every class in the department |
| Parent | `sunil.sharma@example.com` | Aditi's parent |
| Parent | `salma.khan@example.com` | Farhan's parent |
| Parent | `ramesh.yadav@example.com` | Uday's parent |
| Admin | `ADM001` | Meenakshi Iyer, Registrar's Office: the ERP admin panel |

## ERP modules

| Module | Students & parents | Faculty | Admin |
| --- | --- | --- | --- |
| Timetable | Weekly classes and labs, current class highlighted | Teaching week | — |
| Assignments | Submit work, see marks and feedback | Post, see who's missing, grade | — |
| Results | Internal/external marks, grades, SGPA, CGPA | Marks entry (`/marks`) | — |
| Exam schedule | Dates, halls, eligibility from attendance | Their papers | — |
| Fees | Invoices, late fees (₹50/day, max ₹1,500), demo online payment, receipts | — | Dues and defaulters, record cash/DD payments, raise fees |
| Library | Books issued, fines (₹2/day), renew, catalogue | Same | Issue, return, add books |
| Hostel | Room, warden, mess timings, complaints | — | Update and resolve complaints |
| Transport | Routes, bus pass, apply (adds the fee) | Routes | Routes and passes |
| Leave | Apply and withdraw medical/duty/personal leave | Approve or reject | Approve or reject |
| People & subjects | — | — | Add/remove accounts, reset passwords, subjects and teachers, post notices |

The server code is in `server/erp.ts` (routes and calculations) and `server/erp-seed.ts`
(demo data). An existing `server/data/db.json` gets the ERP collections added on the
next start; nothing else in it changes. Payments are simulated: no money moves.

The login screen lists these accounts; tap one to fill it in. The test students
are defined in `PERSONAS` in `server/seed.ts`; the other 18 classmates get
random attendance.

Setu AI chat history and saved answers are seeded too (`CHATS` in
`server/seed.ts`) for `21CS001`, `21CS006`, `21CS011`, `21CS020`,
`sunil.sharma@example.com` and `FAC102`. They include both AI answers and one
offline handbook answer.

Other commands:

- `npm run build` then `npm start` serves the built app and the API together on port 8787.
- `npm run reset` deletes the database; restart the server to seed it afresh.
- `npm run typecheck` checks types.

## Setu AI

To turn on the AI, set `NVIDIA_API_KEY` in `.env` to a key from https://build.nvidia.com
(it starts with `nvapi-`). The server reads it on the next question, so no restart is needed.
Each question is answered by NVIDIA's API with `NVIDIA_MODEL`
(default `nvidia/nemotron-3.5-super-vl-120b-a12b`) through the OpenAI-compatible
endpoint (`NVIDIA_BASE_URL`, default `https://integrate.api.nvidia.com/v1`). Any
`<think>` reasoning the model streams is hidden. The model gets two things:

- The whole knowledge base: every handbook section, office and notice, including
  attachments such as the exam timetable (`knowledgeBase()` in `server/knowledge.ts`).
  It sits in the system prompt.
- A live record of the signed-in user (`contextFor()` in `server/index.ts`). For a
  student or parent it holds per-subject attendance, classes needed or still
  missable, the last 10 classes, absence dates, the projection and risk score,
  teacher messages, tasks, notice read status and the exam countdown. For faculty
  it holds every student in their classes (the whole department for the HOD) with
  risk details, and the flags sent. The record mirrors what the app shows that
  user, so a student never gets another student's data.

Without a key the chat still works offline. Personal questions (attendance, a
subject, tasks, teacher messages, exams, and for faculty the at-risk list) are
answered from the live data. Anything else quotes the closest handbook section and
office entry. Offline answers are labelled "AI is offline".

The handbook in `server/knowledge/handbook.md` is sample text. Replace it, or
add more `.md` files to that folder, with your college's real rules; every `##`
section is indexed as one passage.

## How the pieces fit

```
shared/types.ts      Types used by both the API and the app
server/
  index.ts           REST API, auth, chat streaming
  db.ts              JSON-file database (server/data/db.json), seeded on first run
  seed.ts            Demo college: 24 students (6 scripted test cases), 4 subjects, about 10 weeks of classes
  risk.ts            Attendance maths and the defaulter risk score
  knowledge.ts       BM25 retrieval over handbook, offices and notices
  chat.ts            NVIDIA API call (streaming), with the offline fallback
  erp.ts             ERP routes: academics, fees, library, hostel, transport, leave, admin
  erp-seed.ts        ERP demo data, added to any database that lacks it
src/
  ui.tsx             Toran components: AppBar, BottomNav, Card, Badge, Banner, Tabs, Sheet...
  screens/           Login, Student (Home, Attendance), Chat, Notices, Routing, More, Faculty
```

### Attendance and the risk score

Attendance is computed from class sessions, never stored as a percentage. For
each student and subject, `server/risk.ts` works out the current percentage,
how many classes in a row would restore 75%, how many can still be missed, and
a projected end-of-semester percentage at the student's recent pace.

The defaulter score is a logistic function of four signals: that projection,
the current shortfall, consecutive absences, and how far recent attendance has
dropped below the student's own average. The weights are set by hand. To make
it a trained model, fit those weights on past semesters' attendance and final
defaulter lists; the inputs and the screen stay the same.

## What is and isn't real yet

- Sign-in uses seeded accounts with hashed passwords and server-side sessions.
  There is no registration, password reset or college SSO.
- Data lives in one JSON file, which suits a demo and a single server. Move to
  Postgres or similar before real use.
- Alerts appear inside the app (banners for students and parents, messages from
  faculty). Nothing is sent by SMS, email or push; the notification switches in
  Profile only store the preference.
- "Directions" shows written walking directions from the office directory, not a map.
- Notices and the office directory are seeded; there is no admin screen to edit them.
