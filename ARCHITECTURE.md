# Multi-Tenant Mail Server System Architecture
====================================

[ Public Internet ]
    |
    | (DNS Records: MX, SPF, TXT, DKIM, DMARC)
    v
[ Domain Provider ]
    | (TXT _verify.yourdomain.com = token)  <-- For App Verification
    | (MX mail.yourdomain.com)              <-- Directs Mail to Server
    | (SPF, DKIM, DMARC for email auth)     <-- Email Authentication Standards
    v
[ Haraka SMTP Server ] (Listens on Ports: 25, 587, potentially 993 for IMAP)
    ├── Config Directory: /my-mail-server/config/
    │   └── plugins (File listing enabled plugins)
    │       ├── connect_tls         (Enables TLS encryption)
    │       ├── helo                (Checks HELO/EHLO commands)
    │       ├── auth_flat_file      (Authenticates users via config/auth_flat_file.ini)
    │       │   └── config/auth_flat_file.ini (e.g., admin@domain.com: bcrypt_hash)
    │       ├── data.headers        (Parses email headers)
    │       ├── rcpt_to.in_host_list (Accepts mail only for configured domains)
    │       ├── queue/smtp_forward  (Forwards mail to another SMTP server or local delivery agent)
    │       └── ../plugins/store_email (Reference to custom plugin)
    │
    ├── Plugins Directory: /my-mail-server/plugins/
    │   └── store_email.js (Custom Plugin Logic)
    │       └── Hook: hook_data_post (Runs after email data is received)
    │           └── Action: Saves raw email content to /emails/<timestamp>.eml (or S3)
    │
    ├── Incoming Email Data Flow:
    │   └── External Sender -> DNS -> Haraka Port 25 -> Plugins Process -> store_email saves -> Stored in /emails/
    │
    └── Outgoing Email Handling (Triggered by Backend API):
        └── Backend API (via Nodemailer) -> Haraka Port 25/587 -> queue/smtp_forward -> External Recipient SMTP Server

[ Backend API ] (Express.js + Prisma ORM)
    ├── Entry Point: /server.js (Initializes Express App)
    ├── API Routes Directory: /api/routes/
    │   │
    │   ├── POST /auth/signup
    │   │   └── Logic: Creates Company + Admin User -> Returns JWT
    │   │
    │   ├── GET /domains (Requires JWT Auth)
    │   │   └── Logic: Lists domains for companyId
    │   │
    │   ├── POST /domains (Requires JWT Auth)
    │   │   └── Logic: Creates domain + verificationToken
    │   │
    │   ├── GET /users (Requires JWT Auth)
    │   │   └── Logic: Lists users for companyId
    │   │
    │   ├── POST /users (Requires JWT Auth)
    │   │   └── Logic: Creates email users (post-domain verification)
    │   │
    │   └── /emails (future routes for email management)
    │
    ├── Auth Middleware:
    │   └── JWT + bcrypt (secures routes by companyId)
    │
    ├── DNS Verification Service:
    │   └── Uses Node.js `dns.resolveTxt` to check _verify.<domain>.com TXT record
    │
    └── Dependencies:
        ├── Nodemailer (For Mail Sending)
        │   └── Connects to Haraka (localhost:25 or 587)
        └── Mailparser (For Email Parsing, if needed beyond Haraka)

[ Database ] (SQLite / PostgreSQL)
    ├── Schema Definition (`prisma/schema.prisma`):
    │   ├── model Company { id, name, createdAt, users User[], domains Domain[] }
    │   ├── model User { id, email, password, isAdmin, companyId, company Company }
    │   ├── model Domain { id, name, companyId, verified, verificationToken, company Company, users EmailUser[] }
    │   └── model EmailUser { id, email, password, domainId, domain Domain }
    │
    └── Storage Characteristics:
        └── Multi-tenant: All queries filtered by `companyId` to ensure data isolation.

[ Storage Options ] (For Email Content)
    ├── Option 1: Filesystem
    │   └── Directory: `/emails`
    │   └── Format: Raw .eml files (e.g., `<timestamp>.eml`)
    │   └── Managed by: `store_email.js` Haraka plugin
    │
    └── Option 2: S3 (or compatible object storage)
        └── Configuration: Requires AWS SDK/credentials.
        └── Benefit: Scalability, durability.
        └── Managed by: Modified `store_email.js` or dedicated storage service.

