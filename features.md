# Home Inventory Manager — Feature & Functionality Survey

> Candidate #360 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| HomeZada | Web + mobile | Commercial (freemium; ~$99/yr premium) | https://www.homezada.com |
| Encircle | Web + mobile | Commercial (subscription; targets restoration pros) | https://www.getencircle.com |
| SaveOr | iOS + web | Commercial (freemium) | https://www.saveor.com |
| Artifcts | Web + mobile | Commercial (freemium) | https://artifcts.com |
| Vorby | iOS + Android | Commercial (freemium) | https://vorby.com |
| Sortly | iOS + Android + web | Commercial (free tier; $29–$59/mo paid) | https://www.sortly.com |
| Homebox | Web (self-hosted) | Open source (MIT) | https://github.com/hay-kot/homebox |
| Grocy | Web (self-hosted) | Open source (MIT) | https://grocy.info |
| NAIC Home Inventory | iOS + Android | Free (non-profit / government) | https://content.naic.org/consumer/home-inventory |
| KnowYourStuff (III) | iOS + Android | Free (non-profit) | https://www.knowyourstuff.org |

---

## Feature Analysis by Solution

### HomeZada

**Core features**
- Room-by-room home inventory cataloguing with photo and document attachments
- AI-assisted item identification from photos, reducing manual data entry
- Major Assets Forecasting: AI-estimated replacement years and costs for large home systems (HVAC, roofing, appliances)
- Maintenance scheduling with automated reminders
- Home improvement project tracking and budget management
- Financial dashboard for home-related expenses
- PDF and spreadsheet export; ZIP backup of all photos and documents
- Move feature: transfers inventory and documents between properties
- Supports multiple properties (additional charge per property)

**Differentiating features**
- Full home management platform — inventory is one module inside a broader home lifecycle system
- Predictive replacement cost forecasting for major assets (AI-powered)
- Real estate professional tier that lets agents/builders manage client home data

**UX patterns**
- Onboarding via property setup wizard (address, property type, rooms)
- Dashboard surfaces upcoming maintenance tasks and cost forecasts
- Progressive disclosure: basic free tier, advanced features behind paywall

**Integration points**
- None publicly documented as open API; primarily a closed SaaS

**Known gaps**
- Limited estate planning / beneficiary assignment workflows
- No barcode scanning for item lookup noted in current reviews
- Professional tier adds cost friction for individual homeowners

**Licence / IP notes**
- Proprietary SaaS; data portability via export but no open API

---

### Encircle

**Core features**
- Photo and video capture linked to rooms and items
- Floor plan creation with drag-and-drop
- Contents inventory with categories, quantities, and values
- Moisture data and drying log tracking (restoration-specific)
- E-signature collection for field approval
- Detailed report generation (schedules of loss, photo reports, field documentation)
- Zapier integration for connecting to thousands of third-party apps
- Open REST API for custom integrations

**Differentiating features**
- Purpose-built for insurance restoration professionals, not homeowners
- Extremely detailed loss documentation (moisture readings, drying logs)
- Integrates with Xactimate (industry-standard estimating software) and Matterport (3D scanning)
- Strong multi-party workflow: field tech captures, office reviews, insurer receives

**UX patterns**
- Field-first design: optimised for rapid mobile capture in damaged properties
- Job-centric workflow rather than home-centric
- Structured report templates drive consistency

**Integration points**
- Open REST API
- Zapier connector
- Native integrations: Xactimate, Matterport, Albi, AnswerForce, iRestore, Job Dox, VCA Software, PSA, RealWork, Xcelerate

**Known gaps**
- Not designed for ongoing home inventory maintenance — built for claims events
- No estate planning, moving, or downsizing workflows
- Pricing targets commercial restoration companies, not individual homeowners

**Licence / IP notes**
- Proprietary SaaS; open REST API available

---

### SaveOr

**Core features**
- Item cataloguing with photos, videos, voice notes, and documents
- Room and location organisation
- Beneficiary assignment per item (estate planning)
- Collaborative family access: invite heirs to view, comment, or help decide disposition
- Personal Property Memorandum (PPM) PDF export for legal estate documents
- Move Manager Report export for professional move managers
- AI photo recognition to pre-populate item details
- Insurance documentation export

