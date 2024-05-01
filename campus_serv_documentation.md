# CampusServ: Master Technical Documentation

**Project Name:** CampusServ
**Team:** Group 88
**Institution:** Kwame Nkrumah University of Science and Technology (KNUST), Kumasi, Ghana
**Date:** June 2026

---

## PROJECT IDENTITY

### What CampusServ Is
CampusServ is a peer-to-peer (P2P) student service marketplace tailored specifically for the ecosystem of the Kwame Nkrumah University of Science and Technology (KNUST). It serves as a digital bridge between students who have skills or services to offer (Providers) and students who urgently need those services (Students/Buyers). Whether it's last-minute printing delivery to a hostel, hair braiding in a hall of residence, tutoring for an upcoming mid-semester exam, or fixing a broken laptop, CampusServ digitizes and organizes the informal campus gig economy.

### The Core Thesis
Generic marketplaces (like Upwork or local classifieds) fail in a Ghanaian campus environment due to several hyper-local constraints:
1. **Trust and Safety:** Students are hesitant to invite strangers into their hostels (e.g., Brunei, Republic Hall). A platform restricted to verified KNUST students significantly lowers the trust barrier.
2. **Economic Reality:** Budgets are extremely tight. A hyper-local bidding system allows price discovery that matches student purchasing power.
3. **Friction of Discovery:** Currently, services are advertised via chaotic WhatsApp status updates and group broadcasts. This is ephemeral, unsearchable, and lacks accountability. CampusServ structures this chaos.
4. **Geography:** Services are often hyper-local (e.g., "I need someone in Gaza hostel").

### Primary Goals
1. **Economic Empowerment:** Allow students to monetize their free time and skills securely.
2. **Friction Reduction:** Replace WhatsApp negotiations and cash-in-hand unreliability with structured bidding and integrated mobile money (MoMo) / Paystack payments in GHS.
3. **Trust Building:** Implement a closed-loop review and identity verification system.

---

## SECTION 1 — TECHNOLOGY STACK DEEP DIVE

### Frontend — React Native / Expo

**Why React Native?**
React Native was chosen over Flutter or Native iOS/Android because of the team's existing proficiency in JavaScript/TypeScript and the vast ecosystem of React libraries. It allows a single codebase to output native iOS and Android apps, which is critical since the KNUST student body is split roughly 60/40 between Android and iOS. A PWA was rejected because it lacks the deep OS integration required for reliable background location tracking and native push notifications.

**Why Expo?**
Expo (specifically the managed workflow with EAS Build) was selected to eliminate the overhead of managing native iOS (Xcode) and Android (Gradle) build environments. 
- **Expo Router:** Used for file-based routing, mirroring Next.js paradigms.
- **EAS Build:** Used for cloud compilation.
- **Expo Notifications:** Handles in-app alerts and local scheduling.

**Project Structure:**
```text
mobile/
├── assets/             # Fonts, images, splash screens
├── src/
│   ├── components/     # Reusable UI components (Buttons, Cards, Inputs)
│   ├── navigation/     # Navigators (AppNavigator.tsx, ProviderNavigator.tsx)
│   ├── screens/        # Full-page views
│   ├── services/       # API clients and interceptors
│   ├── store/          # Zustand state management slices
│   └── styles/         # Theme tokens and global styles
├── App.tsx             # Entry point
└── app.json            # Expo configuration
```

**Navigation:**
Using `@react-navigation/native` and `@react-navigation/bottom-tabs`. 
- **AppNavigator:** The main stack for students, containing the Home Feed, Search, and Orders.
- **ProviderNavigator:** A distinct tab structure for service providers to manage their active listings, incoming bids, and earnings.

**Design System:**
- **Aesthetic:** Glassmorphism. Chosen to feel premium, modern, and distinct from the cluttered, flat interfaces of typical university portals.
- **Palette:** 
  - Midnight Navy (`#0A192F`) for deep backgrounds.
  - Electric Violet (`#7C3AED`) for primary interactive elements.
  - Neon Green (`#10B981`) for success states and active indicators.
- **State Management:** `zustand` is used for lightweight global state (e.g., current user profile, theme preference), while `@tanstack/react-query` handles all server state (caching, deduping, background fetching).
- **API Communication:** `axios` with interceptors that inject the JWT bearer token into every outgoing request and seamlessly attempt token refreshes on `401 Unauthorized` responses.

### Backend — Spring Boot Microservices

**Why Spring Boot?**
Spring Boot was chosen over Node.js for its robust type safety, mature dependency injection, and unparalleled ecosystem for building enterprise-grade microservices (Spring Cloud). 

**Microservices Architecture:**
A microservices approach was chosen to allow independent scaling. For example, the `request-service` (bidding) will experience much higher read/write volume than the `user-service`.

