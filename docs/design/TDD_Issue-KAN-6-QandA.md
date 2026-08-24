# TDD Discovery Questionnaire — KAN-6: Add a Contact Form to the Website

**Status**: DISCOVERY (Iteration 1) — awaiting stakeholder answers before full TDD is authored
**Issue**: [KAN-6](https://clancymendonca.atlassian.net/browse/KAN-6) — "Add a contact form to the website"
**Repository**: `clancymendonca-bluerose/Test123`
**Design Mode**: `new_application`
**Author**: architect-ai (iteration 1)

---

## 1. Why this is a questionnaire, not a TDD yet

The repository currently contains only a placeholder `README.md` — there is no `package.json`, no `src/` tree, no build config, and no prior TDD, PRD, or architecture doc. The declared tech stack (TypeScript, React SPA, Vite, npm) is a *target* stack, not yet a scaffolded application.

The ticket body is a good functional spec for the form itself:

> Add name, email, and message fields. Validate all required fields. Show a success message after submission. Show an error message if submission fails. Make the form responsive on desktop and mobile.

But it says nothing about **where submitted data goes**, which is the single decision that determines almost everything else in the architecture (hosting model, whether a backend exists at all, security posture, data retention). Answering that wrong in iteration 2 would mean re-architecting rather than extending, so it's being raised here explicitly instead of assumed.

**How to use this doc**: Each question has a recommended default. If no answer is provided before the next iteration, the architect will proceed with the **Recommended Default** and record that choice as an explicit assumption in `TDD.md`.

---

## 2. Section A — Application Foundation

The repo has no scaffold yet. This blocks everything else.

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| A1 | Do we scaffold a new Vite + React + TypeScript project from scratch in this repo, or is there an existing site (built elsewhere) that this form must be embedded into? | Determines whether iteration 2 is "build a site" or "build one component into an existing DOM/CMS." | (a) Scaffold fresh Vite+React+TS SPA; (b) Embed as a widget/script into an existing non-React site (e.g. WordPress, Webflow, static HTML); (c) Add to an existing SPA not yet in this repo | (a) Scaffold fresh Vite+React+TS SPA — matches the declared tech stack and the empty repo state |
| A2 | Is this a single-page contact form site, or does the contact form live inside a larger multi-page site (home, about, contact, etc.)? | Affects routing (React Router vs none) and whether this issue includes building out other pages. | (a) Contact form is the whole app; (b) Multi-page site with routing, contact form is one route/section | (b) Multi-page site with client-side routing (React Router), contact form as `/contact` route — ticket says "add a contact form to the website," implying a site already conceptually exists |
| A3 | Package manager / Node version constraints? | Affects CI config and lockfile choice. | npm (per repo context), pnpm, yarn | npm, Node 20 LTS |

---

## 3. Section B — Form Submission Backend (Critical Path)

This is the highest-risk unknown. A React SPA cannot durably deliver form data anywhere by itself — it needs a destination.

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| B1 | What should happen to a submitted contact form? | Determines whether we need a backend service, a serverless function, or a third-party integration. | (a) Email notification to a business inbox; (b) Stored in a database for later review (admin dashboard); (c) Both; (d) Forwarded to a CRM/ticketing system (e.g., HubSpot, Zendesk) | (a) Email notification only — simplest, matches "small marketing site contact form" scope implied by the ticket; no CRM/dashboard was requested |
| B2 | Do we own/operate backend infrastructure (a server or serverless functions), or should submission go through a third-party form backend (e.g., Formspree, Getform, EmailJS, AWS SES via a lightweight function)? | A pure static SPA with **no backend of our own** cannot send email or persist data without exposing secrets in client code. This is a hard technical constraint, not a preference. | (a) Managed third-party form endpoint (fastest, minimal code, but data passes through a vendor); (b) Self-hosted serverless function (e.g., Vercel/Netlify Function, AWS Lambda) that calls an email provider (SES/SendGrid/Postmark) server-side; (c) Full backend service (Node/Express) with its own deployment | (b) Serverless function + transactional email provider — keeps secrets server-side, no full backend to operate, deploys alongside a static host (Vercel/Netlify) |
| B3 | If a serverless/backend function is used, what hosting platform hosts both the static SPA and the function? | Determines deployment topology and whether Section F (Deployment) needs a platform decision now. | Vercel, Netlify, AWS (S3+CloudFront+Lambda), Cloudflare Pages+Workers, self-hosted Node server | Vercel (or Netlify) — zero-config static+function hosting, minimal ops overhead for a contact form |
| B4 | What email address(es) receive submissions, and who owns that mailbox/DNS for sender verification (SPF/DKIM)? | Needed to configure the email provider and avoid submissions landing in spam. | Provide destination address(es) | *(requires stakeholder answer — no safe default; will be an explicit `[ASSUMPTION NEEDED]` in TDD.md if unanswered)* |
| B5 | Do submitted messages need to be retrievable later (audit trail, "did we get this lead") beyond the notification email? | Determines whether B1(b)/(c) applies and whether a database is in scope. | Yes → add lightweight persistence (e.g., serverless DB like Supabase/DynamoDB); No → email-only, stateless function | No — email-only unless stakeholder confirms a need for a submissions log |

---

## 4. Section C — Validation & UX Behavior

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| C1 | Client-side only validation, or also server-side (defense in depth)? | Client-side validation is UX; it does not stop a malicious or scripted POST directly to the endpoint. | (a) Client-side only; (b) Client-side + server-side re-validation in the function | (b) Both — client-side for instant feedback, server-side re-validation in the function before sending email (never trust client input) |
| C2 | Exact validation rules for each field? | "Validate all required fields" is underspecified — email format, message length limits, name character restrictions all affect the schema. | Specify per field | Name: required, 1–100 chars; Email: required, RFC 5322-reasonable regex or HTML5 `type="email"`; Message: required, 1–2000 chars |
| C3 | How should success/error be surfaced — inline message, toast/snackbar, redirect to a "thank you" page, or modal? | Affects component design and whether routing needs a confirmation route. | Inline message, toast, redirect, modal | Inline message replacing the form (or shown above it), with the form fields cleared on success — no redirect, keeps user in context |
| C4 | Should the form show field-level errors on blur/submit, or only on submit? | UX polish decision but affects component structure now vs. later rework. | On-blur, on-submit only, on-change after first submit attempt | On-submit initially, then on-change for touched fields (standard progressive validation pattern) |

---

## 5. Section D — Security & Abuse Prevention

Contact forms are a standard spam/abuse target; this needs explicit sign-off since the ticket doesn't mention it.

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| D1 | Is bot/spam protection required (CAPTCHA, honeypot field, rate limiting)? | Public contact forms without protection get spammed within days of launch. | (a) Honeypot field only (invisible, no user friction); (b) reCAPTCHA/hCaptcha (adds friction + third-party script); (c) Rate limiting at the function level; (d) Combination | (d) Honeypot field + IP-based rate limiting in the serverless function — no visible CAPTCHA friction unless spam volume proves it's needed |
| D2 | Are there compliance requirements around storing/processing name+email (GDPR, CCPA, etc.) given the target audience/region? | Affects whether consent language or a privacy-policy link is required near the form, and data retention rules if persistence is added (see B5). | Specify applicable regulations/regions | *(requires stakeholder answer — assume no persistence beyond transient email delivery, which minimizes compliance surface, until confirmed otherwise)* |
| D3 | Should the submission endpoint be protected against CSRF, given it's a public unauthenticated form? | Standard for any POST endpoint; contact forms are usually unauthenticated by design, so the real mitigation is origin checking + rate limiting rather than CSRF tokens. | Origin/Referer allow-list on the function, or a CSRF token | Origin allow-list check in the serverless function (reject requests not originating from the site's own domain) plus rate limiting |

