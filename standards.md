# Standards & API Reference

> Project: Co-Working Space Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015** — Quality Management Systems
- Official URL: https://www.iso.org/iso-9001-quality-management.html
- Relevant to coworking space management platforms that require consistent service quality, process documentation, and continuous improvement frameworks.

**ISO/IEC 27001:2022** — Information Security Management Systems
- Official URL: https://www.iso.org/iiec-27001-information-security-management.html
- Critical for coworking platforms handling member data, payment information, and access credentials. Covers data protection, access controls, and information security policies required for SaaS compliance.

**ISO/IEC 27701:2019** — Privacy Information Management
- Official URL: https://www.iso.org/standard/71670.html
- Extends ISO 27001 with specific privacy management requirements. Coworking platforms must implement privacy controls for member personal data, billing information, and activity tracking to meet GDPR/CCPA compliance.

**ISO 45001:2018** — Occupational Health & Safety Management
- Official URL: https://www.iso.org/iso-45001-occupational-health-and-safety.html
- Relevant for coworking operators managing facility safety, emergency procedures, and member safety compliance in shared workspaces.

### W3C & IETF Standards

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- Official URL: https://tools.ietf.org/html/rfc5545
- Essential for calendar data interchange in booking systems. Defines the standard data format for encoding calendar events, which enables integration with Google Calendar, Outlook, and other calendar applications used by members.

**RFC 4791 — Calendaring Extensions to WebDAV (CalDAV)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc4791
- Protocol for accessing, managing, and sharing calendar data over HTTP/HTTPS. Coworking platforms can expose calendar data (availability, bookings) via CalDAV to integrate with client calendar applications.

**RFC 5546 — iCalendar Transport-Independent Interoperability Protocol (iTIP)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc5546
- Specifies the mechanism for exchanging calendar invitations and meeting requests between scheduling systems, supporting automated booking confirmations and meeting synchronisation.

**RFC 6638 — Scheduling Extensions to CalDAV**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6638
- Extends CalDAV with scheduling capabilities, allowing calendar servers to handle booking requests, confirmations, and cancellations in a standardized way.

**RFC 7230 — HTTP/1.1 Message Syntax and Routing**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7230
- Foundational HTTP protocol specification. All REST APIs for coworking platforms must comply with HTTP semantics for request/response handling.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6750
- Specifies how OAuth 2.0 bearer tokens are used in HTTP Authorization headers. Standard for authenticating API requests in coworking management integrations.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP methods (GET, POST, PUT, DELETE) and status codes used in REST APIs. Essential for designing RESTful coworking management APIs.

### Data Model & API Specifications

**OpenAPI 3.0/3.1 Specification**
- Official URL: https://spec.openapis.org/oas/v3.0.3.html (v3.0.3) or https://spec.openapis.org/oas/v3.1.0 (v3.1)
- Industry standard for documenting REST APIs. Coworking platforms should expose their APIs via OpenAPI specifications to enable automated documentation, code generation, and integration testing.

**JSON Schema Draft 7+**
- Official URL: https://json-schema.org/
- Standard for validating JSON data structures. Used in API request/response validation, configuration schemas, and data model documentation for coworking platforms.

**JSON:API Specification**
- Official URL: https://jsonapi.org/
- Specification for building consistent RESTful JSON APIs. Useful for standardising coworking API response formats (resource objects, relationships, metadata).

**GraphQL Specification**
- Official URL: https://spec.graphql.org/
- Alternative to REST for API design. GraphQL allows flexible querying of coworking data (bookings, memberships, billing), reducing over-fetching and under-fetching of data.

**Protocol Buffers (protobuf)**
- Official URL: https://developers.google.com/protocol-buffers
- Binary data serialisation format used for efficient service-to-service communication. Useful for internal APIs between microservices in a coworking platform (e.g., access control <-> billing).

### Security & Authentication Standards

**OAuth 2.0 Authorization Framework**
- Official URL: https://tools.ietf.org/html/rfc6749
- Standard for delegated access and third-party integrations. Coworking platforms should support OAuth 2.0 for securely granting third-party apps (HR systems, accounting software) access to member data and booking information.

**OpenID Connect 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0. Enables single sign-on (SSO) for coworking members using corporate identity providers, improving member experience and reducing password management burden.

**JSON Web Token (JWT) — RFC 7519**
- Official URL: https://tools.ietf.org/html/rfc7519
- Standard for securely transmitting claims in compact form. Used in stateless API authentication, mobile app authentication, and short-lived access credentials.

**General Data Protection Regulation (GDPR)**
- Official URL: https://gdpr-info.eu/
- EU data protection law. Coworking platforms handling members in EU must comply with GDPR requirements: consent management, data minimisation, right to access/delete, data breach notification, DPA with processors.

**California Consumer Privacy Act (CCPA)**
- Official URL: https://oag.ca.gov/privacy/ccpa
- California state privacy law. Coworking platforms with California members must provide: right to know, access, delete, and opt-out of data sharing. Requires privacy policies and opt-out mechanisms.