[ Webmail UI ] (Client-Side Application)
    ├── Option 1: Custom Build (React + Vite + Tailwind CSS)
    │   ├── Key Components:
    │   │   ├── Inbox View (List/Read Emails)
    │   │   ├── Compose View (Write/Send Emails)
    │   │   ├── Domain Management Interface (for Admins)
    │   │   └── User Management Interface (for Admins)
    │   ├── Interaction: Communicates with Backend API for all actions.
    │   └── Optional Feature: Real-time inbox updates using Socket.IO.
    │
    └── Option 2: SnappyMail (Pre-built Webmail Client)
        └── Integration: Requires IMAP/SMTP server access (e.g., Dovecot if using filesystem storage). Less direct integration with the custom backend API.

[ Admin Panel ] (Part of the React + Vite + Tailwind CSS UI)
    ├── Key Features:
    │   ├── Create Domain (UI triggers `POST /domains`, displays token)
    │   ├── Check DNS Verification (UI triggers `POST /domains/:id/verify`)
    │   ├── Create Email IDs (UI triggers `POST /users` only after domain verification)
    │   ├── View Logs (Requires backend logging implementation and API endpoint)
    │   └── Data Filtering: All displayed data is fetched via API, ensuring company isolation.
    │
    └── Connects To:
        └── Backend API endpoints (e.g., `/api/domains`, `/api/users`, `/api/logs`)

[ Optional Features Implementation ]
    ├── Socket.IO:
    │   └── Backend: Emit event from `store_email.js` or API after saving email.
    │   └── Frontend: Listen for events to refresh inbox.
    ├── node-cron:
    │   └── Backend: Scheduled job checks DB for emails flagged for sending.
    ├── DKIM Signing:
    │   └── Haraka: Requires `dkim_sign` plugin configuration.
    └── API Tokens:
        └── Backend: Add DB model for tokens, create API routes for management.

[ High-Level Data Flow Example (Signup to Email) ]
    1. User signs up via UI -> `POST /auth/signup` -> Company & Admin User created in DB -> JWT returned to UI.
    2. Admin logs in, navigates to Domains section in UI.
    3. Admin adds `example.com` via UI -> `POST /domains` -> Domain record created (verified=false, token='abc') -> Token 'abc' shown in UI.
    4. Admin goes to DNS Provider, adds TXT record: `_verify.example.com` = `abc`.
    5. Admin clicks "Verify" in UI -> `POST /domains/:id/verify` -> Backend checks DNS -> Finds 'abc' -> Updates Domain `verified=true`.
    6. Admin navigates to Email Users section, creates `info@example.com` via UI -> `POST /users` -> Backend verifies domain ownership -> EmailUser created in DB.
    7. External user sends email to `info@example.com` -> DNS MX directs to Haraka -> Haraka processes -> `store_email.js` saves to `/emails/12345.eml`.
    8. Admin logs into Webmail UI -> UI fetches emails via API (which reads from `/emails/` or uses IMAP) -> Email is displayed.
    9. Admin composes reply via UI -> `POST /send-email` -> Backend uses Nodemailer -> Sends via Haraka -> Delivered to recipient.

[ Suggested Project Folder Structure ]
    /complete-email-solutions/
    ├── /api/                 # Express.js backend source (routes, middleware, services)
    ├── /my-mail-server/      # Haraka configuration and custom plugins
    │   ├── /config/          # Haraka config files (plugins, auth_flat_file.ini, etc.)
    │   └── /plugins/         # Custom Haraka plugins (e.g., store_email.js)
    ├── /prisma/              # Prisma schema and migrations
    │   └── schema.prisma
    ├── /web-ui/              # React Frontend (Admin Panel + Webmail) - Managed by Vite
    │   ├── /src/
    │   └── package.json      # Frontend dependencies
    ├── /emails/              # Default storage for raw .eml files (add to .gitignore)
    ├── .env                  # Environment variables (DB connection, JWT secret) - DO NOT COMMIT
    ├── .env.example          # Example environment file
    ├── .gitignore
    ├── package.json          # Root project dependencies (Express, Prisma Client, etc.)
    ├── server.js             # Backend API entry point (e.g., starts Express server)
    ├── README.md             # Main project documentation
    └── ARCHITECTURE.md       # This file
