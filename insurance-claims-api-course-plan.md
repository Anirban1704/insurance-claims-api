# Insurance Claims Processing API — Build Course

A phased, from-scratch plan to build a .NET Core Web API simulating a BFSI claims workflow, deployed to Azure with Docker and GitHub Actions CI/CD. Each phase has a clear goal, exact steps, and a "you'll know it's working when..." checkpoint.

**Prerequisites to install before Phase 1:**
- Visual Studio 2022 (Community edition is free) — during install, select the **ASP.NET and web development** workload
- .NET 8 SDK (comes bundled with VS 2022, but verify: open a terminal and run `dotnet --version` — should show 8.x)
- Docker Desktop (free) — needed from Phase 5 onward
- Git (usually bundled with VS, but confirm with `git --version`)
- Azure Free Account (you said you have this already) — gives you $200 credit for 30 days plus always-free tiers on some services
- GitHub account (you have this)

---

## Phase 1 — Scaffold the Project (Day 1)

**Goal:** Get a working, empty Web API project running locally.

1. Open Visual Studio → **Create a new project**
2. Search the template list for **"ASP.NET Core Web API"** — this is Microsoft's official starter template, already wired for controllers, Swagger, and dependency injection. This is the "template" you asked about — no need to find one externally, it ships with Visual Studio.
3. Name it `InsuranceClaimsApi`. Choose a solution location you'll remember (e.g., `C:\Projects\InsuranceClaimsApi`)
4. On the next screen:
   - Framework: **.NET 8.0 (Long Term Support)**
   - Authentication type: **None** (you'll add JWT manually in Phase 3 — this teaches you the mechanics rather than hiding them)
   - Configure for HTTPS: checked
   - Enable OpenAPI (Swagger): checked
5. Click **Create**. Visual Studio generates a working project with a sample `WeatherForecastController`.
6. Press **F5** (or click the green Run arrow). A browser opens showing the Swagger UI at something like `https://localhost:7xxx/swagger`.

**Checkpoint:** You see the Swagger page with a `/WeatherForecast` endpoint you can test-click. Delete `WeatherForecastController.cs` and the `WeatherForecast.cs` model once confirmed working — you don't need them going forward.

7. Initialize Git: In Visual Studio, bottom-right corner → **Add to Source Control** → Git. Then push to a **new GitHub repository** (Visual Studio has a built-in "Publish to GitHub" flow under the Git menu, or use `git remote add origin <your-repo-url>` from a terminal if you prefer).

---

## Phase 2 — Define the Domain Model & Database (Days 2–4)

**Goal:** Model claims and policies, connect to a real database, and get Entity Framework Core migrations working.

1. In **NuGet Package Manager** (right-click project → Manage NuGet Packages), install:
   - `Microsoft.EntityFrameworkCore.SqlServer`
   - `Microsoft.EntityFrameworkCore.Design`
   - `Microsoft.EntityFrameworkCore.Tools`

2. Create a `Models` folder. Add:

```csharp
public class Policy
{
    public int Id { get; set; }
    public string PolicyNumber { get; set; }
    public string HolderName { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }
    public decimal CoverageLimit { get; set; }
    public bool IsActive { get; set; }
}

public enum ClaimStatus
{
    Submitted,
    DocumentsPending,
    UnderReview,
    Approved,
    Rejected,
    Settled
}

public class Claim
{
    public int Id { get; set; }
    public int PolicyId { get; set; }
    public Policy Policy { get; set; }
    public DateTime IncidentDate { get; set; }
    public string Description { get; set; }
    public decimal ClaimAmount { get; set; }
    public ClaimStatus Status { get; set; } = ClaimStatus.Submitted;
    public DateTime SubmittedAt { get; set; } = DateTime.UtcNow;
    public string? DocumentUrl { get; set; } // filled in once Blob Storage is wired up in Phase 6
}
```

3. Create a `Data` folder with `AppDbContext.cs`:

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    public DbSet<Policy> Policies { get; set; }
    public DbSet<Claim> Claims { get; set; }
}
```

4. **For now, develop against a local SQL Server** (LocalDB, which ships with Visual Studio) — you'll point to Azure SQL later in Phase 6. This avoids burning Azure credits while you're still writing and debugging code.

   In `appsettings.json`, add:
   ```json
   "ConnectionStrings": {
     "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=InsuranceClaimsDb;Trusted_Connection=True;"
   }
   ```

5. In `Program.cs`, register the DbContext:
   ```csharp
   builder.Services.AddDbContext<AppDbContext>(options =>
       options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
   ```

6. Open the **Package Manager Console** (Tools → NuGet Package Manager → Package Manager Console) and run:
   ```
   Add-Migration InitialCreate
   Update-Database
   ```

**Checkpoint:** Open **SQL Server Object Explorer** in Visual Studio (View menu), find your LocalDB instance, and confirm the `Policies` and `Claims` tables exist.

---

## Phase 3 — Build the Core API + Business Logic (Days 5–9)

**Goal:** Working CRUD endpoints plus the actual claims workflow rules.

1. Create a `Repositories` folder — implement a simple repository pattern (interface + implementation) for `Policy` and `Claim` data access. This matches the "layered architecture" you listed as a goal.
2. Create a `Services` folder — this is where business rules live, separate from the controller:
   - `PolicyValidationService`: checks a policy is active, not expired, and the incident date falls within the policy period
   - `ClaimEligibilityService`: checks the claim amount doesn't exceed the policy's coverage limit
   - `ClaimWorkflowService`: handles status transitions (e.g., you can't jump from `Submitted` directly to `Settled` — enforce the pipeline order)
3. Create `Controllers/ClaimsController.cs` and `Controllers/PoliciesController.cs` with standard REST endpoints:
   - `POST /api/claims` — submit a new claim (runs policy validation + eligibility checks before saving)
   - `GET /api/claims/{id}` — view a claim
   - `PUT /api/claims/{id}/status` — transition status (this is where workflow rules apply)
   - `GET /api/policies/{id}` — view policy details
4. Test everything through the Swagger UI as you build each endpoint — don't wait until the end to test.

**Checkpoint:** You can submit a claim via Swagger, see it saved with status `Submitted`, and successfully move it through the pipeline to `Approved` — but the API rejects an attempt to skip straight to `Settled` from `Submitted`.

---

## Phase 4 — JWT Authentication (Days 10–12)

**Goal:** Lock down the API so only authenticated requests can submit or modify claims.

1. Install NuGet package: `Microsoft.AspNetCore.Authentication.JwtBearer`
2. Create a simple `AuthController` with a `/api/auth/login` endpoint that accepts a username/password (hardcode a test user for now, or add a basic `Users` table) and returns a signed JWT if valid.
3. In `Program.cs`, configure JWT Bearer authentication (issuer, audience, signing key — store the key in `appsettings.json` for now, and in **User Secrets** so it never gets committed to Git: right-click project → **Manage User Secrets**).
4. Add `[Authorize]` attribute to your `ClaimsController` and `PoliciesController`.
5. In Swagger, add the "Authorize" button support so you can paste in a JWT and test protected endpoints directly from the browser.

**Checkpoint:** Calling `/api/claims` without a token returns `401 Unauthorized`. Logging in via `/api/auth/login`, copying the returned token into Swagger's Authorize dialog, and retrying the same call now succeeds.

---

## Phase 5 — Dockerize the App (Days 13–14)

**Goal:** Run the exact same API inside a container, locally, before touching Azure.

1. Right-click the project in Visual Studio → **Add** → **Docker Support**. Visual Studio auto-generates a `Dockerfile` for you — this answers your "where do I find a template" question for Docker specifically: Visual Studio's built-in tooling writes a working one, you don't need to write it from scratch.
2. Visual Studio will offer to run the project in a container directly — try this (green Run button now shows a Docker icon). Confirm Swagger still loads, now served from inside a container.
3. Build the image manually from a terminal to understand what's happening under the hood:
   ```
   docker build -t insuranceclaimsapi .
   docker run -p 8080:80 insuranceclaimsapi
   ```
4. Visit `http://localhost:8080/swagger` to confirm it works identically to running it natively.

**Checkpoint:** You can explain, in an interview, the difference between running the app with `dotnet run` versus `docker run` — and why containerizing it means "it behaves the same on my machine and in Azure."

---

## Phase 6 — Move to Azure (Days 15–19)

**Goal:** Real cloud infrastructure — Azure SQL, Blob Storage, and App Service.

1. **Azure SQL Database**: In the Azure Portal, create a new **Azure SQL Database** (choose the free/lowest-tier serverless option to conserve credits). Get its connection string from the portal (Settings → Connection Strings).
2. Update your app to use this connection string **for the Azure/production environment** — best practice is to keep LocalDB for local development and Azure SQL only for the deployed version, using ASP.NET Core's environment-based configuration (`appsettings.Production.json` or Azure App Service's Configuration blade for secrets).
3. Re-run your EF Core migrations against Azure SQL:
   ```
   dotnet ef database update --connection "<your-azure-connection-string>"
   ```