**Differentiating features**
- Explicitly designed for estate planning, downsizing, moving, and insurance (four use cases in one)
- Captures provenance, stories, and sentimental context alongside financial value — goes beyond pure documentation
- PPM export is a legally recognised estate planning format
- Invite-and-collaborate model for family members and estate attorneys

**UX patterns**
- Use-case-first onboarding: user selects primary goal (moving, estate, insurance, downsizing)
- Item storytelling interface encourages narrative alongside data
- Family sharing prominently featured in navigation

**Integration points**
- No publicly documented external API; primarily a closed consumer app

**Known gaps**
- iOS-centric (primary App Store presence)
- Limited barcode scanning
- No valuation / replacement cost estimation
- No maintenance scheduling

**Licence / IP notes**
- Proprietary SaaS consumer app; no open API

---

### Artifcts

**Core features**
- Item documentation with photos, descriptions, and provenance notes
- Sentimental and monetary value fields
- Story capture for family heirlooms and significant objects
- Sharing items with family members or estate attorneys
- Digital archive of family objects and their histories

**Differentiating features**
- Strongest focus on provenance, legacy, and sentimental value — closest to a digital family archive
- Designed to complement estate planning documents
- Emphasises item "stories" as a first-class feature

**UX patterns**
- Object-first rather than room-first navigation
- Story and photo capture flow drives engagement
- Positioned as a family legacy platform rather than an insurance tool

**Integration points**
- No publicly documented API

**Known gaps**
- No insurance export or loss documentation
- No barcode scanning or serial number tracking
- No replacement value estimation
- No maintenance scheduling
- Limited room/location organisation

**Licence / IP notes**
- Proprietary SaaS; no open API

---

### Vorby

**Core features**
- AI image recognition to identify household items from photos
- Forward purchase confirmation emails to auto-populate inventory
- Barcode scanning with automatic product detail lookup
- Natural language search ("Where are my headphones?")
- Voice assistant integration: Alexa, Siri, Google Assistant
- Warranty tracking with expiration reminders
- Receipt and manual storage per item
- End-to-end encryption (client-side encryption before upload)
- Multiple home support (primary, vacation, storage, rental)
- Family/tenant sharing with granular permissions
- PDF and CSV export

**Differentiating features**
- Most AI-native home inventory app currently available: photo recognition + email parsing + barcode lookup
- Voice assistant integration is unique in the category
- Client-side end-to-end encryption differentiates on privacy
- Natural language search across the inventory

**UX patterns**
- Multiple frictionless capture methods (photo, email, barcode)
- Progressive feature discovery from free to paid
- Privacy-first messaging as a core differentiator

**Integration points**
- Voice: Alexa, Siri, Google Assistant
- Email parsing (purchase confirmation forwarding)
- No documented public API

**Known gaps**
- No estate planning workflows or beneficiary assignment
- No maintenance scheduling
- No floor plan or room visualisation
- Relatively new product; long-term reliability unproven

**Licence / IP notes**
- Proprietary SaaS; no open API

---

### Sortly

**Core features**
- Visual inventory with high-resolution photo uploads
- Barcode and QR code scanning (including custom QR label generation)
- Custom fields for detailed item attributes
- Hierarchical folder structure for organisation (location → sub-location → item)
- Low-stock and date-based alerts
- Cloud sync across devices with offline support
- Team/multi-user access with role-based permissions
- CSV import/export; reports and bill of materials

**Differentiating features**
- Business/team-oriented design with multi-user workflows and role permissions
- Custom QR code label generation and printing
- Strong serial number and tag tracking for electronics and equipment
- Bill of Materials report generation

**UX patterns**
- Business inventory UX adapted for personal use
- Folder-based hierarchy feels familiar to power users
- Free tier limited to 100 items; upgrades prompt at limit

**Integration points**
- Zapier integration
- REST API (business tiers)
- CSV import from other systems

**Known gaps**
- Not designed for home-specific use cases (estate planning, insurance documentation, moving)
- No photo recognition AI
- No maintenance scheduling or warranty alerts
- Overkill complexity for casual homeowners

