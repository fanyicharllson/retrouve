# Retrouvé

**Vos documents officiels, rendus en sécurité.** / *Your official documents, safely returned.*

Retrouvé is a mobile app that returns lost official documents (CNI, birth certificates, student cards, driving licences, diplomas…) to their owners in Cameroon.

1. A **finder** scans a found document. OCR reads it on the phone and the image is masked before upload.
2. The finder drops it at a nearby **partner point** (shop, cyber café, campus office, commissariat), or keeps it and hands it over in a public place.
3. The **owner**, who declared the loss, is matched automatically and notified by push.
4. The owner proves ownership (photo of the document or questions), pays a small fee by **MTN MoMo / Orange Money** (held in escrow), and collects the document by showing a **pickup QR code**.
5. The finder (and the partner point) are paid once the handover is confirmed.

> Final year project by **Fanyi Charllson** — ICT University, Yaoundé (2026–2027).

## Features (MVP)

- Phone number login (OTP), French / English
- Found-document scan with on-device OCR and masking
- Partner point map, drop-off and handover by QR code
- Loss declarations with automatic matching
- Ownership verification per document type
- MoMo / Orange Money payment held in escrow, automatic reward split
- Custody audit log; unclaimed documents returned to the authority after 90 days
- Admin space: partner approval, disputes, statistics

Full details: [docs/SPECIFICATION.md](docs/SPECIFICATION.md)

## Tech stack

| Part | Technology |
|---|---|
| Mobile | Expo (React Native), TypeScript, Expo Router, NativeWind, React Query, Zustand |
| OCR | ML Kit text recognition (on device) |
| API | Go, chi, sqlc + pgx, goose, river (jobs) |
| Database | PostgreSQL (+ pg_trgm for name matching) |
| API contract | OpenAPI 3 (oapi-codegen / openapi-typescript) |
| Payments | Notch Pay or CinetPay (MTN MoMo, Orange Money) |
| Notifications | Expo push + SMS |
| Hosting | Railway / Render, EAS Build |

## Repository layout

```
retrouve/
├── apps/
│   └── mobile/          # Expo app (finder, owner, partner, admin modes)
├── services/
│   └── api/             # Go REST API
├── api/
│   └── openapi.yaml     # API contract shared by mobile and backend
├── docs/
│   └── SPECIFICATION.md
└── README.md
```

## Getting started

> Setup instructions will be added as each part is built.

Requirements: Node.js LTS, Go (latest stable), PostgreSQL 16+, an Android device or emulator.

## Status

🚧 In development — build phase Nov 2026 → Mar 2027, pilot and testing in spring 2027.

## License

Proprietary — © 2026 Fanyi Charllson. All rights reserved. See [LICENSE](LICENSE).