1. **api-gateway:** The single entry point for all mobile traffic. Handles routing, rate limiting, and initial JWT signature verification.
2. **eureka-server:** Service registry for internal discovery.
3. **auth-service:** Issues JWTs, handles login, registration, and password hashing (BCrypt).
4. **user-service:** Manages student profiles, verification status, and provider portfolios.
5. **request-service:** Manages the core marketplace: service listings, bidding lifecycle, and order management.
6. **job-service:** Handles background cron jobs and scheduled tasks (e.g., bid expiry).
7. **payment-service:** Integrates with Paystack for GHS transactions, escrow, and payout logic.
8. **supporting-service:** Handles auxiliary features like WebSockets (chat), Locations (Google Maps), and Notifications.

**Communication:**
Inter-service communication is primarily synchronous REST via OpenFeign for immediate data needs, falling back to asynchronous patterns for eventual consistency.

### Database Layer

**PostgreSQL:**
PostgreSQL is the primary datastore for all services, chosen for its strong ACID compliance, JSONB support (used for flexible listing attributes), and PostGIS extension for spatial queries. Each microservice has its own logical database schema to enforce bounded contexts (e.g., `jdbc:postgresql://localhost:5433/campusserv`).

**Schema Highlights:**
- **Users Table:** `id` (UUID), `student_id` (String, unique), `email` (String, unique), `password_hash`, `role` (Enum: STUDENT, PROVIDER, ADMIN).
- **ServiceListings Table:** `id` (UUID), `provider_id` (FK), `title`, `description`, `base_price` (Decimal), `status` (Enum).
- **Bids Table:** `id` (UUID), `listing_id` (FK), `student_id` (FK), `amount` (Decimal), `status`.

**Migrations:**
Managed exclusively by Flyway (`spring.flyway.enabled=true`), ensuring reproducible environments from development to production.

### Real-Time Layer — WebSockets

WebSockets are localized within the `supporting-service` (`WebSocketConfig.java`, `ChatController.java`). 
- **Use Case:** Required for the in-app messaging system where students and providers negotiate details after a bid is accepted, and for live order status updates.
- **Implementation:** Spring WebSocket with STOMP. The frontend connects upon app initialization (if authenticated) and listens on private queues (`/user/queue/messages`).

### Location Services — Google Maps

- **Implementation:** The `supporting-service` exposes a `LocationController.java`. The frontend uses `react-native-maps`.
- **Purpose:** To calculate distances between students and providers, ensuring services are fulfilled within realistic KNUST campus boundaries (e.g., preventing a student in Ayeduase from accepting a time-sensitive delivery bid from a provider in Bomso, unless agreed upon).

---

## SECTION 2 — CORE FEATURES: PROCESS-BY-PROCESS BREAKDOWN

### 2.1 — User Registration & Authentication
1. **Initiation:** User enters KNUST student email and Student ID on the mobile app.
2. **Validation:** `auth-service` validates the email domain (`@st.knust.edu.gh`).
3. **Creation:** A new User record is created in `user-service` with role `STUDENT`.
4. **JWT Issuance:** `auth-service` generates a short-lived Access Token and a long-lived Refresh Token.
5. **Storage:** The frontend stores these tokens in `expo-secure-store`.

### 2.2 — Service Listing Creation
1. **Action:** A Provider taps "Create Listing".
2. **Data Entry:** Title, description, category (e.g., "Graphic Design", "Laundry"), and base pricing are entered. Images are selected via `expo-image-picker`.
3. **Upload:** Images are compressed and sent to object storage; URLs are returned.
4. **Commit:** The `request-service` saves the `ServiceListing` record. Status defaults to `ACTIVE`.

### 2.3 — Multi-Provider Bidding Lifecycle
1. **Request:** A student browses listings and initiates a "Request for Service".
2. **Notification:** The provider receives an in-app notification via `supporting-service`.
3. **Bidding:** The provider responds with a specific bid amount in GHS and estimated completion time.
4. **Acceptance:** The student reviews the bid in the `RequestDetailsScreen`. Upon tapping "Accept", the `request-service` transitions the Bid status to `ACCEPTED` and generates an `Order` record. All other competing bids are marked `REJECTED`.

### 2.4 — Order / Booking Management
1. **In Progress:** Once an order is generated, the provider is expected to fulfill the service. Both parties gain access to a real-time chat room via WebSockets.
2. **Completion:** The provider marks the order as "Delivered".
3. **Confirmation:** The student must confirm the completion. If confirmed, funds are released. If disputed, an admin intervention workflow is triggered.

### 2.5 — Payments (GHS)
1. **Gateway:** Paystack is used to handle GHS transactions (`payment-service/src/main/resources/application.yml`).
2. **Flow:** When a student accepts a bid, they are prompted to pay via Mobile Money (MTN/Vodafone) using the Paystack checkout.
3. **Escrow:** The `payment-service` receives the Paystack webhook (`whsec_...`), verifies the signature, and marks the internal `Transaction` as `HELD`.
4. **Payout:** Upon order confirmation, the funds (minus platform commission) are credited to the Provider's internal wallet for withdrawal.