**OWASP Top 10**
- Official URL: https://owasp.org/www-project-top-ten/
- Current guide (2021): https://owasp.org/www-project-top-ten/2021/
- Security best practices for web applications. Coworking platforms should mitigate risks: injection attacks, broken authentication, sensitive data exposure, XML external entities (XXE), broken access control, security misconfiguration, XSS, insecure deserialization, using components with known vulnerabilities, insufficient logging.

**NIST Cybersecurity Framework**
- Official URL: https://www.nist.gov/cyberframework
- Voluntary framework for managing cybersecurity risk. Useful for coworking platforms to structure security programs around: Identify, Protect, Detect, Respond, Recover.

**PCI DSS 4.0** (if handling credit card data directly)
- Official URL: https://www.pcisecuritystandards.org/
- Payment Card Industry compliance. Coworking platforms integrating payment processing must implement PCI DSS controls (encryption, secure transmission, access control, regular testing).

### Access Control & Smart Lock Standards

**Kisi API Specification**
- Official URL: https://docs.kisi.io/platform/apis/
- JSON-based REST API for managing access control. Supports user provisioning, lock/unlock operations, access rights management, and integration with coworking booking systems.

**Salto KS Connect API**
- Official URL: https://developer.saltosystems.com/ks/connect-api/
- REST-based API for integrating Salto cloud-based access control. Supports access provisioning, revocation, and real-time lock state management for coworking access control.

**RFID and NFC Standards** (ISO/IEC 14443, ISO/IEC 15693)
- Relevant for mobile credential systems using contactless cards or smartphone-based unlock mechanisms in coworking spaces.

### Payment Processing Standards

**Stripe API Specification**
- Official URL: https://docs.stripe.com/api
- RESTful API for payments, invoicing, and billing. Industry standard for SaaS subscription billing. Supports recurring charges, webhooks, idempotency, and comprehensive error handling.

**PCI DSS 3.2.1** (Deprecated; PCI DSS 4.0 current)
- Standard for secure payment processing. Coworking platforms using Stripe typically avoid direct PCI compliance by delegating payment processing to Stripe.

---

## Similar Products — Developer Documentation & APIs

### Nexudus

- **Description:** Enterprise-grade coworking management platform supporting 3,000+ locations globally. Offers multi-location management, automated billing, booking, and access control integration.
- **API Documentation:** https://developers.nexudus.com/reference/getting-started-with-your-api-1
- **SDKs/Libraries:** TypeScript/JavaScript client available at https://github.com/oddbit/nexudus-js
- **Developer Guide:** https://help.nexudus.com/docs/api-getting-started and https://developers.nexudus.com/reference/about-the-public-api
- **Standards:** REST API (JSON over HTTP), bearer token authentication, webhook support
- **Authentication:** Bearer tokens (OAuth-style) and Basic Auth via TLS 1.2+

### Cobot

- **Description:** White-label coworking software emphasising simplicity and ease of use. Strong focus on membership management, resource booking, and billing automation.
- **API Documentation:** https://dev.cobot.me/api-docs
- **SDKs/Libraries:** No official SDKs listed; webhook and REST API available
- **Developer Guide:** https://helpcenter.cobot.me/en/articles/5144468-using-our-api and access control integration guide at https://dev.cobot.me/page/access-control-system-integration
- **Standards:** RESTful API (JSON), webhook-based events for bookings, payments, and cancellations
- **Authentication:** API token-based authentication

### Archie

- **Description:** G2-rated #1 coworking software with strong community engagement features. Integrates 40+ third-party tools and provides white-label customisation.
- **API Documentation:** https://archieapp.co/integrations (open API available; detailed REST docs via integrations page)
- **SDKs/Libraries:** Not explicitly documented; Zapier integration available
- **Developer Guide:** https://archieapp.co/coworking-software/automations and integrations page
- **Standards:** REST API with webhook support, integrates 40+ platforms natively
- **Authentication:** API key or OAuth-based authentication (details on integrations page)

### Spacebring

- **Description:** Unified coworking platform automating billing, access control, and member onboarding. Emphasises remote access control management without on-site hardware dependencies.
- **API Documentation:** https://developer.spacebring.com/introduction
- **SDKs/Libraries:** Not explicitly listed; REST API and webhooks available
- **Developer Guide:** https://help.spacebring.com/en/articles/7900838-get-started-with-api-webhooks
- **Standards:** RESTful API (HTTPS enforced), JSON request/response, webhook-driven event notifications
- **Authentication:** Client ID and Client Secret (available via Network settings > Developers)

### OfficeRnD Flex

- **Description:** Analytics-first coworking platform with advanced occupancy dashboards, CRM features, and visitor management. Emphasises data-driven space optimisation.
- **API Documentation:** Not extensively documented in public resources; integrations via Zapier and direct connections
- **SDKs/Libraries:** Integration via Zapier and Slack API
- **Developer Guide:** https://www.officernd.com/ (integrations and automation pages)
- **Standards:** RESTful integrations, Zapier-based automation, Slack webhook integration
- **Authentication:** OAuth-based integrations (specifics via Zapier or direct partnership)