4. **Azure Blob Storage**: Create a **Storage Account** in the Azure Portal, then a container inside it (e.g., `claim-documents`). Install `Azure.Storage.Blobs` NuGet package. Add a simple upload endpoint on your `ClaimsController` (`POST /api/claims/{id}/documents`) that uploads a file to this container and saves the resulting URL into your `Claim.DocumentUrl` field.
5. **Azure App Service**: Create a new **App Service** (Linux, Docker container option) in the portal. This is where your containerized API will actually run.

**Checkpoint:** You can upload a mock document through your API and see the file appear in your Azure Storage container via the Azure Portal's Storage Explorer.

---

## Phase 7 — CI/CD with GitHub Actions (Days 20–22)

**Goal:** Every push to `main` automatically builds, tests, and deploys to Azure.

1. In your GitHub repo, go to **Actions** tab → GitHub will suggest workflow templates based on your repo content; since you have a Dockerfile, it'll suggest a **Docker** workflow — this is the "template" for your CI/CD pipeline, and you should start from it rather than writing YAML from scratch.
2. Customize the suggested `.github/workflows/main.yml` to:
   - Build the Docker image
   - Push it to **Azure Container Registry** (create one in the Azure Portal first) or directly to your App Service
   - Deploy to Azure App Service using the `azure/webapps-deploy@v2` action
