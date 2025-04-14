# Custom Multi-Tenant Mail Server System (Haraka + Node.js)

## Overview

This project provides a complete, multi-tenant mail server system built entirely with Node.js technologies, primarily using Haraka for the SMTP server. It allows companies to sign up, manage their own domains and email users, ensuring data isolation between tenants. A key feature is mandatory DNS verification for domain ownership before a domain can be used for sending emails.

## Goal

To build a secure, scalable, and cross-platform (Windows & Linux) mail system where:
- Each user belongs to a specific company.
- The first user signing up for a company becomes the admin.
- Admins can manage domains, email IDs, and potentially API access for their company.
- Data is strictly segregated per company (tenant).
- Domain ownership must be verified via DNS TXT records before activation.
- The system uses Haraka for SMTP, avoiding dependencies like Postfix.

## Features

- **Multi-Tenancy:** Securely isolates data and configuration per company.
- **Admin Role:** Automatic admin assignment upon company signup.
- **Domain Management:** Add and manage custom domains.
- **DNS Verification:** Mandatory TXT record check to confirm domain ownership.
- **Email User Management:** Create email addresses under verified domains.
- **API Security:** Routes secured via JWT and filtered by company context.
- **Email Storage:** Simple filesystem storage for incoming emails (customizable).
- **Email Sending:** Uses Nodemailer for outbound mail via the local Haraka instance.
- **Cross-Platform:** Designed to run on both Windows and Linux.

## Tech Stack

