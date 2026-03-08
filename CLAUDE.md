# CLAUDE.md — Azure DevOps Demo Generator

This file provides guidance for AI assistants working on this codebase.

## What This Application Does

The **Azure DevOps Demo Generator** is a template-based project provisioning tool that:

1. Authenticates users via OAuth 2.0 with Azure DevOps
2. Presents a library of 79+ pre-built project templates (SmartHotel360, PartsUnlimited, eShopOnWeb, etc.)
3. Creates fully configured Azure DevOps projects from those templates, including:
   - Work items (epics, features, bugs, tasks)
   - Build and release pipelines
   - Git repositories with sample code
   - Teams, iterations, area paths, queries
   - Dashboards, test plans, service endpoints
4. Extracts existing Azure DevOps projects into reusable templates
5. Exposes REST APIs for programmatic project creation

## Repository Layout

```
/
├── src/
│   ├── VstsDemoBuilder/          # Main ASP.NET MVC 5 web application
│   │   ├── Controllers/          # MVC and Web API controllers
│   │   ├── Services/             # Business logic layer
│   │   ├── ServiceInterfaces/    # Service contracts (IProjectService, etc.)
│   │   ├── Models/               # View models and DTOs
│   │   ├── Views/                # Razor (.cshtml) views
│   │   ├── Templates/            # JSON project templates (79+ folders)
│   │   ├── App_Start/            # Application startup configuration
│   │   ├── Content/              # Static CSS, images
│   │   ├── Scripts/              # JavaScript (jQuery, Knockout.js)
│   │   ├── Web.config            # Main application configuration
│   │   └── packages.config       # NuGet package list
│   └── VstsRestAPI/              # Reusable Azure DevOps REST API wrapper library
│       ├── Build/                # Build definitions
│       ├── Release/              # Release definitions
│       ├── Git/                  # Git repositories and PRs
│       ├── ProjectsAndTeams/     # Projects, teams, area/iteration paths
│       ├── WorkItemAndTracking/  # Work items, boards, backlogs
│       ├── QueriesAndWidgets/    # Queries and dashboards
│       ├── Wiki/                 # Wiki pages
│       ├── TestManagement/       # Test plans and cases
│       ├── DeliveryPlans/        # Delivery plans
│       ├── DeploymentGroup/      # Deployment groups
│       ├── ExtensionManagement/  # Marketplace extensions
│       ├── Viewmodel/            # Typed response models (17 sub-folders)
│       └── JSON/                 # JSON template files for the library
├── docs/                         # Markdown documentation (Jekyll site)
├── .github/workflows/            # GitHub Actions (CodeQL, ADO sync)
├── Images/                       # Documentation images
└── Report/                       # Doxygen-generated API reference HTML
```

## Technology Stack

- **Language**: C# targeting .NET Framework 4.7
- **Web Framework**: ASP.NET MVC 5.2.7 + ASP.NET Web API 5.2.3
- **View Engine**: Razor (`.cshtml`)
- **Frontend**: jQuery 1.12.4, Knockout.js 2.2.0, Bootstrap
- **Dependency Injection**: Unity container (`UnityConfig.cs`)
- **Logging**: log4net (rolling file appender)
- **Auth**: OAuth 2.0 via `Microsoft.IdentityModel.Clients.ActiveDirectory`
- **Azure DevOps SDK**: `Microsoft.VisualStudio.Services.Client` v15.112.1
- **Serialization**: Newtonsoft.Json 12.0.3
- **Feature Flags**: LaunchDarkly 5.4.0
- **Analytics**: Google Analytics Tracker 5.0.0
- **API Docs**: Swashbuckle/Swagger 5.6.0
- **ORM**: EntityFramework 6.2.0 (referenced but application is largely stateless)
- **Build**: NuGet + MSBuild

## Key Source Files

| File | Size | Purpose |
|------|------|---------|
| `src/VstsDemoBuilder/Services/ProjectService.cs` | ~3,100 lines | Core orchestration of project creation; the heart of the application |
| `src/VstsDemoBuilder/Services/ExtractorService.cs` | ~1,640 lines | Exports existing ADO projects as reusable templates |
| `src/VstsDemoBuilder/Controllers/EnvironmentController.cs` | ~46 KB | Web UI controller managing the creation workflow |
| `src/VstsDemoBuilder/Controllers/Apis/ProjectController.cs` | ~20 KB | REST API endpoints for project creation |
| `src/VstsDemoBuilder/Controllers/ExtractorController.cs` | ~27 KB | Controller for template extraction workflow |
| `src/VstsDemoBuilder/Web.config` | ~130+ lines | All application settings, API versions, auth config |
| `src/VstsRestAPI/ApiServiceBase.cs` | 63 lines | Base class for all Azure DevOps API operations |

## Architecture and Patterns

### Layered Architecture
```
Controllers (HTTP layer)
    ↓
Service Interfaces (IProjectService, IAccountService, etc.)
    ↓
Services (business logic — ProjectService, ExtractorService, etc.)
    ↓
VstsRestAPI (Azure DevOps REST API wrapper)
    ↓
Azure DevOps REST APIs
```

### Progress Tracking Pattern
Project creation is asynchronous. `ProjectService` writes status updates into a static `StatusMessages` dictionary keyed by a session/project ID. The UI polls for these updates. All writes use `lock(ProjectService.objLock)` for thread safety.

