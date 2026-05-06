# Standards & API Reference

> Project: Home Inventory Manager · Generated: 2026-05-04

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 30173:2023 — Digital twin: Concepts and terminology**
- URL: https://www.iso.org/standard/81442.html
- Defines foundational concepts for digital twins of physical assets, including the lifecycle data model that underpins a comprehensive home inventory. A home inventory that captures item condition, age, and maintenance history is effectively a lightweight digital twin of household contents.

**ISO 16739-1:2024 — Industry Foundation Classes (IFC) for data sharing in construction and facility management**
- URL: https://www.iso.org/standard/84123.html
- The IFC schema defines structured representations for building components, spaces, and equipment. The home inventory domain shares conceptual overlap with IFC's asset management extensions, particularly for built-in appliances and systems. Relevant as a reference data model even though IFC targets professional construction use cases.

**ISO 19650 — Organisation and digitisation of information about buildings and civil engineering works (BIM)**
- URL: https://www.iso.org/standard/68078.html
- Specifies requirements for information management over the asset lifecycle, including unique asset identifiers and information containers. Informs how unique item IDs and change history should be managed in a home inventory.

**ISO/IEC Guide 77 / ISO 13584 (PLIB) — Parts library standard for product properties**
- URL: https://www.iso.org/standard/40440.html
- Defines standardised data element types for product properties (manufacturer, model number, dimensions, material). Relevant for the product metadata schema used when cataloguing household items.

**ISO/IEC 27001:2022 — Information security management systems**
- URL: https://www.iso.org/standard/82875.html
- Provides the framework for protecting sensitive personal and financial data stored in a home inventory system. Particularly relevant given that inventory data includes item valuations, serial numbers, and proof-of-ownership documents that are attractive to burglars if exposed.

---

### W3C & IETF Standards

**Schema.org — Product, ItemList, PropertyValue**
- URL: https://schema.org/Product | https://schema.org/ItemList
- Schema.org's Product type and related PropertyValue structures provide a widely-adopted vocabulary for describing physical items (name, model, serial number, manufacturer, offer price, availability). JSON-LD encoding of inventory items using Schema.org enables interoperability with search engines, insurance portals, and third-party apps.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, self-contained tokens used for authentication and secure data exchange. Essential for mobile and web API security in a home inventory system where access control (family sharing, read-only insurer views) must be enforced.

**RFC 8252 — OAuth 2.0 for Native Apps**
- URL: https://datatracker.ietf.org/doc/html/rfc8252
- Specifies how to implement OAuth 2.0 correctly in mobile and desktop applications, requiring the system browser (not embedded webviews), PKCE, and claimed HTTPS redirect schemes. Directly applicable to the iOS/Android authentication flow.