**Licence / IP notes**
- Proprietary SaaS; API available on paid tiers

---

### Homebox (open source)

**Core features**
- Item cataloguing with photos, descriptions, serial numbers, and custom fields
- Location and label organisation (hierarchical tree view)
- QR code label generator per item
- Bill of materials report
- Multi-tenant group support (family members share one instance)
- Search across all items
- SQLite database, embedded web UI
- REST API (Go backend)
- Docker/container deployment

**Differentiating features**
- Fully self-hosted, zero data shared with third parties
- MIT licence — freely forkable and modifiable
- Extremely lightweight (~50MB idle memory for the full container)
- Active community fork at sysadminsmedia/homebox continuing development

**UX patterns**
- Minimal, functional UI prioritising data capture over storytelling
- Designed for tech-comfortable homeowners willing to self-host
- Family sharing via group model with single shared instance

**Integration points**
- REST API
- Home Assistant integration possible via API
- Docker Hub image for easy deployment

**Known gaps**
- No AI photo recognition
- No insurance export or estate planning workflows
- No maintenance scheduling
- No mobile-native apps (web-only, mobile-responsive)
- No warranty tracking or expiration reminders
- No valuation or replacement cost estimation

**Licence / IP notes**
- MIT licence — fully open source; original repository: https://github.com/hay-kot/homebox; active fork: https://github.com/sysadminsmedia/homebox

---

### Grocy (open source)

**Core features**
- Grocery and consumable inventory with stock tracking and expiration dates
- Shopping list generation based on minimum stock levels
- Recipe management with ingredient availability checks
- Meal planning calendar
- Household task management and scheduling
- Equipment/asset management section (non-consumable items)
- Battery tracking for household devices
- Barcode scanning via browser camera
- Comprehensive REST API with Swagger UI
- Home Assistant integration

**Differentiating features**
- Broadest scope of any self-hosted option — covers consumables, assets, recipes, tasks, and batteries
- Highly active community with regular releases
- REST API is unusually well-documented for an open-source project
- Companion Android/iOS apps available (Grocy Android, Grocy Mobile)

**UX patterns**
- Power-user dashboard; steep learning curve for newcomers
- Strong community documentation and tutorials
- Modular — users can ignore sections they don't need

**Integration points**
- Full REST API with Swagger documentation
- Home Assistant integration (native community integration)
- Barcode scanners (USB and camera-based)
- Community plugins and companion apps

**Known gaps**
- No AI capabilities
- Equipment section is basic compared to dedicated home inventory apps
- No insurance export, estate planning, or moving workflows
- Primarily consumable-focused; asset tracking is secondary
- Requires self-hosting technical capability

**Licence / IP notes**
- MIT licence — fully open source; https://github.com/grocy/grocy

---

### NAIC Home Inventory App

**Core features**
- Item cataloguing by category
- Barcode scanning for accurate item identification
- Photo upload and export
- Disaster preparedness advice
- Insurance claim filing guidance
- Free; no in-app purchases

**Differentiating features**
- Official product of the National Association of Insurance Commissioners — carries regulatory credibility
- Disaster preparedness information embedded alongside inventory

**UX patterns**
- Simple, category-driven entry
- Designed for non-technical users; minimal friction

**Integration points**
- None documented

**Known gaps**
- Minimal features compared to commercial apps
- No cloud backup (user is responsible for data)
- No AI, valuation, estate planning, or maintenance features
- Development may be slow given non-profit governance

**Licence / IP notes**
- Free, government/non-profit product; not open source

---

### KnowYourStuff (Insurance Information Institute)

**Core features**
- Photo capture of belongings
- Brief item annotation
- Export of belongings list for insurance claims
- No in-app purchases; completely free

**Differentiating features**
- Official product of the Insurance Information Institute
- Integration with major carrier websites for claim submission

**UX patterns**
- Designed for maximum simplicity: open, point camera, annotate, done
- Positioned as a one-time setup tool rather than ongoing inventory management

**Integration points**
- Insurance carrier website integrations for claim submission

