# ResolveAI

ResolveAI is an AI-assisted IT incident management system with a Java 21/JavaFX desktop client and a separate Spring Boot REST backend. The backend owns MySQL 8 persistence, SQL operations, transactions, and server-side role checks. The app supports ticket creation, search, filters, assignment, workflow updates, editing, administrator deletion, comments, history, SLAs, and optional AI suggestions with human approval.

For copyable setup steps and a walkthrough of the main workflows, see the [step-by-step guide](docs/STEP_BY_STEP.md).

## Prerequisites

- JDK 21
- Maven 3.9 or newer
- MySQL 8.0 or newer

Confirm Java and Maven are available in PowerShell:

```powershell
java -version
mvn -version
```

## Quick Start

1. Create a MySQL database and application user using the SQL in [docs/STEP_BY_STEP.md](docs/STEP_BY_STEP.md).
2. Start the backend from `backend/` in one PowerShell terminal. Set DB credentials in that terminal:

```powershell
$env:RESOLVEAI_DB_URL = 'jdbc:mysql://localhost:3306/resolveai?serverTimezone=UTC&allowPublicKeyRetrieval=true'
$env:RESOLVEAI_DB_USER = 'resolveai_app'
$env:RESOLVEAI_DB_PASSWORD = 'your-local-password'
& (Join-Path $env:TEMP 'apache-maven-3.9.9\bin\mvn.cmd') spring-boot:run
```

3. In a second terminal at the repository root, run the desktop client:

```powershell
$env:RESOLVEAI_API_BASE_URL = 'http://localhost:8080/api'
& (Join-Path $env:TEMP 'apache-maven-3.9.9\bin\mvn.cmd') javafx:run
```

Install Maven normally to use `mvn` instead of the temporary path. On first backend startup, MySQL tables and idempotent demo rows are initialized from the backend's `src/main/resources/db` scripts. The desktop talks to the backend over HTTP and has no database credentials.

Run the projects' tests separately:

```powershell
# From repository root: desktop client
mvn clean verify

Push-Location backend
mvn clean verify
Pop-Location
```

## Optional AI Provider

Without an API key, the app uses a local heuristic classifier. To enable the OpenAI-compatible chat completions provider, also configure `RESOLVEAI_AI_API_KEY`; `RESOLVEAI_AI_ENDPOINT` and `RESOLVEAI_AI_MODEL` are optional overrides. The key is read from the environment only. AI output is validated and stored as unapproved. A support user must explicitly approve a suggestion before supported fields can be applied; troubleshooting text is never executed.

## Roles And Demo Data

The seeded identities and the in-app user selector exist to demonstrate role-based checks, not to authenticate a production user. Requesters are limited to their own incidents. Agents can work on incidents assigned to them. Managers and administrators can assign incidents and perform management actions. Replace the demo selector with a real authentication provider before production use.

The seeded records are examples. Do not use the demo database as a production dataset.

## Backend And Security Notes

- `backend/` is a standalone Spring Boot project and owns MySQL credentials, schema initialization, JDBC operations, and transactions.
- The JavaFX client connects through repository adapters in `src/main/java/com/resolveai/api/` and `RESOLVEAI_API_BASE_URL`.
- Backend endpoint root: `http://localhost:8080/api`; health check: `/health`.
- Incident writes and corresponding history events are transactional and use prepared SQL parameters.
- Desktop HTTP/database-bound operations run on a background executor.
- The `X-ResolveAI-User-Id` header currently carries a seeded demo user's ID for role demonstration. It can be forged and is not authentication. Use a real identity provider/JWT principal before exposing the backend beyond a trusted local demo.
- Both response and resolution target times are stored. The current breach indicator evaluates the resolution deadline while the incident remains unresolved; it does not pause clocks for waiting status.
- Selecting an incident displays all persisted incident fields, including reporter, owner/team, lifecycle timestamps, SLA targets, comments, and audit history.
- Incident create, edit, assignment, status changes, and comments are written to MySQL. Permanent incident deletion is administrator-only and removes the incident's comments and history through foreign-key cascades; linked AI suggestions are retained with their incident reference cleared.

## Project Layout

```text
src/main/java/com/resolveai/
	api/          HTTP client and REST repository adapters
	ai/           External/local AI providers and suggestion approval
	exception/    Validation, authorization, not-found, and data errors
	model/        Incident, user, workflow, AI, and SLA types
	repository/   Repository contracts used by the service layer
	security/     Role-based access checks
	service/      Validation, lifecycle rules, and prioritization queue
	ui/           Shared JavaFX display formatting
	MainApp.java  Desktop application and view composition
src/main/resources/
	db/           Reference MySQL schema and demo seed
	styles.css    JavaFX styling
backend/
	src/main/java/com/resolveai/backend/
	  api/        REST controllers and request/response DTOs
	  service/    Role checks, SQL persistence, and transactions
	src/main/resources/db/  Backend schema and demo seed scripts
src/test/java/  Desktop service and AI workflow tests
```