### Template System
Each template lives in `src/VstsDemoBuilder/Templates/<TemplateName>/` and contains JSON files describing every Azure DevOps artefact to create (work items, pipelines, repositories, dashboards, etc.). The `ProjectService` reads these files and makes the corresponding REST API calls.

### Dependency Injection
Unity is configured in `App_Start/UnityConfig.cs`. All service dependencies are registered there. Always inject services through their interfaces (`IProjectService`, not `ProjectService`).

## Naming Conventions

| Artifact | Convention | Example |
|----------|-----------|---------|
| Controllers | PascalCase + `Controller` suffix | `AccountController` |
| Services | PascalCase + `Service` suffix | `ProjectService` |
| Interfaces | `I` prefix + PascalCase | `IProjectService` |
| Models / DTOs | PascalCase singular | `Project`, `AccessDetails` |
| Properties | PascalCase | `ProjectId`, `IsAuthenticated` |
| Methods | PascalCase | `CreateProject`, `GetStatusMessage` |
| Namespaces | `VstsDemoBuilder.<Layer>` / `VstsRestAPI.<Feature>` | `VstsDemoBuilder.Services` |

## Development Workflow

### Prerequisites
- Windows with IIS (or IIS Express)
- Visual Studio 2017+ (or MSBuild)
- .NET Framework 4.7 SDK
- NuGet CLI
- An Azure DevOps organization and an OAuth application registration

### Local Setup
See `docs/Local-Development.md` for the full walkthrough, including IIS site setup, SSL certificate, and OAuth app registration. Key config values that must be set in `Web.config`:

```xml
<appSettings>
  <add key="ClientId" value="..." />          <!-- OAuth app client ID -->
  <add key="ClientSecret" value="..." />      <!-- OAuth app client secret -->
  <add key="RedirectUri" value="..." />       <!-- OAuth callback URL -->
  <add key="AzureDevOpsURL" value="https://dev.azure.com/" />
</appSettings>
```

### Build
```bash
nuget restore VSTSDemoGeneratorV2.sln
msbuild VSTSDemoGeneratorV2.sln /p:Configuration=Release
```

### Running Locally
The project requires IIS (not IIS Express by default) due to OAuth redirect URIs. Follow `docs/Local-Development.md` for the IIS site creation steps.

### No Automated Tests
There is currently no unit or integration test project in the solution. Validation is done manually or through the CodeQL GitHub Actions workflow.

## CI / CD

| Workflow | File | Trigger |
|----------|------|---------|
| CodeQL security scan | `.github/workflows/codeql.yml` | Manual (`workflow_dispatch`) |
| GitHub → ADO issue sync | `.github/workflows/ado-gh-sync.yml` | Issue events |

The CodeQL workflow builds the solution with:
```bash
nuget restore && msbuild VSTSDemoGeneratorV2.sln
```

## Adding or Modifying Templates

1. Create a new folder under `src/VstsDemoBuilder/Templates/<TemplateName>/`
2. Add the required JSON files (work items, build definitions, release definitions, etc.)
3. Register the template in the template index/list used by `TemplateService`
4. Test by selecting your template in the UI and creating a project

Refer to an existing template like `Gen-SmartHotel360` as the canonical example.

## Adding a New Azure DevOps API Operation

1. Create a new class (or add a method) in the appropriate `VstsRestAPI/<Feature>/` folder
2. Inherit from `ApiServiceBase` (provides `ExecuteGetRequest`, `ExecutePostRequest`, etc.)
3. Use `Configuration` for base URL and API version values
4. Add a strongly-typed ViewModel in `VstsRestAPI/Viewmodel/<Feature>/`
5. Call the new API class from `ProjectService` or `ExtractorService`

## Configuration Reference (`Web.config`)

Key `<appSettings>` entries:

| Key | Purpose |
|-----|---------|
| `ClientId` / `ClientSecret` | OAuth app credentials |
| `RedirectUri` | OAuth callback |
| `AzureDevOpsURL` | Base URL (`https://dev.azure.com/`) |
| `DefaultTemplate` | Template shown first in UI (SmartHotel360) |
| `ApiVersion` | Default ADO REST API version (4.1) |
| `ApiVersion_5` | Newer endpoint version (5.0) |
| `GoogleAnalyticsKey` | GA tracking ID |
| `LaunchDarklyKey` | Feature flag SDK key |

Session is in-process (`InProc`) with a 30-minute timeout. HTTPS is required for auth cookies.

## Security Notes

- Do **not** commit real `ClientId`, `ClientSecret`, or API keys to source control
- The application requires HTTPS in production (`requireSSL="true"` on cookies)
- OAuth tokens are stored in session, not persisted to a database
- See `SECURITY.md` for the vulnerability disclosure policy

## Documentation

| Document | Location |
|----------|----------|
| Local dev setup | `docs/Local-Development.md` |
| User guide | `docs/Using-The-Generator.md` |
| Template extractor guide | `docs/Using-The-Template-Extractor.md` |
| REST API reference | `docs/Azure-DevOps-Demo-Generator-REST-API-Reference.md` |
| FAQ | `docs/FAQ.md` |
| Limitations | `docs/Limitations.md` |
| Private templates | `docs/Using-Private-template-URL.md` |
| Contribution guide | `CONTRIBUTING.md` |
| Doxygen API docs | `Report/` (HTML) |
