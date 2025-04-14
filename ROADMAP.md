# Project Roadmap: Multi-Tenant Mail Server System

This document outlines the planned development phases for building the Complete Email Solutions multi-tenant mail server.

## Phase 1: Foundation & Initial Setup (Estimated Time: 1-2 days)

*   [ ] **Goal:** Establish the basic project structure, install core dependencies, and set up initial configurations.
*   [ ] **Tasks:**
    *   [ ] Create project directory structure (`api`, `my-mail-server`, `prisma`, `web-ui`, `emails`).
    *   [ ] Initialize `package.json` at the root and install backend dependencies (Express, Prisma, bcrypt, jsonwebtoken, dotenv, etc.).
    *   [ ] Initialize Vite + React + TS project in `web-ui` and install frontend dependencies.
    *   [ ] Initialize Haraka configuration within `my-mail-server` (`haraka -i .`).
    *   [ ] Set up basic Express server in `api/server.js`.
    *   [ ] Initialize Prisma (`npx prisma init`) and define initial `schema.prisma` with `Company` and `User` models.
    *   [ ] Create `.env` and `.env.example` files for environment variables (DB URL, JWT Secret).
    *   [ ] Set up basic `.gitignore`.

## Phase 2: Core Authentication & Tenancy (Estimated Time: 2-3 days)

*   [ ] **Goal:** Implement user signup, login, and JWT-based authentication, ensuring basic multi-tenant separation.
*   [ ] **Tasks:**
    *   [ ] Implement `POST /api/auth/signup` endpoint:
        *   Hashing passwords (`bcrypt`).
        *   `upsert` logic for `Company`.
        *   Create `User` linked to `Company`.
        *   Generate JWT token.
    *   [ ] Implement `POST /api/auth/login` endpoint:
        *   Find user by email.
        *   Compare hashed password (`bcrypt.compare`).
        *   Generate JWT token.
    *   [ ] Create JWT authentication middleware (`/api/middleware/auth.js`).
        *   Verify token.
        *   Attach `req.user = { userId, companyId, isAdmin }` to requests.
    *   [ ] Apply auth middleware to protected routes (initially maybe a test route like `GET /api/me`).
    *   [ ] Implement basic Prisma query filtering based on `req.user.companyId` in test routes.
    *   [ ] Create basic Signup/Login forms in the `web-ui`.

## Phase 3: Haraka Integration & Email Storage (Estimated Time: 2-3 days)

*   [ ] **Goal:** Configure Haraka to receive emails and store them using a custom plugin.
*   [ ] **Tasks:**
    *   [ ] Configure `my-mail-server/config/plugins` to enable necessary plugins (`connect_tls`, `helo`, `data.headers`, `rcpt_to.in_host_list`, `../plugins/store_email`).
    *   [ ] Configure `my-mail-server/config/host_list` with initial test domain(s).
    *   [ ] Create the custom `my-mail-server/plugins/store_email.js` plugin.
        *   Implement `hook_data_post`.
        *   Write logic to stream `connection.transaction.message_stream` to a file in the `/emails` directory (e.g., `emails/${Date.now()}.eml`).
    *   [ ] Test basic email receiving using a tool like `swaks` or `telnet` to send an email to Haraka.
    *   [ ] Ensure Haraka starts correctly and the plugin loads/functions.

## Phase 4: Domain Management & DNS Verification (Estimated Time: 3-4 days)

*   [ ] **Goal:** Allow admins to add domains and implement the DNS verification flow.
*   [ ] **Tasks:**
    *   [ ] Add `Domain` model to `prisma/schema.prisma` (including `verified`, `verificationToken`). Run `prisma migrate dev`.
    *   [ ] Implement `POST /api/domains` endpoint (Auth required):
        *   Generate unique `verificationToken`.
        *   Create `Domain` record linked to `req.user.companyId`.
    *   [ ] Implement `GET /api/domains` endpoint (Auth required):
        *   Fetch domains filtered by `req.user.companyId`.
    *   [ ] Implement DNS checking logic (using `dns.resolveTxt`) in a service (`/api/services/dnsVerifier.js`).
    *   [ ] Implement `POST /api/domains/:id/verify` endpoint (Auth required):
        *   Fetch domain by `id` and `companyId`.
        *   Call DNS verification service.
        *   Update `Domain.verified` status if TXT record matches token.
    *   [ ] Create UI components in `web-ui` for:
        *   Listing domains.
        *   Adding a new domain (displaying the required TXT record/token).
        *   Triggering the verification check.

## Phase 5: Email User Management (Estimated Time: 2-3 days)