---

## 6. Section E — Non-Functional Requirements

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| E1 | "Responsive on desktop and mobile" — any specific breakpoints or design system to match? | Affects CSS approach (custom CSS, CSS Modules, Tailwind, component library). | Specify breakpoints/design system if one exists | Mobile-first, fluid layout with a single breakpoint at 768px (stacked fields on mobile, two-column label/input or full-width on desktop); plain CSS Modules (no framework) since no design system exists yet |
| E2 | Accessibility requirements (WCAG level)? | Contact forms are a common a11y failure point (label association, error announcement). | WCAG 2.1 AA (common baseline), or none specified | WCAG 2.1 AA baseline: labeled inputs, `aria-invalid`/`aria-describedby` for errors, keyboard-navigable, focus management on submit |
| E3 | Browser support matrix? | Affects whether polyfills or older-browser CSS fallbacks are needed. | Specify, or default to modern evergreen | Last 2 versions of Chrome, Firefox, Safari, Edge — no IE11 support (standard for a new Vite app) |
| E4 | Any i18n/localization requirement for form labels and messages? | Affects whether text is hardcoded or routed through an i18n library. | Yes/No, and target locales | No — English only, unless stated otherwise |

---

## 7. Section F — Deployment & Operations

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| F1 | Target hosting platform for the built SPA (and function, per B3)? | Determines `DEPLOYMENT_STRATEGY.md` content and CI/CD pipeline. | Vercel, Netlify, AWS, Cloudflare, other | Vercel (co-locates static hosting + serverless function, generous free tier for a low-traffic contact form) |
| F2 | Is there an existing CI/CD pipeline (GitHub Actions, etc.) this should plug into, or does one need to be created? | Repo has no `.github/workflows` yet. | Create new, or specify existing | Create a minimal GitHub Actions workflow: lint + typecheck + build on PR, deploy on merge to `main` |
| F3 | Do we need monitoring/alerting on the submission function (e.g., alert if email delivery starts failing)? | Silent failures on a contact form mean lost leads with no signal. | Yes (specify tool), No | Yes — minimal: log delivery failures, surface via the hosting platform's function logs; add a paging alert only if this becomes business-critical |