### Stripe (Payment Processing)

- **Description:** Leading payment processing and billing platform used by most coworking management software. Handles recurring subscriptions, invoicing, and webhook-driven billing events.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** Official SDKs for JavaScript/TypeScript, Python, Java, Go, Ruby, .NET, PHP, and more at https://docs.stripe.com/libraries
- **Developer Guide:** https://docs.stripe.com/billing/subscriptions/webhooks (subscription webhooks) and https://docs.stripe.com/webhooks (general webhook guide)
- **Standards:** RESTful API, JSON request/response, webhook-based event notifications, OpenAPI specification available
- **Authentication:** API keys (live and test mode), Bearer token via Authorization header (RFC 6750)

### Google Calendar API

- **Description:** Calendar integration standard used by most coworking platforms to sync bookings and availability with member personal calendars.
- **API Documentation:** https://developers.google.com/calendar/api
- **SDKs/Libraries:** Official client libraries for Python, Java, JavaScript, Go, Ruby, PHP, .NET
- **Developer Guide:** https://developers.google.com/calendar/api/quickstart
- **Standards:** RESTful API, JSON, supports iCalendar format (RFC 5545) for import/export
- **Authentication:** OAuth 2.0 (for user calendars) or service account authentication

### Microsoft Outlook/Microsoft Graph API

- **Description:** Enterprise calendar and contact API used for integrating coworking bookings with corporate calendar systems via Microsoft 365.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/overview
- **SDKs/Libraries:** Official SDKs for .NET, Python, JavaScript, Java, Go, PowerShell
- **Developer Guide:** https://learn.microsoft.com/en-us/graph/auth/
- **Standards:** RESTful API, JSON, supports CalDAV (RFC 4791) and iCalendar format
- **Authentication:** OAuth 2.0 via Microsoft identity platform

### Zapier Platform

- **Description:** No-code automation platform enabling coworking software to integrate with 6,000+ applications without custom development.
- **API Documentation:** https://zapier.com/platform/
- **SDKs/Libraries:** Zapier CLI and Node.js SDK for building custom integrations
- **Developer Guide:** https://platform.zapier.com/build/intro
- **Standards:** Webhook-based trigger/action architecture, JSON payloads, OAuth 2.0 for app authentication
- **Authentication:** OAuth 2.0, custom authentication schemes per integration

---

## Notes

### Emerging Standards & Gaps

- **Mobile Credential Standards:** No universal standard for smartphone-based access (Apple Wallet, Google Pay integration with smart locks). Most implementations are vendor-specific. Open standards in this area (e.g., via Seam API, device abstraction) are still emerging.

- **Community Moderation Standards:** No established standard for community feed content moderation, member reporting, or automated spam detection in SaaS platforms. This remains a gap where AI-augmented solutions could add value.

- **Flexible Billing Models:** Most platforms assume monthly recurring billing. Industry standards for usage-based, outcome-based, or hybrid billing models are underdeveloped; Stripe Metered Billing is one option but lacks wide adoption.

- **Occupancy Analytics Standardisation:** No industry standard data model for occupancy, utilisation, and space analytics metrics. Vendors use proprietary reporting schemas, making cross-platform benchmarking difficult.

- **MCP Server Protocol:** Model Context Protocol (if applicable for AI-assisted features) is emerging. Official specifications are available at https://modelcontextprotocol.io/ but adoption in coworking tools is nascent.

### Vendor Lock-in Considerations

- **Access Control Hardware:** No unified API standard across Kisi, Salto, Brivo, Hakuna. Coworking platforms must implement vendor-specific SDKs or use abstraction layers (e.g., Seam API) to avoid lock-in.

- **Calendar Integrations:** iCalendar (RFC 5545) and CalDAV (RFC 4791) are open standards, reducing lock-in. Most platforms support Google Calendar and Outlook via standard APIs.

- **Payment Processing:** Stripe is near-universal for coworking platforms. Implementing a payment abstraction layer can reduce future switching costs.

### Recommended Architectural Alignment

1. **API Design:** Follow OpenAPI 3.1 specification for all REST endpoints. Publish auto-generated API documentation using Swagger/Redoc.
2. **Authentication:** Implement OAuth 2.0 for third-party integrations; JWT for internal stateless auth.
3. **Data Exchange:** Use iCalendar (RFC 5545) for calendar data to ensure compatibility with Google Calendar, Outlook, and other calendar clients.
4. **Billing:** Integrate Stripe webhooks for reliable, event-driven payment handling. Implement idempotency keys on all write operations.
5. **Access Control:** Abstract away vendor-specific APIs (Kisi, Salto) using an internal gateway pattern or third-party service (e.g., Seam) to reduce lock-in.
6. **Privacy & Security:** Implement GDPR/CCPA compliance by default (consent management, data retention, audit logging). Target ISO 27001 certification for enterprise customers.
7. **Webhooks:** Support webhook subscriptions for real-time integrations (booking events, payment events, access changes) to enable automation via Zapier and custom integrations.