*   [ ] **Goal:** Enable admins to create email accounts under verified domains.
*   [ ] **Tasks:**
    *   [ ] Add `EmailUser` model to `prisma/schema.prisma`. Run `prisma migrate dev`.
    *   [ ] Implement `POST /api/users` endpoint (Auth required):
        *   Validate that the target `Domain` exists, belongs to the `companyId`, and is `verified`.
        *   Hash password (`bcrypt`).
        *   Create `EmailUser` record linked to the `Domain`.
    *   [ ] Implement `GET /api/users` endpoint (Auth required):
        *   Fetch `EmailUser` records, likely filtered by `domainId` and ensuring the domain belongs to the `companyId`.
    *   [ ] Create UI components in `web-ui` for:
        *   Listing email users per domain.
        *   Adding new email users (form should only allow selection of verified domains).

## Phase 6: Email Sending Integration (Estimated Time: 2-3 days)

*   [ ] **Goal:** Integrate Nodemailer to allow authenticated users to send emails via Haraka.
*   [ ] **Tasks:**
    *   [ ] Install `nodemailer`.
    *   [ ] Configure Nodemailer transporter in the backend to connect to `localhost:25` (or `localhost:587` if auth is needed later).
    *   [ ] Implement a basic `POST /api/send-email` endpoint (Auth required):
        *   Receive `to`, `subject`, `body`, `fromEmailUserId`.
        *   Validate `fromEmailUserId` belongs to the authenticated user's company and a verified domain.
        *   Use Nodemailer transporter to send the email.
    *   [ ] Create a basic "Compose" form in the `web-ui` to test sending.
    *   [ ] (Optional) Configure Haraka `auth_flat_file` and Nodemailer for authenticated sending via port 587 if required.

## Phase 7: Webmail/Admin UI Development (Estimated Time: 5-7 days)

*   [ ] **Goal:** Build out the primary user interface for administration and potentially basic webmail functionality.
*   [ ] **Tasks:**
    *   [ ] Structure the React application (routing, state management).
    *   [ ] Implement Tailwind CSS for styling.
    *   [ ] Build out the Admin Dashboard components (integrating with API endpoints from previous phases).
    *   [ ] Refine Domain and Email User management UI/UX.
    *   [ ] **If building custom webmail:**
        *   Implement API endpoints (`GET /api/emails`, `GET /api/emails/:id`) to retrieve stored emails (requires parsing `.eml` files or using an IMAP interface).
        *   Create Inbox, Email Detail, and Compose views in React.
    *   [ ] **If using SnappyMail:**
        *   Set up SnappyMail separately.
        *   Potentially configure an IMAP server (like Dovecot) to serve emails from the `/emails` directory if filesystem storage is used.

## Phase 8: Refinement, Security & DKIM (Estimated Time: 3-5 days)

*   [ ] **Goal:** Enhance security, improve error handling, and set up DKIM for better deliverability.
*   [ ] **Tasks:**
    *   [ ] Add robust input validation (e.g., using `express-validator`) to all API endpoints.
    *   [ ] Implement comprehensive error handling middleware in Express.
    *   [ ] Review and enhance security measures (e.g., rate limiting on auth endpoints).
    *   [ ] Configure Haraka DKIM signing plugin (`dkim_sign`). Generate keys, set up DNS records.
    *   [ ] Test DKIM signing.
    *   [ ] Refine logging throughout the application (Haraka and API).

## Phase 9: Deployment Preparation (Estimated Time: 2-4 days)

*   [ ] **Goal:** Prepare the application for deployment.
*   [ ] **Tasks:**
    *   [ ] Choose deployment strategy (Docker, PaaS, VM).
    *   [ ] Create Dockerfiles for API, Haraka (if using Docker).
    *   [ ] Configure production database (e.g., PostgreSQL). Update `.env` for production.
    *   [ ] Set up production build for the `web-ui` (`npm run build`).
    *   [ ] Configure a reverse proxy (like Nginx or Caddy) if needed.
    *   [ ] Finalize DNS records (MX, SPF, DKIM, DMARC) for the production domain.
    *   [ ] Set up SSL/TLS certificates (e.g., Let's Encrypt).

## Phase 10: Testing & Documentation Finalization (Estimated Time: Ongoing + 2-3 days)

*   [ ] **Goal:** Ensure application stability and complete documentation.
*   [ ] **Tasks:**
    *   [ ] Write unit/integration tests for critical API endpoints and services.
    *   [ ] Perform end-to-end testing of core workflows (signup, domain verification, email send/receive).
    *   [ ] Review and update `README.md`, `ARCHITECTURE.md`, and `ROADMAP.md`.
    *   [ ] Add deployment instructions to `README.md`.

---

**Note:** Estimated times are rough guides and can vary based on complexity and developer experience. Some tasks can be parallelized. Testing should ideally occur throughout each phase.
