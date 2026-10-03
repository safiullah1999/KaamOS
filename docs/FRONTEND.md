# Frontend Subsystem — Context (`frontend/`)

Read `CLAUDE.md` first. The frontend makes the agent's work **legible and controllable**. It renders
decisions; it never makes them. The demo is won or lost here.

---

## 1. Surfaces (four — all live in `frontend/`)
1. **WhatsApp simulator** — phone-like chat; send as a site engineer or supplier; see agent replies.
   Demo's left panel. Posts to `/channels/simulator` via a Next route handler. **The org comes from the
   logged-in session** (the route handler forwards the cookie); the payload carries only the simulated
   sender + content — never an org id.
2. **Owner dashboard** — demo's right panel: requests, RFQs, ranked quotes, policy flags, pending
   approvals, "Needs review" items, weekly report, and an **Ask panel** (copilot `assist` + cited RAG
   answers, e.g. "Why is Supplier B flagged?") — a panel *inside* the dashboard, not a fifth surface.
3. **Approval page** (`/approvals/{token}`) — quote comparison + red policy **reasons** + Approve/Reject;
   one tap → backend signals Temporal. Viewing needs only the token; **Approve/Reject requires being logged
   in as the required role** (not logged in → login → back to the same link).
4. **Audit timeline** — per request, the "why did the agent do this?" trail (from `AuditEvent`s).

> The simulator is a **frontend** surface. The backend `channels/` app holds the adapter, the webhook,
> and the **persona responder**. Out of scope: marketing site, settings sprawl, mobile, supplier portal.

## 2. Tech & auth
- Next.js (App Router) + strict TS + Tailwind. Server components by default.
- **Auth = cookie session + CSRF.** The browser calls **Next.js route handlers (same origin)**, which
  proxy to DRF with the session cookie — **no cross-origin DRF calls, no CORS**. Polling hits the route
  handlers too. Guest owner login for the public demo (backed by local Ollama).
- No heavy state lib; no localStorage for domain data.

## 3. Supplier personas (demo)
Personas **reply automatically** by default — the backend persona responder generates scripted replies
with configurable behavior (fast / slow / overpriced / handwritten-photo) and a small delay, so the demo
is one-click. A **manual mode** toggle lets you type a supplier reply instead. This keeps the 3-minute
video hands-off while still exercising the real extraction path (including the handwritten-quote photo).

## 4. Key screens
**Dashboard (`/`):** `RequestList` (status chips: extracting / RFQ sent / quotes in / awaiting approval /
PO issued / delivered / matched / **stalled**); `RequestDetail` (timeline + `QuoteComparisonTable` +
`PolicyFlags` + `ApprovalPanel` + `NeedsReview` + `AuditTimeline`); `WeeklyReport`; `AskPanel` (P5).
`QuoteComparisonTable` shows per-supplier **delivered total** across lines and a per-line "vs benchmark"
delta (red if >5% over). **Partial quotes** are shown greyed with an "incomplete — not ranked" tag. The
winning row is the one the **backend's deterministic ranking** chose — the UI only highlights it.
`NeedsReview` shows low-confidence extractions (photo beside the extracted table) and uncorrelated supplier
replies, for an owner/manager to confirm or correct. `AskPanel` renders citations (doc title + page) or the
backend's refusal verbatim.
**Approval page:** read-only comparison + explicit reasons ("amount over PKR 500k", "6% over benchmark") +
Approve/Reject + countdown to `expires_at`.
**Simulator:** role toggle (engineer / supplier-A/B/C), text + voice-note + image upload; shows the
**confidence tier** the backend used (rule / classifier / LLM) as a badge.

## 5. Frontend tasks by phase (these are explicit deliverables)
- **Phase 2:** login page, simulator UI + a minimal request view incl. `NeedsReview` (prove intake end-to-end).
- **Phase 4:** owner dashboard (list/detail, quote comparison, policy flags) + approval page (login-gated decide).
- **Phase 5:** audit timeline UI + `AskPanel` (copilot + cited RAG).
- **Phase 6:** weekly report (`/reports/weekly`) + cost dashboard + polish for the split-screen video.

## 6. Data fetching & realtime
Server components for initial loads (org scoping is server-side). Live updates = simple polling (2–3s) on
the active request via a route handler; SSE only if time allows. The frontend never filters by org — the
backend enforces it via RLS.

## 7. Design & conventions
Clean, dense B2B "ops console"; obvious status chips + red flags; money right-aligned, PKR with separators;
split-screen must look good for the video; support Urdu/Roman-Urdu (don't mangle RTL/mixed text). Strict TS,
no `any`; types mirrored from backend serializers; `"use client"` only where needed; format money/dates in
`lib/format.ts`; accessible (keyboard approve/reject). For a distinctive look, consult the `frontend-design`
skill — restrained; clarity over flourish.

## 8. Do-NOT list
No totals/ranking/approval logic in the browser (display backend results) · no domain data in
localStorage/sessionStorage · no screens beyond the four without moving the idea to `docs/NEXT.md` · don't
bypass the signed-link/approval endpoint · don't call DRF cross-origin — go through Next route handlers ·
rely on the session cookie + RLS, never client-supplied org ids.
