# Provider Registry

A small provider and license registry built with ASP.NET Core Web API, Entity Framework Core, SQLite, React, and Vite. Provider and license records are loaded from the API; the frontend contains no business sample data.

## Requirements

- .NET 8 SDK
- Node.js and npm

## Run locally

Start the API from the repository root:

```powershell
dotnet restore CandidateAssessment.sln
dotnet run --no-launch-profile --urls http://localhost:5000 --project backend\Candidate.API
```

In another terminal, start the React app:

```powershell
cd frontend\React.UI
npm ci
npm run dev
```

Open the address printed by Vite (normally `http://localhost:5173`). Vite forwards `/api` requests to `http://localhost:5000`; keep the API on that address while using the development server. Swagger is available at `http://localhost:5000/swagger` in the Development environment.

Build the solution/API with `dotnet build CandidateAssessment.sln`. Build the React production bundle separately with `npm run build` from `frontend\React.UI`.

## Data model and database

The SQLite database is [providers.db](backend/Candidate.API/Database/Data/providers.db). Entity Framework Core creates the schema on API startup and stores the file in `backend/Candidate.API/Database/Data/`. The provider table includes a GUID key, name, county, lifecycle status, creation/update timestamps, and soft-delete fields. Each license has a required provider foreign key, license number, stored status, expiration date, and creation timestamp. A unique index prevents duplicate license numbers for the same provider. Provider deletion does not cascade to licenses.

The API uses a global EF query filter to exclude deleted providers from normal queries. A corresponding license filter excludes licenses owned by deleted providers. Deletion sets `IsDeleted`, `DeletedAtUtc`, and `UpdatedAtUtc`; it never removes the provider or associated licenses. The `GET /api/providers/deleted` endpoint intentionally bypasses those filters for audit and maintenance queries. It is included for this assignment and should be protected by authorization in a deployed system.

The database is seeded only when it contains no providers. Its examples demonstrate an active provider with a valid active license, an active provider with an expired license, an inactive provider with a valid license, a suspended license, and a soft-deleted provider whose data remains available through the audit endpoint. License validity is calculated from `ExpirationDate` using the current UTC date; a suspended license remains suspended even when its expiration date has passed. Provider status and license validity are evaluated separately.

`EnsureCreated` is used to make the sample database easy to run without a migration step. It does not update an existing schema; a production system should replace it with versioned EF Core migrations. The schema is defined in the EF model, so no standalone SQL scripts are currently required. See [Database/README.md](backend/Candidate.API/Database/README.md) for the database artifact location.

## API

| Method | Route | Description |
| --- | --- | --- |
| `GET` | `/api/providers` | List non-deleted providers |
| `GET` | `/api/providers?search=Fulton` | Search provider names and counties |
| `GET` | `/api/providers?scenario=active-with-active-licenses` | Active providers with at least one currently active license |
| `GET` | `/api/providers?scenario=active-with-expired-licenses` | Active providers with at least one expired license |
| `GET` | `/api/providers/{id}` | Provider detail and associated licenses |
| `GET` | `/api/providers/deleted` | Soft-deleted providers for audit |
| `POST` | `/api/providers` | Create a provider |
| `PUT` | `/api/providers/{id}` | Update a provider |
| `DELETE` | `/api/providers/{id}` | Soft-delete a provider |

Provider status must be `Active`, `Inactive`, or `Pending`. API validation enforces required names and counties, field lengths, and allowed statuses. A license's effective status is returned as `Active`, `Expired`, or `Suspended`; expiration is derived by the API rather than inferred from provider status.

## React workflows

The directory loads providers from `GET /api/providers`, supports name/county search and the two license scenarios, and provides create, edit, and confirmed soft-delete workflows. Selecting a provider loads its license details from the API. Loading, empty, validation, and request-error states are shown in the UI. The Vite development proxy keeps browser requests on `/api`; a deployment should route that path to the API or configure an equivalent same-origin reverse proxy.

## Assumptions and trade-offs

- Providers can have multiple licenses, and license records are read-only in this assignment because only provider CRUD and license viewing are required.
- License stored status is `Active` or `Suspended`; `Expired` is a derived status based on the expiration date.
- The sample audit endpoint is unauthenticated to keep the local exercise easy to inspect. Production access must be restricted.
- There is no pagination or concurrency token because the included dataset and workflows are intentionally small.
- The database file is included as a deliverable. The API seeds a new/empty database; it will not overwrite existing data.

## Further improvements

With more time, I would add EF Core migrations, API and UI automated tests (including persistence tests for soft deletion), authentication and role-based audit access, pagination, and operational logging/health checks.