- **SMTP Server:** [Haraka](https://haraka.github.io/) (Node.js)
- **Backend Framework:** [Express.js](https://expressjs.com/)
- **Database ORM:** [Prisma](https://www.prisma.io/)
- **Database:** SQLite (default) / PostgreSQL (recommended for production)
- **Frontend Build Tool:** [Vite](https://vitejs.dev/)
- **Frontend Framework:** [React](https://reactjs.org/) (+ [Tailwind CSS](https://tailwindcss.com/))
- **Webmail UI:** Custom (Built with Vite+React) or [SnappyMail](https://github.com/the-djmaze/snappymail)
- **Email Parsing:** [mailparser](https://nodemailer.com/extras/mailparser/) (Node.js)
- **Email Sending:** [Nodemailer](https://nodemailer.com/)
- **Authentication:** JWT + [bcrypt](https://github.com/kelektiv/node.bcrypt.js)
- **Email Storage:** Filesystem (default) or S3-compatible storage
- **Real-time (Optional):** [Socket.IO](https://socket.io/)

## Prerequisites

- **Operating System:** Windows 10/11 or Ubuntu 20.04+
- **Domain:** A public domain name (e.g., `yourcompany.com`) for which you can manage DNS records.
- **Node.js:** Version 16 or higher (Install from [nodejs.org](https://nodejs.org/)).
- **Open Ports:** Ensure ports 25 (SMTP), 587 (Submission), and potentially 993 (IMAP if using SnappyMail/custom IMAP) are open on your server/firewall.

## Installation & Setup

1.  **Install Node.js:**
    *   **Windows:** Download and run the installer from [nodejs.org](https://nodejs.org/).
    *   **Linux (Ubuntu/Debian):**
        ```bash
        sudo apt update
        sudo apt install nodejs npm -y
        ```

2.  **Install Haraka Globally:**
    ```bash
    npm install -g Haraka
    ```

3.  **Clone the Repository (or Create Project Structure):**
    ```bash
    git clone <your-repo-url> complete-email-solutions
    cd complete-email-solutions
    # OR manually create the structure outlined below
    ```

4.  **Initialize Haraka Configuration (if setting up manually):**
    *   Navigate to the intended Haraka config directory (e.g., `my-mail-server` inside the project root).
    *   Run: `haraka -i .`

5.  **Initialize Frontend Project (inside `web-ui`):**
    *   Navigate into the `web-ui` directory: `cd web-ui`
    *   Run the Vite creation command (choose React and TypeScript/JavaScript):
        ```bash
        # Using npm
        npm create vite@latest . --template react-ts
        # OR using yarn
        # yarn create vite . --template react-ts
        # OR select options interactively: npm create vite@latest .
        ```
    *   Install frontend dependencies:
        ```bash
        npm install # Or yarn install
        ```
    *   Navigate back to the project root: `cd ..`

6.  **Install Backend Dependencies (Root Level):**
    ```bash
    npm install # Or yarn install
    ```

7.  **Setup Database:**
    *   Configure your database connection string in `.env` (see `.env.example`).
    *   Run Prisma migrations:
        ```bash
        npx prisma migrate dev --name init
        ```

8.  **Configure Haraka:**
    *   Edit `my-mail-server/config/plugins` to enable necessary plugins (see list below).
    *   Configure `my-mail-server/config/auth_flat_file.ini` for initial admin/testing access (use bcrypt for passwords).
    *   Ensure the custom `store_email.js` plugin (or your chosen storage mechanism) is in `my-mail-server/plugins/`.

9.  **Configure DNS:**
    *   Set up MX, SPF, DKIM, and DMARC records for your mail domain pointing to your server's IP address. See the "DNS Setup" section below.

10. **Start the Servers:**
    *   Start the Haraka SMTP server (consult Haraka documentation for running as a service).
    *   Start the Express.js backend API (from root): `npm run start` or `npm run dev`.
    *   Start the Vite development server for the UI (from `web-ui` directory): `npm run dev`.

## Configuration

### Haraka Plugins (`my-mail-server/config/plugins`)

Ensure at least the following plugins are enabled (remove the `#`):

```
connect_tls
helo
auth_flat_file
data.headers
rcpt_to.in_host_list
queue/smtp_forward
# Custom storage plugin (relative path from config dir)
../plugins/store_email
```

### Authentication (`my-mail-server/config/auth_flat_file.ini`)

Add users for SMTP authentication (e.g., for testing sending via port 587). Passwords **must** be bcrypt hashed.

```ini
[users]
testuser@yourdomain.com = $2b$10$.....................................................
```

Generate hashes using Node.js:
```javascript
const bcrypt = require('bcrypt');
bcrypt.hash("yourChosenPassword", 10).then(console.log);
```

### Environment Variables (`.env`)

Configure database connection strings, JWT secrets, etc.

```env
DATABASE_URL="postgresql://user:password@host:port/database?schema=public" # Or file:./dev.db for SQLite
JWT_SECRET="your-super-secret-jwt-key"
# Add other relevant variables (e.g., S3 credentials if used)
```

## Database Schema (Prisma)

The schema (`prisma/schema.prisma`) defines the multi-tenant structure:

- `Company`: Represents a tenant.
- `User`: Represents an admin user belonging to a `Company`.
- `Domain`: Represents a domain owned by a `Company`, including verification status and token.
- `EmailUser`: Represents an email account created under a verified `Domain`.

## Key Workflows

### 1. Signup & Authentication

- **Endpoint:** `POST /auth/signup`
- **Process:**
    1. Accepts company name, user email, and password.
    2. Creates the `Company` if it doesn't exist (using `prisma.company.upsert`).
    3. Creates the `User`, linking them to the `Company` and marking them as `isAdmin=true`.
    4. Hashes the password using `bcrypt`.
    5. Returns a JWT token for session management.

### 2. Domain DNS Verification

- **Process:**
    1. Admin adds a domain via the UI (`POST /domains`).
    2. Backend generates a unique `verificationToken` and saves it with the `Domain` record (`verified=false`).
    3. UI instructs the admin to add a TXT record to their domain's DNS settings:
        - **Type:** `TXT`
        - **Name:** `_verify.yourdomain.com` (replace `yourdomain.com`)
        - **Value:** The generated `verificationToken`
    4. Admin clicks a "Verify DNS" button in the UI.
    5. Backend (`POST /domains/:id/verify`) uses Node.js `dns.resolveTxt` to query the TXT record.
    6. If the queried value matches the stored `verificationToken`, the backend updates the `Domain` record to `verified=true`.
    7. Email users can only be created under domains where `verified=true`.

## API Security

All API routes that manage company-specific resources (domains, email users, etc.) **must** be protected by JWT authentication middleware. The middleware should decode the token, identify the user, and attach `req.user` (containing `userId` and `companyId`) to the request object.

Route handlers must then use `req.user.companyId` to filter database queries, ensuring users can only access data belonging to their own company.

**Example:**
```javascript
// Middleware (simplified)
const authenticateJWT = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (token) {
    jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
      if (err) return res.sendStatus(403);
      req.user = user; // Contains { userId, companyId, ... }
      next();
    });
  } else {
    res.sendStatus(401);
  }
};

// Route Handler
app.get('/api/domains', authenticateJWT, async (req, res) => {
  const domains = await prisma.domain.findMany({
    where: { companyId: req.user.companyId } // Filter by company!
  });
  res.json(domains);
});
```

## Email Sending (Nodemailer)

Use Nodemailer configured to send through the local Haraka instance (typically on port 25 or 587 if authentication is required).

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
    host: "localhost", // Or your Haraka server address
    port: 25,          // Use 587 if authentication is needed
    secure: false,     // Set to true if using TLS on port 465 (less common for local)
    // auth: { user: 'smtp_user', pass: 'smtp_pass' } // If using port 587 with auth
    tls: {
        rejectUnauthorized: false // Necessary if using self-signed certs locally
    }
});

async function sendEmail() {
    try {
        let info = await transporter.sendMail({
            from: '"Sender Name" <sender@yourverifieddomain.com>',
            to: "recipient@example.com",
            subject: "Hello ✔",
            text: "Hello world?",
            html: "<b>Hello world?</b>",
        });
        console.log("Message sent: %s", info.messageId);
    } catch (error) {
        console.error("Error sending email:", error);
    }
}
```

## Webmail UI

- **Option 1: Custom React + Tailwind:** Provides full control over the UI/UX but requires significant development effort.
- **Option 2: SnappyMail:** A modern, fast webmail client (fork of RainLoop). Easier to integrate but requires IMAP/SMTP access to the mail storage. If using filesystem storage, you'll need an IMAP server (like Dovecot) pointing to the mail storage location.

## Admin Panel UI (React)

A dedicated interface (likely part of the main web application) for company admins to:
- Manage domains (add, view status, trigger verification).
- Manage email users (create/delete accounts under verified domains).
- View basic logs or statistics (optional).
- Manage API tokens (optional).

## DNS Setup (Essential for Mail Delivery)

Configure these DNS records for your primary mail domain (`yourdomain.com`) at your DNS provider:

1.  **MX (Mail Exchanger):** Points incoming mail to your server.
    *   **Type:** `MX`
    *   **Name:** `@` or `yourdomain.com.`
    *   **Value:** `mail.yourdomain.com.` (or your server's hostname)
    *   **Priority:** `10` (lower number means higher priority)

2.  **A (Address):** Maps your mail hostname to your server's IP.
    *   **Type:** `A`
    *   **Name:** `mail` (or the hostname used in MX record)
    *   **Value:** `your-server-ip-address`

3.  **TXT (SPF - Sender Policy Framework):** Helps prevent spoofing by listing authorized sending IPs.
    *   **Type:** `TXT`
    *   **Name:** `@` or `yourdomain.com.`
    *   **Value:** `"v=spf1 ip4:your-server-ip-address -all"` (Replace IP. `-all` means only this IP is allowed)

4.  **TXT (DMARC - Domain-based Message Authentication, Reporting & Conformance):** Policy for SPF/DKIM failures.
    *   **Type:** `TXT`
    *   **Name:** `_dmarc`
    *   **Value:** `"v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com"` (`p=none` starts in monitoring mode)

5.  **TXT (DKIM - DomainKeys Identified Mail):** Cryptographically signs outgoing emails. Requires a Haraka plugin (like `dkim_sign`) or external setup. The specific record name/value depends on the selector and key generated.

6.  **TXT (Domain Verification):** As described above, used by the application itself.
    *   **Type:** `TXT`
    *   **Name:** `_verify`
    *   **Value:** `<verificationToken>` (Managed per-domain within the app)

## Project Folder Structure (Suggestion)

```
/complete-email-solutions
|-- /api                  # Express.js backend source
|   |-- /routes           # API route handlers (auth, domains, users, etc.)
|   |-- /middleware       # Authentication, error handling, etc.
|   |-- /services         # Business logic (DNS checking, etc.)
|   |-- /utils            # Helper functions
|   |-- server.js         # Express app entry point
|-- /my-mail-server       # Haraka configuration and plugins
|   |-- /config           # Haraka config files (main, plugins, auth_flat_file.ini)
|   |-- /plugins          # Custom Haraka plugins (store_email.js)
|-- /prisma               # Prisma schema and migrations
|   |-- schema.prisma
|   |-- migrations/
|-- /web-ui               # React Frontend (Admin Panel + Webmail if custom)
|   |-- /src
|   |-- package.json
|-- /emails               # Default storage for raw .eml files (ensure .gitignore)
|-- .env                  # Environment variables (DB connection, JWT secret)
|-- .env.example          # Example environment file
|-- .gitignore
|-- package.json          # Project dependencies (Express, Prisma, etc.)
|-- README.md             # This file
```

## Optional Features

- **Real-time Inbox Updates:** Use Socket.IO triggered by the `store_email` Haraka plugin.
- **Scheduled Emails:** Implement using `node-cron` and database flags.
- **DKIM Signing:** Integrate a DKIM signing plugin for Haraka for better email deliverability.
- **Per-Company API Tokens:** Allow companies to generate tokens for programmatic access.
- **Attachment Handling:** Enhance `store_email` or use dedicated storage (like S3) and link in the DB.
- **Advanced Queueing:** Replace `queue/smtp_forward` with more robust queueing like `queue/rabbitmq` if needed.

## Contributing

(Add guidelines if you plan to accept contributions)

## License

(Specify your project's license, e.g., MIT, Apache 2.0)