**RFC 7516 — JSON Web Encryption (JWE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7516
- Standard for encrypting JSON payloads. Relevant to client-side encryption of inventory data before cloud upload, a key differentiator in privacy-first implementations.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the Link header field and hypermedia linking model used in REST APIs. Relevant to HATEOAS-style REST API design for the home inventory data service.

**W3C Verifiable Credentials Data Model**
- URL: https://www.w3.org/TR/vc-data-model/
- An emerging standard for issuing tamper-evident digital claims. Applicable to proof-of-ownership certificates that a home inventory could generate for insurance claims or resale.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing REST APIs. Any home inventory API — whether public or for insurer integrations — should be described with an OpenAPI 3.1 spec to enable code generation, SDK publishing, and integration partner onboarding.

**JSON Schema (draft 2020-12)**
- URL: https://json-schema.org/specification
- Provides the vocabulary for validating the structure of home inventory data payloads. Used to define and validate item records, room structures, document attachments, and export formats.

**ACORD Property & Casualty Data Standards**
- URL: https://www.acord.org/standards-architecture/acord-data-standards/Property_Casualty_Data_Standards
- ACORD defines the insurance industry's canonical data exchange formats (XML and AL3). The ACORD 140 Property Section captures insured property details including location, values, and coverages. A home inventory exporting ACORD-compatible data (licensed usage) would integrate directly with carrier systems and claims platforms.

**ACORD Next-Generation Digital Standards (NGDS)**
- URL: https://www.acord.org/standards-architecture/acord-data-standards/next-generation-digital-standards
- ACORD's API-oriented successor to the ACORD XML batch formats, supporting microservices and fine-grained insurance transactions. Relevant for real-time inventory submission at policy inception or claim creation.

**GS1 Global Trade Item Number (GTIN) / Barcode Standards**
- URL: https://www.gs1.org/standards/barcodes
- GS1 barcodes (EAN-13, UPC-A, GS1-128) are the standard encoding for product identification. A barcode-scanning inventory app must parse GS1 barcodes and look up GTIN records to pre-populate item details (manufacturer, product name, model).

**Open Product Data / Open Grocery / Open Food Facts APIs**
- URL: https://world.openfoodfacts.org/data | https://www.upcitemdb.com/
- Community-maintained product databases that can be queried by barcode for product metadata. Relevant to the product lookup service used when scanning item barcodes during cataloguing.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 | https://openid.net/connect/
- The standard framework for delegated authorisation and identity federation. A home inventory app should implement OIDC for sign-in (supporting Google, Apple, and email providers) and OAuth 2.0 for third-party integrations (e.g., insurance portal access).

**PKCE — Proof Key for Code Exchange (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Required for all mobile OAuth flows to prevent authorisation code interception. Must be implemented in any native iOS or Android application.

**OWASP Mobile Application Security Verification Standard (MASVS)**
- URL: https://mas.owasp.org/MASVS/
- Defines security requirements for mobile applications including local data storage, cryptography, authentication, and network communication. A home inventory app storing sensitive financial and personal data should meet MASVS Level 2.

**OWASP API Security Top 10**
- URL: https://owasp.org/API-Security/
- The canonical checklist for REST API security vulnerabilities. Directly applicable to the home inventory API, particularly around broken object-level authorisation (preventing one user accessing another user's inventory).

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr.eu/
- The EU privacy regulation governing collection, storage, and processing of personal data. A home inventory app processing EU users' financial asset data, photos, and personal documents must implement GDPR-compliant data processing agreements, consent flows, right-to-erasure, and data portability mechanisms. Non-compliance penalties reach 4% of global annual revenue or €20M.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- The California equivalent of GDPR. Any consumer-facing US app must provide opt-out from data sale, right to know, and right to delete.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to external data and tools. A home inventory MCP server would expose tools for querying the inventory ("list all items in the living room worth over $500"), updating item records, and generating insurance reports. This would enable AI assistants (Claude, Copilot, etc.) to help users manage their inventory through natural language without a dedicated UI.
- Relevant MCP tool types: `resource` (read item records, room lists), `tool` (add item, update value, generate report), `prompt` (guided insurance claim flow)

---

## Similar Products — Developer Documentation & APIs

### Encircle

- **Description:** Field documentation platform used by insurance restoration contractors to document property damage and contents inventories. Offers the most complete open API in the category.
- **API Documentation:** https://help.encircleapp.com/hc/en-us/articles/12036459891853-Encircle-API
- **SDKs/Libraries:** No official SDKs; REST API with standard JSON payloads
- **Developer Guide:** https://www.getencircle.com/integrations/
- **Standards:** REST/JSON; Zapier integration
- **Authentication:** API key / OAuth (via Zapier connector)

---

### Grocy

- **Description:** Open-source self-hosted household management system with a comprehensive REST API covering inventory, tasks, recipes, and equipment. The most developer-friendly open-source option in the category.
- **API Documentation:** Available at `<instance-url>/api` with built-in Swagger UI
- **SDKs/Libraries:** Community Python client (`pygrocy`); no official SDKs
- **Developer Guide:** https://github.com/grocy/grocy — README and wiki
- **Standards:** REST/JSON; OpenAPI 3.0 (Swagger); fully documented endpoints
- **Authentication:** API key (header: `GROCY-API-KEY`)

---

### Sortly

- **Description:** Visual inventory management SaaS with REST API access on paid business tiers. Originally designed for small business inventory but adapts to home use.
- **API Documentation:** Available to business/enterprise customers on request
- **SDKs/Libraries:** No official SDKs; Zapier connector available
- **Developer Guide:** https://help.sortly.com/
- **Standards:** REST/JSON; Zapier integration
- **Authentication:** API key (paid tiers)

---

### Homebox

- **Description:** Open-source, self-hosted home inventory system written in Go. Provides a REST API that can be integrated with Home Assistant and other home automation tools.
- **API Documentation:** Built into the running instance at `<host>/api/docs`
- **SDKs/Libraries:** None official; community Home Assistant integration
- **Developer Guide:** https://github.com/sysadminsmedia/homebox (active fork)
- **Standards:** REST/JSON; OpenAPI spec generated from Go annotations
- **Authentication:** Bearer token (JWT) via login endpoint

---

### Google Cloud Vision API

- **Description:** Google's general-purpose computer vision API, used by multiple home inventory apps for AI item recognition. Provides object detection, label detection, OCR, and logo detection from images.
- **API Documentation:** https://cloud.google.com/vision/docs
- **SDKs/Libraries:** Official SDKs for Python, Node.js, Java, Go, Ruby, PHP, C#
- **Developer Guide:** https://cloud.google.com/vision/docs/how-to
- **Standards:** REST/JSON and gRPC; OpenAPI
- **Authentication:** OAuth 2.0 / Service Account / API key

---

### Google ML Kit (On-Device Vision)

- **Description:** On-device machine learning SDK for iOS and Android. Includes object detection and tracking with categories including "home goods" — enabling AI item recognition without sending photos to a server.
- **API Documentation:** https://developers.google.com/ml-kit/vision/object-detection
- **SDKs/Libraries:** Official Android (Kotlin/Java) and iOS (Swift/ObjC) SDKs
- **Developer Guide:** https://developers.google.com/ml-kit/vision/object-detection/android
- **Standards:** On-device; no network calls for inference
- **Authentication:** None (on-device processing)

---

### API4AI Furniture & Household Stuff Recognition API

- **Description:** Specialist cloud API for identifying and counting furniture and household items in images. Purpose-built for the home inventory use case, unlike general-purpose vision APIs.
- **API Documentation:** https://api4.ai/apis/object-detection
- **SDKs/Libraries:** REST samples on GitHub: https://github.com/api4ai/household-stuff-recognition
- **Developer Guide:** https://api4.ai/docs
- **Standards:** REST/JSON; returns bounding boxes and labels in JSON
- **Authentication:** API key

---

### ACORD Standards (Insurance Data Exchange)

- **Description:** The insurance industry's canonical data exchange standards, covering property details, claims, and policy data. Direct integration would enable one-click inventory submission to insurance carriers.
- **API Documentation:** https://www.acord.org/standards-architecture/acord-data-standards/next-generation-digital-standards
- **SDKs/Libraries:** Vendor-provided; IBM and other enterprise vendors offer ACORD adapters
- **Developer Guide:** https://hicronsoftware.com/blog/acord-data-standards-insurance/
- **Standards:** ACORD XML, AL3, NGDS (REST/JSON)
- **Authentication:** Carrier-specific; typically OAuth 2.0 or mTLS

---

### UPC Item DB / Open Barcode Databases

- **Description:** Community and commercial barcode lookup services that return product name, brand, model, and category from a scanned barcode. Core dependency for auto-populating item details from a barcode scan.
- **API Documentation:** https://www.upcitemdb.com/api/v1 | https://world.openfoodfacts.org/data
- **SDKs/Libraries:** REST; no official SDKs (simple GET requests)
- **Developer Guide:** https://www.upcitemdb.com/api/doc
- **Standards:** REST/JSON; GS1 GTIN barcode format
- **Authentication:** API key (UPC Item DB); open (Open Food Facts)

---

## Notes

**Gap: No open standard for home inventory data exchange.** Unlike insurance (ACORD) or construction (IFC/COBie), there is no published open standard for the format of a personal home inventory export. This is an opportunity: defining and publishing a JSON Schema for a portable home inventory record (items, rooms, attachments, valuations) could become a de-facto standard adopted by the community and referenced by open-source tools.

**Gap: ACORD licensing cost.** Direct implementation of ACORD XML schemas requires an ACORD membership and licence. An open-source home inventory tool may be better served by designing an export format that maps conceptually to ACORD without implementing ACORD schemas directly, allowing insurance carriers to accept and translate the data on their end.

**Emerging: AI home scan via multi-modal models.** Multimodal LLMs (GPT-4o, Claude 3.x) can now analyse a photo of a room and produce a structured list of identifiable items. This eliminates the need for purpose-built object detection APIs for the initial cataloguing step, though purpose-built models (ML Kit, API4AI) remain useful for high-accuracy serial number OCR and barcode decoding.

**Emerging: MCP as an integration layer.** An MCP server wrapping the home inventory API would allow Claude, Copilot, and other AI assistants to query and update the inventory through natural language — enabling use cases such as "add the TV I just bought" from a conversation, or "generate an insurance report for everything in the living room" without opening the app.
