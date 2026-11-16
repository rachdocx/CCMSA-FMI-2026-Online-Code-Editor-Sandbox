# Online Editor Compiler & Sandbox

Project for "Cloud Computing using Microsoft Azure", completed as part of the BSc in Computer Science curriculum at the University of Bucharest third year, first semester.

---

##  Problem

Setting up local compilers and toolchains is tedious, especially when you just want to quickly write and run some code. This solution is a browser-based code editor and compiler that lets users write, run, and save code in multiple languages - with per-user file management and AI debugging.

---

##  Functional Requirements

The platform provides a comprehensive suite of developer capabilities directly within the browser:

* **User Authentication & Identity Management**:
  * Register, sign in, and authenticate securely using **Microsoft Entra ID** (OAuth 2.0 / OpenID Connect) via email/password, Microsoft accounts, or social providers (GitHub, Google) with optional Multi-Factor Authentication (MFA).
  * Manage active sessions via JWT tokens and view user profile details (`GET /auth/me`).

* **Project Management**:
  * Create new programming projects configured with a target runtime environment (e.g., Python, C/C++, JavaScript, Java).
  * View, browse, and switch between projects owned by the authenticated user (`GET /projects`).
  * Rename project titles, update project configurations, or delete entire projects along with their underlying source code files and metadata.

* **File & Workspace Management**:
  * Create, open, edit, save, rename/move, and delete source code files and directories in a virtual project tree.
  * Edit code with syntax highlighting, autocomplete, line numbers, and indentation support powered by **Monaco Editor** (the core editor behind VS Code).
  * Automatically persist files to **Azure Blob Storage** under isolated, per-user folder paths (`users/{userId}/projects/{projectId}/...`).

* **Code Execution & Sandboxing**:
  * Submit code files for one-click compilation and execution (`POST /execute`).
  * Provide optional standard input (`stdin`) to test interactive scripts and user prompts.
  * Run code inside ephemeral, network-disabled **Azure Container Instances** tailored to each language environment.
  * Enforce safety limits to prevent abuse: 30-second execution timeout, 256 MB RAM ceiling, and read-only filesystem except `/tmp`.
  * Return and display execution results in real-time, including standard output (`stdout`), error messages (`stderr`), exit status codes, and execution duration.

* **Execution History & Observability**:
  * Automatically log every code execution run into **Azure Cosmos DB** (timestamp, language, code snapshot, exit code, and outputs).
  * Browse and review execution history per project (`GET /execute/history`) to inspect past runs, compare outputs, and track performance.

* **AI-Assisted Debugging**:
  * Request automated debugging support with a single click whenever execution fails or produces compiler/runtime errors (`POST /ai/debug`).
  * Send the faulty source code, runtime language, and error logs to **Azure OpenAI Service (GPT-4o)** for deep diagnosis.
  * Receive plain-language explanations of the root cause, suggested fixes, and ready-to-apply corrected code snippets.

---

### Component Responsibilities