---

## 8. Section G — Testing Strategy

| # | Question | Why it matters | Options | Recommended Default |
|---|----------|-----------------|---------|----------------------|
| G1 | What test coverage is expected — unit tests for validation logic, component tests, end-to-end submission test? | Affects tooling choice (Vitest, React Testing Library, Playwright/Cypress). | Specify | Vitest + React Testing Library for component/validation unit tests; one Playwright E2E happy-path test (fill form → submit → see success message) with the network call mocked |
| G2 | Should the E2E test hit a real (staging) email provider, or mock the submission endpoint? | Real email sends in CI are flaky/costly and can trigger spam filters. | Real vs. mocked | Mocked at the network boundary in CI; a manual smoke test against staging before first production release |

---

## 9. Assumptions the Architect Will Proceed With if Unanswered

If stakeholder input is not provided before iteration 2, the following defaults will be adopted and explicitly labeled as assumptions in `TDD.md` (never silently):

1. **A1/A2**: Fresh Vite + React + TypeScript SPA, multi-page with client-side routing, contact form at `/contact`.
2. **B1/B2/B3**: Email-only delivery via a Vercel serverless function calling a transactional email provider (Postmark or SendGrid); hosted on Vercel.
3. **B5**: No database/persistence — stateless email relay only.
4. **C1/C2**: Client-side + server-side validation; Name (1–100 chars), Email (HTML5 email format), Message (1–2000 chars), all required.
5. **C3**: Inline success/error messaging, no redirect.
6. **D1**: Honeypot field + rate limiting; no visible CAPTCHA at launch.
7. **D2**: No PII persistence beyond transient in-memory processing during email send, minimizing compliance exposure — **must be revisited if B5 changes to "store submissions."**
8. **E1**: Mobile-first CSS Modules, single breakpoint at 768px, no external UI framework.
9. **E2**: WCAG 2.1 AA baseline.
10. **F1/F2**: Vercel hosting; new minimal GitHub Actions CI (lint/typecheck/build).
11. **G1/G2**: Vitest + RTL unit tests, one mocked Playwright E2E happy path.

**Open item requiring a real answer (no safe default exists)**: **B4** — the destination email address(es) for submissions. This must be provided before the email-sending function can be implemented; it will be tracked as a blocking `[ASSUMPTION NEEDED]` in `TDD.md` until answered.

---

## 10. Next Step

Once Section A–G answers (or explicit approval of the Section 9 defaults) are available, iteration 2 will produce the full `TDD.md`, `SYSTEM_ARCHITECTURE.md`, `API_CONTRACTS.md` (the submission endpoint contract), and `SECURITY_DESIGN.md` covering the abuse-prevention approach chosen in Section D.