3. In Azure Portal → your App Service → **Deployment Center**, get the **Publish Profile**, and add it as a GitHub **repository secret** (Settings → Secrets and variables → Actions → New repository secret). Reference this secret in your workflow file — never hardcode credentials directly in the YAML.
4. Push a small change to `main` and watch the Actions tab run the pipeline live.

**Checkpoint:** A code push triggers a visible pipeline run in the Actions tab, ending in a green checkmark, and refreshing your Azure App Service URL shows the updated API live.

---

## Phase 8 — Polish & Optional React Frontend (Days 23–28, if time allows)

**Goal:** Round out the project into something demo-able end-to-end.

1. Add a `README.md` to your repo explaining the architecture, tech stack, and how to run it — this is often the first thing a reviewer actually opens.
2. If time allows, scaffold a small React app (`npx create-react-app claims-frontend` or Vite for a lighter setup) with two screens: submit a claim, and view claim status. Point it at your deployed Azure API.
3. Record a short screen-capture demo (even 60–90 seconds) showing a claim being submitted and moving through the status pipeline — genuinely useful to have ready to share, separate from the CV itself.

---

## Weekly Pace Summary

| Week | Focus |
|---|---|
| Week 1 | Phases 1–3: scaffold, database, core API + business logic |
| Week 2 | Phases 4–5: JWT auth, Docker |
| Week 3 | Phases 6–7: Azure infrastructure, CI/CD |
| Week 4 | Phase 8: polish, optional frontend, documentation |

Work through each checkpoint before moving to the next phase — skipping ahead when something's half-working is where most personal projects stall out.