**Known gaps**
- Extremely limited feature set — annotation only, no detailed metadata
- No cloud backup independent of device
- No estate planning, moving, or maintenance features
- Unclear whether it remains actively maintained

**Licence / IP notes**
- Free, non-profit product; proprietary

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Item name, description, category, location (room), purchase date, purchase price, and serial number fields
- Photo attachment per item (multiple photos supported)
- Receipt, warranty, and document storage per item
- Cloud backup and sync across devices
- PDF or spreadsheet export of the full inventory
- Search and filter across items
- Barcode scanning for item identification

### Differentiating Features
- AI photo recognition to auto-populate item details from a camera image
- Voice assistant integration for hands-free queries
- Client-side end-to-end encryption for maximum privacy
- Estate planning workflows: beneficiary assignment, PPM export, provenance storytelling
- Replacement value / market price estimation and update
- Insurance-specific report formats (schedule of loss, itemised claim documents)
- Moving mode: room-by-room packing lists, box tracking, arrival checklist
- Self-hosting option for data-sovereign users
- Professional tiers serving real estate agents, restoration contractors, or move managers

### Underserved Areas / Opportunities
- AI-powered room scan: photograph a room and have the system identify and propose catalogue entries for all visible items simultaneously
- Ongoing replacement value tracking from retail databases (not just purchase price)
- Integrated maintenance scheduling tied to specific inventory items (e.g., HVAC filter alerts linked to the HVAC entry)
- Cross-platform moving workflow that uses the existing item database to generate packing lists, box manifests, and delivery checklists
- Estate planning guided flow: step-by-step beneficiary assignment, provenance capture, and legal document export in one workflow
- Resale value estimation: pulling used-market prices for items approaching end-of-life or being considered for downsizing
- Open data portability: most commercial apps lock data in proprietary formats; an open-format (JSON/CSV) standard export is rare

### AI-Augmentation Candidates
- Item identification from photos (currently manual for most apps; AI-native in Vorby and HomeZada)
- Purchase email parsing to auto-create inventory entries
- Serial number OCR from product stickers and labels
- Natural language search across the inventory ("find all items bought before 2018 worth more than $500")
- Depreciation and replacement cost estimation from market data
- Automatic category and room suggestion based on item type and context
- Duplicate detection when adding items that may already be catalogued

---

## Legal & IP Summary

No patent or copyright concerns were identified in the open-source products surveyed (Homebox and Grocy are MIT-licensed). Commercial products (HomeZada, Encircle, SaveOr, Artifcts, Vorby, Sortly) are proprietary SaaS platforms; their feature sets are publicly described and may be studied for design inspiration, but their code, data formats, and UI designs are protected. The NAIC and Insurance Information Institute apps are government/non-profit products and freely available to use. There are no known patents on AI-based household item identification that would block a new open-source implementation, though this area should be monitored as AI computer vision patents are actively filed. ACORD insurance data standards are proprietary specifications (licensed by ACORD) and would require a licence for direct implementation of their XML schemas; however, the general concepts (policy, item, claim) are not protectable.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Item cataloguing: name, description, category, location (room), purchase date, purchase price, current value, serial number
- Multiple photo attachments per item
- Receipt, warranty, and document attachment per item
- Barcode scanning for item lookup (product name, model pre-fill)
- Cloud backup with offline support
- PDF inventory export suitable for insurance claims
- Basic search and filter

**Should-have (v1.1)**
- AI photo recognition to propose item details from a photo
- Purchase confirmation email parsing for automated item entry
- Room-based organisational hierarchy with visualisation
- Warranty expiration reminders
- Multiple home / location support
- Family sharing with view/edit permissions
- CSV/JSON export for data portability

**Nice-to-have (backlog)**
- Moving mode: packing lists, box manifests, delivery checklists from existing inventory
- Estate planning workflow: beneficiary assignment, PPM export, provenance story capture
- Replacement value estimator from live retail price data
- Resale value estimation from used-market data
- Voice assistant integration (Alexa, Siri, Google Assistant)
- Self-hosted deployment option (Docker image with open API)
- Insurance carrier integrations for one-click claim submission
