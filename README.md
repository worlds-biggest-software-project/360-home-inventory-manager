# Home Inventory Manager

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source home inventory platform that makes documenting possessions effortless for insurance, moving, downsizing, and estate planning.

Home Inventory Manager is a cataloguing system for household possessions, designed for homeowners, renters, and families who need an accurate record of what they own. It addresses a problem people know they have but consistently defer: when a fire, flood, theft, or estate event occurs, there is no systematic record of items, purchase prices, or serial numbers to support insurance claims or inheritance decisions.

---

## Why Home Inventory Manager?

- Most incumbents are proprietary SaaS (HomeZada, Encircle, SaveOr, Artifcts, Vorby, Sortly) that lock data in closed formats with no open API or portability standard.
- Existing apps tend to focus narrowly on a single use case — insurance documentation, estate provenance, or restoration claims — rather than supporting the full lifecycle (insurance, moving, downsizing, estate planning) in one tool.
- Open-source alternatives (Homebox, Grocy) lack AI photo recognition, insurance export, estate planning workflows, maintenance scheduling, and replacement-value estimation.
- Commercial pricing creates friction for individual homeowners (HomeZada premium ~$99/yr; Sortly $29–$59/mo paid tiers; HomeZada charges per additional property).
- AI computer vision is now capable enough to dramatically lower the activation energy of first-time cataloguing — but is only available in a handful of closed consumer apps.

---

## Key Features

### Item Cataloguing & Capture

- Item records with name, description, category, location (room), purchase date, purchase price, current value, and serial number.
- Multiple photos, receipts, warranties, manuals, and documents per item.
- Barcode and serial number scanning via device camera with manufacturer lookup.
- Search and filter across the full inventory.

### Organisation & Sharing

- Room-, building-, and storage-location hierarchy for navigation and export.
- Multiple home / location support (primary, vacation, storage, rental).
- Family sharing with view/edit permissions and role-based access.
- Cloud backup with offline-first capture and background sync.

### Insurance, Estate & Moving Workflows

- PDF and spreadsheet inventory export structured for insurance claim submission.
- Beneficiary assignment, provenance and sentimental notes, and Personal Property Memorandum (PPM) export for estate planning.
- Moving mode: room-by-room packing lists, box manifests, and arrival checklists generated from the existing inventory.
- Warranty expiration reminders and document storage per item.

### Data Portability & Privacy

- Open-format export (CSV, JSON, PDF) so users are never locked in.
- End-to-end encryption for sensitive financial and personal data.
- Multi-factor authentication and granular sharing controls for insurers, family members, and estate attorneys.

---

## AI-Native Advantage

AI computer vision can identify item type, brand, and category from a photo and pre-populate catalogue fields, turning a tedious manual task into a near-passive one — including a "room scan" mode that proposes entries for every visible item at once. Purchase-confirmation email parsing auto-creates entries, serial-number OCR captures product stickers, and natural-language search lets users ask "find all items bought before 2018 worth more than $500." Live retail-price data feeds replacement and resale value estimates, keeping valuations current rather than frozen at purchase price.

---

## Tech Stack & Deployment

Expected to support both a hosted cloud option and a self-hosted deployment (Docker image with open API), so users with data-sovereignty requirements can run it on their own infrastructure. Mobile clients (iOS, Android) handle room-by-room cataloguing with offline-first sync; a web/desktop interface handles bulk entry, reporting, and family sharing. Open-format export (CSV, JSON, PDF) and a documented REST API are core requirements rather than paid-tier upsells.

---

## Market Context

The home inventory app category is niche but practically important, driven by insurance industry advocacy and rising frequency of natural disasters and property crime. Incumbent pricing ranges from free (NAIC, KnowYourStuff) through freemium consumer apps (HomeZada ~$99/yr premium, SaveOr, Vorby, Artifcts) to professional tiers (Sortly $29–$59/mo, Encircle priced for restoration contractors). Primary buyers are homeowners and renters preparing for insurance or estate events, families managing aging-parent downsizing or inheritance, and professionals (real estate agents, restoration contractors, move managers) serving them.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
