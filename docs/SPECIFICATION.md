# Retrouvé — Product & Technical Specification

> Version 0.2 · 29 October 2026 · Status: draft for review
> Author: Fanyi Charllson (ICT University, final year project)

---

## 1. Summary

**Retrouvé** is a mobile app that returns lost official documents (CNI, birth certificates, passports, diplomas, driving licences…) to their owners in Cameroon.

A **finder** photographs a found document. The app reads it (OCR), stores only a **masked** image, and automatically **matches** it with owners who declared it lost. The finder then either drops it at a nearby **partner point** (shop, cyber café, campus office, commissariat — recommended) or keeps it and **hands it over directly** in a public place. The **owner** proves ownership, pays a small recovery fee by **MTN MoMo / Orange Money** (held in escrow), and collects the document by showing a **pickup QR code** that the agent — or the finder — scans. The finder (and the partner point) receive a share of the fee once the handover is confirmed.

**Tagline:** *Vos documents officiels, rendus en sécurité.* / *Your official documents, safely returned.*

---

## 2. Problem

- Losing an official document blocks daily life: concours, bank / MoMo account opening, travel, checkpoints, parcels, job applications.
- Replacement is slow, costly or sometimes nearly impossible:
  - **CNI:** declaration of loss, birth certificate copy, nationality certificate, stamp (reportedly 10,000 FCFA), trips to an enrolment centre.
  - **Birth certificate:** losing the original can require a *jugement supplétif* (court procedure).
  - **Diplomas:** duplicates are rarely issued; often only an attestation.
- Finders who want to return a document have no reliable channel. Documents end up in commissariat drawers, radio announcements, Facebook "pièces perdues" posts — or in the bin (thousands of CNIs found in a Bafoussam dump, March 2026; ~678,000 uncollected cards in commissariats, 2017).
- Existing apps (FindYourCNI, DocMaster, WeFoundIt, LostPieces, Finam Quick…) are mostly **notice boards**: no physical custody, weak privacy (public name search), no payment protection, little institutional supply.

---

## 3. Objectives

### 3.1 General objective
Design and build a secure mobile platform that connects finders and owners of lost official documents through a network of verified partner points, so documents return to their owners quickly and safely.

### 3.2 Specific objectives
1. Let a finder declare a found document in **under 2 minutes**, with OCR pre-filling the fields.
2. Never store or show an unmasked document image.
3. Automatically match found documents with lost declarations and notify owners by push + SMS.
4. Verify ownership with per-document-type questions before any handover.
5. Collect the recovery fee by MoMo / Orange Money and hold it until the handover is confirmed (escrow).
6. Track every document through a logged custody chain (finder → partner point → owner, or → authority if unclaimed).
7. Reward finders and partner points automatically after a successful handover.
8. Give admins the tools to approve partner points, resolve disputes and monitor activity.

---

## 4. Scope

### 4.1 In scope (MVP — built Nov 2026 → Mar 2027)
- One **Expo / React Native** mobile app (Android first, iOS if time allows) with **role modes**.
- A **Go** REST API + PostgreSQL.
- Five fully supported document types (see §6).
- Cities for the pilot: **Yaoundé** (ICTU campus area first).
- French + English UI.

### 4.2 Out of scope (future work)
- Web dashboard for institutions (police, universities, bus agencies).
- WhatsApp bot, USSD / SMS-only search.
- Moto delivery from partner point to owner.
- Other lost items (phones, bags, keys).
- Coverage outside the pilot city.
- Direct integration with DGSN / civil status systems.

### 4.3 Limitations
- OCR accuracy is lower on handwritten documents (birth certificates) → human confirmation is always required.
- The platform cannot verify a document's authenticity, only that the claimant matches it.
- Payment providers' sandbox may differ from production behaviour.
- The pilot depends on a small number of partner points agreeing to take part.

---

## 5. User roles

One account per person, signed in with **Google** or an **email code** (no paid SMS in the MVP). The phone number is collected in the profile for Mobile Money payouts and refunds, but not verified by SMS. There is **no role picker**: every user is both finder and owner; partner agents are approved by an admin. Roles are cumulative.