| Azure Component | Role | Observation |
|---|---|---|
| **Azure Static Web Apps** | Host the React + Monaco Editor frontend | - |
| **Azure Functions** (C# / .NET 8 Isolated Worker) | Serverless API layer — all business logic | Built on ASP.NET Core integration, pay-per-execution, scales to zero, strongly-typed Azure SDKs |
| **Azure Container Instances** | Isolated code execution per request | Each submission runs in a fresh container — safe, isolated, disposable. Supports custom Docker images per language |
| **Azure Blob Storage** | Store user files in a virtual folder hierarchy | Cheapest storage option, native folder-like structure via path prefixes, scalable to any size |
| **Azure Cosmos DB** (NoSQL) | User profiles, project metadata, execution history | Globally distributed, schema-flexible, perfect for varied metadata. Cloud-native choice |
| **Azure OpenAI Service** | AI-powered debugging assistant | GPT-4o analyzes code + error output and suggests fixes |
| **Microsoft Entra ID** | User registration, authentication, and JWT tokens | Enterprise-grade cloud identity (OAuth 2.0 & OpenID Connect). Supports email/password, Microsoft accounts, and social logins (Google, GitHub) via Entra External ID |
| **Azure Application Insights** | Monitoring, logging, performance tracking | End-to-end observability, tracks API latency, error rates, container spin-up times |

---
##  Architecture Overview

![Architecture Diagram](architecture.drawio.png)
## Database Design

| Reason | Explanation |
|---|---|
| **Schema flexibility** | User metadata, project configs, and execution logs have different shapes — NoSQL handles this naturally |
| **Cloud-native** | Managed service, auto-scaling, built-in replication — fits the course theme |
| **Cost-efficient at low scale** | Free tier: 1000 RU/s + 25 GB — more than enough for a university project |
| **Fast reads** | Execution history queries by userId are fast with partition key design |


---

## API Endpoints

Base URL: `https://<app-name>.azurewebsites.net/api`

### Authentication (Microsoft Entra ID)

Authentication and identity management are delegated directly to **Microsoft Entra ID** (OAuth 2.0 / OpenID Connect) via MSAL (Microsoft Authentication Library) in the React frontend:

- **Sign-in & Sign-up**: Handled via Microsoft Entra ID interactive flow or Azure Static Web Apps built-in auth (`/.auth/login/aad`). Supports email/password, Microsoft accounts, and social identity providers (Google, GitHub) via Microsoft Entra External ID with built-in MFA.
- **JWT Bearer Token**: Upon login, the client receives an access token signed by Microsoft Entra ID and attaches it as `Authorization: Bearer <token>` on all API requests.
- **Backend Validation**: Azure Functions (.NET 8 Isolated Worker with ASP.NET Core JWT Bearer middleware) validates the RS256 signature against Entra ID's JWKS endpoint (`https://login.microsoftonline.com/{tenantId}/discovery/v2.0/keys`), verifying issuer (`iss`), audience (`aud`), and expiration (`exp`).

| Method | Endpoint | Description |
|---|---|---|
| GET | `/auth/me` | Get current authenticated user profile & claims (extracted from validated Entra ID token and synced with Cosmos DB) |

### File Management

| Method | Endpoint | Description |
|---|---|---|
| GET | `/files?projectId={id}` | List file/folder tree for a project |
| POST | `/files` | Create a new file or folder |
| GET | `/files/{fileId}` | Read file content |
| PUT | `/files/{fileId}` | Update file content |
| DELETE | `/files/{fileId}` | Delete a file or folder |
| POST | `/files/move` | Move/rename a file or folder |

### Projects

| Method | Endpoint | Description |
|---|---|---|
| GET | `/projects` | List all projects for current user |
| POST | `/projects` | Create a new project (name, language) |
| GET | `/projects/{id}` | Get project details |
| PUT | `/projects/{id}` | Update project settings |
| DELETE | `/projects/{id}` | Delete project and all its files |

### Code Execution

| Method | Endpoint | Description |
|---|---|---|
| POST | `/execute` | Submit code for execution. Body: `{ language, code, stdin? }`. Spins up an ACI container, runs the code, returns `{ output, exitCode, duration }` |
| GET | `/execute/{id}` | Get result of a previous execution |
| GET | `/execute/history?projectId={id}` | Execution history for a project |

### AI Debug Assistant

| Method | Endpoint | Description |
|---|---|---|
| POST | `/ai/debug` | Send `{ code, language, error }` → returns `{ suggestion, explanation, fixedCode? }` via Azure OpenAI |

---

 | $0 |
| Azure OpenAI Service | Pay-per-token | ~$5–10 |
| Microsoft Entra ID | Free tier (first 50,000 monthly active users free with Entra External ID) | $0 |
| Azure Container Registry | Basic | ~$5 |
| **Total** | | **~$15–30/month** |

> **Tip:** Azure for Students provides **$100 free credit** — more than enough to run this project for the full semester without spending a cent.

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | React + Monaco Editor|
| **API** | Azure Functions (C# / .NET 8 Isolated Worker, ASP.NET Core) |
| **Auth** | Microsoft Entra ID (MSAL, OAuth 2.0 / OIDC) |
| **Code Execution** | Azure Container Instances + Azure Container Registry |
| **File Storage** | Azure Blob Storage |
| **Database** | Azure Cosmos DB: NoSQL |
| **AI** | Azure OpenAI Service|
| **Monitoring** | Azure Application Insights |
| **CI/CD** | GitHub Actions |