---

## SECTION 3 — ARCHITECTURE & SYSTEM DESIGN

### System Diagram (ASCII)
```text
[ React Native Mobile App ]
           |
           v (HTTPS / REST)
+-----------------------------------+
|         API Gateway (8080)        |
+-----------------------------------+
      |           |           |
      v           v           v
+-------+   +---------+   +---------+
| Auth  |   | User    |   | Request |
| (8087)|   | Service |   | Service |
+-------+   +---------+   +---------+
      |           |           |
      v           v           v
 [Postgres]  [Postgres]  [Postgres]

+-----------------------------------+
|       Supporting Service          |<--- (WebSockets) ---> Mobile App
|    (Chat, Location, Notifs)       |
+-----------------------------------+
```

### Deployment & Scalability
Currently designed for a Dockerized deployment. The API Gateway serves as a load balancer and security checkpoint. By isolating the `request-service` (which handles the heavy read/write load of bidding), we can scale it horizontally independent of the `auth-service`.

---

## SECTION 4 — UX & DESIGN SYSTEM

### Glassmorphism Aesthetic
Glassmorphism is applied to modal overlays and prominent cards (like active bids). It uses an `rgba(255, 255, 255, 0.1)` background with a blur filter and a subtle 1px white border. This mimics physical frosted glass, providing a high-tech, layered feel that appeals to a younger demographic.

### Typography
- **Headings:** Bold, geometric sans-serif (e.g., Inter or Poppins) to convey clarity.
- **Body:** System fonts for maximum readability.

### Empty States
Every screen (Active Orders, Inbox, Search Results) features a custom, friendly empty state with an SVG illustration and a call-to-action (e.g., "You have no active bids. Want to browse services?").

---

## SECTION 5 — DATA FLOWS & INTEGRATION MAPS

### Flow: Payment Processing
1. **Client:** Student taps "Pay & Accept Bid". Frontend calls `/api/v1/payments/initialize`.
2. **payment-service:** Creates a pending `Transaction` in Postgres. Calls Paystack API with amount and callback URL.
3. **Paystack:** Returns an authorization URL.
4. **Client:** Frontend opens WebView to Paystack. Student enters MoMo PIN.
5. **Paystack:** Sends asynchronous Webhook to `/api/v1/payments/webhook`.
6. **payment-service:** Verifies signature, updates `Transaction` to `SUCCESS`, and makes internal API call to `request-service` to update Order status.

---

## SECTION 6 — DEVELOPMENT PROCESS & PROJECT MANAGEMENT

### Workflow
- **Version Control:** Git with a Trunk-Based Development approach. Feature branches are merged into `main` via Pull Requests.
- **Team Structure (Group 88):** Divided into Frontend (React Native), Backend (Spring Boot), and QA/Product.
- **Testing:** JUnit and Mockito for backend unit tests. API layer tested manually via Postman collections during the initial phases.

---

## SECTION 7 — BUSINESS LOGIC & RULES DOCUMENTATION

1. **Bid Floor/Ceiling:** A provider cannot submit a bid that is 50% lower or 200% higher than the `base_price` of the listing.
2. **Review Window:** Students have exactly 7 days after an order is marked `COMPLETED` to leave a review. After 7 days, the system auto-assigns a default 5-star rating to ensure providers are not penalized for student apathy.
3. **Commission:** The platform deducts a flat 5% commission from the final agreed bid amount before crediting the provider.

---

## SECTION 8 — GLOSSARY & ENTITY DEFINITIONS

- **User:** The base identity record. Can assume roles of Student or Provider dynamically.
- **ServiceListing:** An advertisement created by a Provider detailing a service they offer.
- **Bid:** A specific financial and temporal offer made by a Provider in response to a Student's request.
- **Order:** A finalized contract created when a Student accepts a Bid.
- **Transaction:** The financial ledger record representing the movement of GHS via Paystack.

---

## APPENDIX — DECISION LOG

| Date | Decision | Alternatives Considered | Rationale |
|------|----------|-------------------------|-----------|
| Q1 | Use Expo Managed Workflow | React Native CLI | Reduced devops overhead; EAS build solves local compiling issues for team members lacking Macs. |
| Q2 | Microservices over Monolith | Spring Boot Monolith | Prepares the architecture for independent scaling of the bidding engine, despite initial complexity overhead. |
| Q3 | Remove Push Notifications | Firebase Cloud Messaging | Simplified delivery relying purely on WebSockets and in-app polling to guarantee delivery regardless of aggressive Android battery optimization. |
| Q4 | Paystack Integration | Flutterwave, Hubtel | Paystack offers superior developer experience and highly reliable webhooks for MTN Mobile Money. |

---
*End of Document*