| Role | Who | Can do |
|---|---|---|
| **User — Finder** | Anyone who finds a document | Scan & declare a found document, see its status, receive a reward to MoMo, build a reputation |
| **User — Owner** | Anyone who lost a document | Declare a loss, pre-register documents, receive match alerts, verify ownership, pay, collect with QR |
| **Partner Agent** | Staff at a registered partner point | Receive documents (scan finder's drop-off QR), bulk-scan documents, see stock, confirm handover (scan owner QR + OTP), see earnings |
| **Admin** | Platform team (the developer in the MVP) | Approve / suspend partner points and agents, resolve disputes and failed verifications, manage document types & fees, view stats, trigger payouts, audit logs |

**Why an admin space?** Someone must approve partner points (otherwise anyone can pretend to be one), handle disputes ("this isn't my document", "partner lost it"), retry failed payouts, and watch for abuse. In the MVP these are **admin screens inside the same mobile app**, visible only to admin accounts. A web dashboard is future work.

---

## 6. Document types

The system is **type-agnostic**: a document type is configuration (fields, matching rule, verification questions, fee, end-of-life destination). Adding a type does not require code changes.

### 6.1 MVP types

| Type | Match keys | Verification questions (examples) | Fee (draft) | Unclaimed → |
|---|---|---|---|---|
| CNI | Doc number (exact) · name + DOB | Last 3 digits of number, place of issue | 1,500 FCFA | Commissariat |
| Birth certificate (acte de naissance) | Name + DOB + place of birth (fuzzy) · acte number | Father's or mother's name, état civil centre | 1,500 FCFA | Mairie / état civil centre |
| Student card | Matricule (exact) · name | Faculty / programme, year | 1,000 FCFA | University administration |
| Driving licence | Licence number (exact) · name + DOB | Categories, place of issue | 1,500 FCFA | Commissariat |
| Diploma / certificate (BAC, GCE, BEPC, Licence…) | Name + DOB + exam year · candidate number | Exam centre / school, session | 2,000 FCFA | Issuing school / office |

### 6.2 Future types
Passport, carte grise, récépissé CNI, nationality certificate, marriage certificate, relevés de notes, health booklet, work / CNPS badges.

**Never stored:** bank card numbers. If a bank card is found, the app only advises the finder to hand it to the bank and the owner to block it.

---

## 7. Core flows

### 7.1 Finder declares a found document
1. Finder taps **"J'ai trouvé un document"**, picks the type.
2. Camera captures the document → **on-device OCR** extracts name, DOB, number.
3. The app **masks** the image on the device (blur photo, number, signature; keeps a recognisable outline). **Only the masked image is uploaded.**
4. Finder confirms / corrects the extracted fields.
5. Finder chooses **how to return it**:
   - **Partner point (recommended):** picks a point on a map → app shows a **drop-off QR code**; status = `DECLARED`. The agent scans the QR and checks the physical document → `IN_CUSTODY`. Declarations not dropped off within **72 h** expire.
   - **Direct handover:** finder keeps the document → status = `WITH_FINDER` (see §7.5).
6. Every code (drop-off, pickup) is stored in the user's account and always available under **My documents**, even after closing the app.

### 7.2 Partner bulk intake
Agent at a commissariat / campus office scans many documents in a row (no finder involved). Each becomes `IN_CUSTODY` directly. Finder share goes to the partner.

### 7.3 Owner declares a loss
1. Owner taps **"J'ai perdu un document"**, picks the type, enters name, DOB and any known number.
2. System runs matching immediately and again each time a new document enters custody.
3. Optional **pre-registration** ("Protect a document" in the **+** menu): owner registers their documents before any loss (number stored **hashed**) → instant exact match later.
4. **My documents** shows each report with a status timeline (Reported → Searching → Possible match → Verified & paid → Ready for pickup → Collected) and tips while waiting (déclaration de perte, idcam.cm pre-enrolment, block SIM / MoMo).
5. Loss reports and found documents are **never public**, not even masked. A search only returns a result when name **and** date of birth both match.

### 7.4 Match → verification → payment → handover
1. Match found → owner gets push + SMS: *"Un document correspondant à votre déclaration a peut-être été retrouvé."* No details are revealed yet.
2. Owner proves ownership in **one** of two ways (3 attempts, then admin review):
   - **Upload a photo, scan or photocopy** of the document from the gallery → read by on-device OCR and compared with the found document (name, DOB, number). The image is deleted after the check.
   - **Answer 2 questions** they know without the card (place / year of issue, last 3 digits, parents' names for a birth certificate…).
3. On success the owner sees where and how to collect, and pays the fee (MoMo / OM). Funds are **held** (escrow).
4. Owner receives a **pickup QR code + 6-digit backup code**, valid 14 days, always available in My documents.
5. At the partner point, the agent scans the QR and checks the owner's face against the document photo, then taps "Handed over" → status = `RETURNED`.
6. Payout split is released (see §9).

> Not in the MVP: in-app selfie / face matching. Only masked images are stored, and reliable face-matching services are costly and sensitive. Phone biometrics (fingerprint, Face ID) cannot prove ownership — they only unlock the phone. The face check happens in person at handover.

### 7.5 Direct handover (finder keeps the document)
1. Owner is verified and has paid → the finder is notified.
2. Finder and owner agree on a **public** meeting place and time through in-app messages; phone numbers stay hidden. A partner point can be used as the meeting place.
3. The owner's app guides them: meet in public → show the code → check the card → tap "I received it".
4. The finder scans the owner's pickup QR (or types the 6-digit code) → the app confirms "verified owner, paid". The finder checks the face and the last 3 digits, hands over and taps "I handed it over".
5. Money is released when **both** confirm, or 24 h after the finder confirms with no problem reported. No handover within 48 h → owner refunded. Either side can report a problem → `DISPUTED`.
6. The finder can switch to a partner point at any time.

### 7.6 Unclaimed documents
After **90 days** in custody with no successful claim → status = `TO_AUTHORITY`. The partner hands it to the destination of its type (commissariat, mairie, school) and confirms in the app → `HANDED_TO_AUTHORITY`.

### 7.7 Not found yet
If no match after 7 days, the owner sees a **replacement guide** for the type (declaration of loss, documents needed, idcam.cm pre-enrolment for CNI). Their declaration stays active for 6 months.

---

## 8. Document lifecycle (state machine)

```
DECLARED ──(drop-off scanned)──▶ IN_CUSTODY ──(owner verified)──▶ CLAIMED
   │                                │  ▲                              │
   │ 72h no drop-off                │  │ payment failed / QR expired  │ payment OK
   ▼                                │  └──────────────────────────────┤
EXPIRED                             │                                 ▼
                                    │                          AWAITING_PICKUP
                                    │ 90 days                         │ QR + OTP scanned
                                    ▼                                 ▼
                              TO_AUTHORITY ──▶ HANDED_TO_AUTHORITY   RETURNED

Any state ──(admin)──▶ DISPUTED ──▶ back to a previous state or CLOSED

Direct handover: DECLARED ──▶ WITH_FINDER ──(owner verified)──▶ CLAIMED ──(paid)──▶ AWAITING_MEETUP
                 ──(finder scans owner QR + both confirm)──▶ RETURNED   (48 h without handover → refund)
```

---

## 9. Payments & fees

- Provider: **Notch Pay** (primary) or **CinetPay**. Collection by MTN MoMo / Orange Money. Payouts (finder / partner) via the provider's transfer API, or manually by admin in the MVP if the API is not available.
- **Escrow:** the fee is collected at claim time and recorded as `HELD`; it is split only after `RETURNED`.
- Refund if the partner cannot hand over within 14 days (lost document, partner closed).
- **Release rule:** the finder's (and partner's) share is credited only after the owner confirms receipt — at a partner point the agent's QR scan + "Handed over" counts as confirmation; in a direct handover the finder taps "I handed it over" **and** the owner taps "I received it". The app reminds the owner automatically.
- **Wallet:** every user has an in-app wallet showing *available* and *on hold* amounts and a history. Withdrawals go to MTN MoMo / Orange Money (minimum 500 FCFA), with the provider's transfer fee shown before confirming.
- **Provider fees:** the payment provider takes a cut on collection (owner pays in) and on payout (withdrawal). The collection fee is taken from the **platform's** share so the finder and partner amounts stay round; the payout fee is shown to the user at withdrawal. Exact rates depend on the chosen provider (Notch Pay / CinetPay).
- **No payment between users:** chat between finder and owner opens only after the owner has paid, and blocks phone numbers and money requests, so payments stay in the app (protecting users and the platform's revenue).
- **Payment timing:** right after successful verification, before pickup. Shown to the owner as "where your money goes".
- **Split** (on a 1,500 FCFA fee), platform share minus provider fees:

| Mode | Finder | Partner point | Retrouvé |
|---|---|---|---|
| Partner point | 40 % (600) | 30 % (450) | 30 % (450) |
| Direct handover | 70 % (1,050) | — | 30 % (450) |
| Bulk intake (no finder) | — | 70 % (1,050) | 30 % (450) |
- The fee is a **service fee** (custody, verification, notification), never a price for the document itself.

---

## 10. Privacy, security & abuse

| Risk | Control |
|---|---|
| Someone harvests personal data | No public listing or name search. Owners only see a match after giving name + DOB, and details only after verification. Search rate-limited per account. |
| Image misuse | Masking on device; unmasked images are never uploaded or stored. |
| Impostor claims a document | Type-specific questions, 3 attempts, face check against the document photo at handover, OTP to the declared phone. |
| Fake partner point | Partner points approved by admin (visit / call, business registration or institution letter). |
| Finder asks for direct payment | App warns: never pay anyone directly; all payments in-app. Reports → admin. |
| Direct handover: unsafe meeting | Public places only, in-app messages with hidden phone numbers, partner point offered as meeting place. |
| Direct handover: finder never shows | Escrow — owner refunded after 48 h; finder reputation lowered. |
| Direct handover: fake confirmation | Money released only when both confirm (or 24 h with no report); disputes go to admin. |
| Partner loses / keeps document | Every custody step is logged with timestamp and agent; disputes and partner rating. |
| Document numbers leaked from DB | Numbers stored as salted hashes for matching; only the last 3 digits kept in clear for display. |
| Account takeover | Google sign-in or email code, short-lived JWT + refresh tokens, device binding for partner agents. |

**Legal basis (to confirm with a lawyer / the administration):**
- Official ID documents remain state property → unclaimed documents always go back to the authority.
- Cameroon personal data protection law (Law n° 2024/017, December 2024 — verify the reference) and cybersecurity law n° 2010/012.
- Pilot partnership with a commissariat and/or ICTU administration.

---

## 11. Matching

For each new `IN_CUSTODY` document and each new loss declaration of the same type:

1. **Exact match** on hashed document number → score 100.
2. Otherwise **fuzzy match**: normalised full name (lowercase, no accents, sorted tokens, trigram similarity via PostgreSQL `pg_trgm`) + exact DOB (+ place of birth for birth certificates).
3. Score ≥ 85 → notify owner. 60–84 → owner sees "possible match", must answer verification questions. < 60 → ignored.
4. One document can match several declarations; the first verified owner wins, the others are notified it was claimed.

---

## 12. Non-functional requirements

- **Performance:** API p95 < 300 ms; matching runs asynchronously (job queue).
- **Offline-first capture:** the finder can scan without network; the upload retries automatically.
- **Low-end devices:** Android 8+, 2 GB RAM; images compressed < 300 KB.
- **Languages:** French and English.
- **Availability:** 99 % during the pilot.
- **Auditability:** every state change logged (who, when, where).
- **Testability:** unit tests on matching, state machine and payment split; end-to-end tests on main flows.

---

## 13. Architecture

```
┌──────────────────────────┐      HTTPS / JSON      ┌───────────────────────────┐
│  Mobile app (Expo / RN)  │ ─────────────────────▶ │  API (Go, chi)            │
│  Finder · Owner ·        │                        │  auth · documents ·       │
│  Partner · Admin modes   │ ◀── push (Expo) ────── │  matching · payments ·    │
│  On-device OCR + masking │                        │  notifications · admin    │
└──────────────────────────┘                        └─────┬─────────┬───────────┘
                                                          │         │
                                     ┌────────────────────┘         └────────────────┐
                                     ▼                                               ▼
                        ┌─────────────────────────┐               ┌──────────────────────────────┐
                        │ PostgreSQL               │               │ External services            │
                        │ data + pg_trgm +         │               │ Notch Pay / CinetPay (pay)   │
                        │ job queue (river)        │               │ SMS provider                 │
                        └─────────────────────────┘               │ Object storage (masked imgs) │
                                                                   └──────────────────────────────┘
```

### 13.1 Mobile
- Expo SDK (latest stable), Expo Router, TypeScript.
- NativeWind (Tailwind) for styling, React Query for server state, Zustand for local state.
- OCR: **ML Kit text recognition on device** (requires an Expo development build) — free, works offline, nothing sensitive leaves the phone.
- Camera: `react-native-vision-camera` or `expo-camera`.
- Maps: `react-native-maps` (partner points).
- Push: Expo Notifications. i18n: `i18next`.

### 13.2 Backend
- **Go** (latest stable), router **chi**.
- **PostgreSQL** with `pgx` + **sqlc** (type-safe queries), **goose** migrations.
- Jobs: **river** (Postgres-backed queue) for matching, SMS, payment webhooks, expiries.
- API contract: **OpenAPI 3** spec → `oapi-codegen` (Go) and `openapi-typescript` (mobile types).
- Auth: Google ID token or email one-time code → JWT access + refresh tokens. Email via a free-tier provider (e.g. Resend); SMS OTP is future work when funded.
- Notifications: Expo push (free) in the MVP; SMS later.
- Storage: S3-compatible bucket (Cloudflare R2 or similar) for masked images.

### 13.3 Hosting
- API + DB: Railway or Render (or a small VPS).
- Mobile builds: EAS Build; distribution via Play Store internal testing during the pilot.

---

## 14. Data model (entities — full schema in the design phase)

- `users` — phone, name, language, roles, MoMo number for payouts.
- `partner_points` — name, type (shop, cyber, campus, commissariat), location, hours, status.
- `partner_agents` — user ↔ partner point.
- `document_types` — code, labels, field schema, match rule, questions, fee, end-of-life destination.
- `found_documents` — type, extracted fields (JSONB), number hash, masked image URL, finder, partner point, status.
- `loss_declarations` — owner, type, name, DOB, number hash, status.
- `registered_documents` — pre-registered documents of an owner.
- `matches` — found document ↔ declaration, score, status.
- `claims` — verification attempts, result, handover QR / OTP, expiry.
- `payments` — claim, amount, provider ref, status (`PENDING`, `HELD`, `RELEASED`, `REFUNDED`).
- `payouts` — beneficiary, amount, status.
- `wallets` — user, available balance, on-hold balance.
- `wallet_transactions` — wallet, type (reward, withdrawal, refund), amount, status (`ON_HOLD`, `AVAILABLE`, `PAID_OUT`), related claim.
- `messages` — handover, sender, text, created at (only after payment; filtered).
- `problem_reports` — user, kind, related document, details, screenshot, status.
- `partner_applications` — user, business name, type, location, hours, documents, status.
- `custody_events` — document, from-state, to-state, actor, partner point, timestamp (audit log).
- `notifications`, `disputes`, `ratings`.

---

## 15. API outline (v1)

| Area | Endpoints (examples) |
|---|---|
| Auth | `POST /auth/google` · `POST /auth/email/request` · `POST /auth/email/verify` · `POST /auth/refresh` |
| Direct handover | `GET/POST /handovers/{id}/messages` · `POST /handovers/{id}/scan` · `POST /handovers/{id}/confirm` · `POST /handovers/{id}/report` |
| Document types | `GET /document-types` |
| Found | `POST /found-documents` · `GET /me/found-documents` · `POST /found-documents/{id}/image` |
| Losses | `POST /loss-declarations` · `GET /me/loss-declarations` · `POST /registered-documents` |
| Matches & claims | `GET /me/matches` · `POST /matches/{id}/verify` · `POST /claims/{id}/pay` |
| Payments | `POST /webhooks/payments/{provider}` |
| Wallet | `GET /me/wallet` · `GET /me/wallet/transactions` · `POST /me/wallet/withdrawals` |
| Support | `POST /problem-reports` · `POST /partner-applications` · `GET /app/version` |
| Partner | `GET /partner-points?near=` · `POST /partner/intake` (scan drop-off QR) · `POST /partner/bulk-intake` · `GET /partner/stock` · `POST /partner/handover` · `POST /partner/to-authority` |
| Admin | `GET/PATCH /admin/partner-points` · `GET/PATCH /admin/disputes` · `GET /admin/stats` · `GET /admin/audit` · `POST /admin/payouts/{id}/retry` |

---

## 16. MVP feature checklist

**Must have**
- [ ] Google / email-code sign-in, profile setup, FR / EN (switchable from onboarding)
- [ ] Onboarding slides + splash screen
- [ ] **+** menu: I found · I lost · Protect a document
- [ ] Declare found document with OCR + on-device masking
- [ ] Return choice: partner point (drop-off QR) or direct handover
- [ ] Partner point map + drop-off QR
- [ ] Direct handover: in-app messages, finder scans owner QR, double confirmation
- [ ] My documents: status timeline, waiting tips, codes always available
- [ ] Privacy-first search (name + date of birth)
- [ ] Wallet: available / on hold, history, withdraw to MoMo / OM
- [ ] Alerts tab, Profile tab (version + check for updates, sign-out confirmation)
- [ ] Help & how it works (in-app browser to the website), report a problem, become a partner point
- [ ] Toast messages for success, waiting, offline and errors
- [ ] Partner: intake scan, bulk intake, stock, handover (QR + OTP)
- [ ] Declare loss + automatic matching + push / SMS notification
- [ ] Ownership verification: document photo upload **or** questions
- [ ] MoMo / OM payment with escrow + payout split
- [ ] Custody audit log
- [ ] Admin: approve partners, disputes, stats
- [ ] 90-day unclaimed → authority flow

**Should have**
- [ ] Pre-registration of documents
- [ ] Replacement guide when nothing is found
- [ ] Finder reputation / badges
- [ ] Offline capture queue

**Future**
- Web dashboard for institutions · WhatsApp bot · USSD / SMS search · "If found, scan me" QR stickers · moto delivery · more document types and cities.

---

## 17. Timeline

| Period | Goal |
|---|---|
| **Nov 2026** | UI design, OpenAPI spec, DB schema, Go API skeleton (auth, document types), Expo app skeleton + navigation |
| **Dec 2026** | Found-document flow (camera, OCR, masking, upload), partner points, partner intake |
| **Jan 2027** | Loss declarations, matching engine, notifications, verification |
| **Feb 2027** | Payments (sandbox → live), escrow + payouts, handover QR / OTP, admin screens |
| **Mar 2027** | Unclaimed flow, polish, security review, tests, first pilot partner points |
| **Spring 2027** | Field testing / pilot, bug fixes, report, defense preparation |

---

## 18. Risks

| Risk | Mitigation |
|---|---|
| Partners refuse to join | Start with ICTU administration + 2–3 shops near campus; show the revenue share |
| Payment API access delayed (KYC) | Apply early; mock provider in development; manual payout fallback |
| OCR poor on some documents | Human confirmation; manual entry fallback |
| Legal objection to handling ID documents | Pilot with a commissariat; unclaimed → authority; fee framed as service fee |
| Low supply of found documents | Bulk intake at institutions is the main supply, not random finders |
| Scope creep | Five document types, one city, mobile only |

---

## 19. Open questions

1. Final fee per type: flat 1,500 FCFA, or tiered (student card 1,000 · CNI / licence / birth certificate 2,000 · passport / diploma 3,000).2. Notch Pay vs CinetPay: payout API availability and KYC requirements for a student project.
3. SMS provider for Cameroon (cost per SMS, sender ID) — only once funded; the MVP uses Google / email sign-in and push notifications.
4. Which partner points agree to the pilot?
5. Is a face check by the agent enough, or should the owner also show another ID?
6. If an owner never confirms receipt in a direct handover, should the finder's reward auto-release after 72 h without a reported problem?
