# Co-Working Space Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A unified, AI-native platform for co-working operators to manage memberships, bookings, billing, access control, and community engagement -- replacing fragmented SaaS tools with a single open-source system.

Co-Working Space Management is an open-source platform that gives flex-space operators everything they need to run their spaces: configurable membership plans, real-time desk and room booking, automated billing, smart-lock integration, and a built-in community feed. It targets independent operators, small chains, and enterprise networks who currently cobble together multiple closed-source SaaS products at significant cost.

---

## Why Co-Working Space Management?

- **Every incumbent is closed-source SaaS.** Nexudus, Coworks, Archie, Spacebring, Cobot, and OfficeRnD Flex all lock operators into proprietary platforms with no access to source code and no self-hosting option. The only open-source alternatives (Nadine Project, MRBS) are narrow in scope and lack modern UX.
- **Per-location subscription pricing scales poorly.** Platforms like Nexudus charge per location, making multi-site expansion expensive. Operators with 10+ locations face substantial recurring costs for features that should be commodity infrastructure.
- **AI capabilities are shallow or absent.** Nexudus offers basic AI-powered demand predictions in its analytics module; no other incumbent provides AI-assisted community moderation, dynamic pricing, occupancy forecasting, or proactive member engagement.
- **Community engagement is an afterthought.** Only Archie and Spacebring treat community feeds as a first-class feature. Most platforms bolt on basic announcements and call it done, leaving operators without tools to drive retention through member interaction.
- **Integration ecosystems are fragmented.** Many platforms rely on Zapier for integrations rather than offering native connectors. Archie stands out with 40+ native integrations, but the rest leave operators to build their own glue.

---

## Key Features

### Membership and Billing

- Configurable plan types: hot-desk, dedicated desk, private office, virtual office, day pass, and credit packs
- Automated recurring billing via Stripe with pro-rated charges, failed-payment retry, and dunning workflows
- Day-pass and drop-in sales for walk-in or online purchase
- Mid-month plan changes, unused credit rollovers, and corporate invoice consolidation
- Digital receipts and integration with QuickBooks and Xero for accounting sync

### Booking and Space Management

- Real-time availability calendar for desks, meeting rooms, phone booths, and custom resource types
- Conflict prevention with optimistic locking to eliminate double-bookings
- Multi-location support with shared member accounts and location-specific resource pools
- Kiosk and room-display booking interfaces for on-site self-service
- Analytics on desk and room utilisation rates, peak occupancy times, and revenue per square metre

### Access Control and Security

- Automated door and WiFi permissions granted and revoked based on active membership or booking
- Integration with smart-lock providers: Kisi, Salto, Brivo, Hakuna Smart
- BLE-based local credential caching for offline mobile unlock
- Visitor pre-registration, host notification, and temporary guest access
- Tablet-based visitor check-in at the front desk

### Community and Events

- Member-facing social feed with posts, announcements, likes, and comments
- Event creation with RSVP, ticketing, and room booking linkage
- Member directory for networking and community discovery
- Direct messaging between members
- Content moderation tools with configurable posting permissions by plan type

### Member Portal and Mobile App

- Self-service account management: booking, invoice history, plan upgrades, and community participation
- Mobile-native apps for iOS and Android
- White-label branding with custom domains, configurable email templates, and branded app appearance
- Operator admin dashboard with role-based access control and customizable views

---

## AI-Native Advantage

This project targets the most significant gaps in the current market by embedding AI across the operator workflow. Predictive occupancy forecasting helps operators optimise staffing and space layout based on historical check-in patterns and demand signals. Dynamic pricing recommendations adjust rates to maximise both revenue and utilisation during off-peak periods. AI-powered community moderation learns community norms to filter spam and flag inappropriate content without manual intervention. Member sentiment analysis across feedback, reviews, and community posts surfaces churn risk and satisfaction trends before they become problems.

---

## Tech Stack & Deployment

The platform is designed for self-hosted, cloud, or hybrid deployment to give operators full control over their data and infrastructure. Access control integration uses an open API layer to avoid vendor lock-in across the fragmented smart-lock hardware market. Stripe serves as the primary payment integration, with an extensible payment gateway architecture for additional processors (PayPal, Square). Calendar interoperability covers Google Calendar, Outlook, and iCal export. The notification stack supports both email and SMS delivery, with native connectors for Slack and Microsoft Teams.

---

## Market Context

The co-working space management software market reached approximately USD 1.94 billion in 2025 and is forecast to grow to USD 2.21 billion in 2026, driven by hybrid-work adoption and the expansion of multi-location flex-space networks. The market is dominated by closed-source SaaS vendors charging per-location subscription fees, with Nexudus alone supporting 3,000+ locations across 90+ countries. Primary buyers are independent co-working operators, multi-site flex-space chains, and corporate real estate teams managing hybrid office portfolios.

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
