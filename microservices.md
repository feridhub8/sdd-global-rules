# Microservices

Applies to projects built as microservices on ASP.NET Core Web API.

## Decided per project

Decide these with me for each project and record the decision in the project's `openspec/config.yaml`:

- .NET version (target framework).
- Repository layout: one repo per service or a monorepo.
- Database: one per service, shared, or mixed; and the database engine.
- Whether development and production databases share one server or run on separate servers.
- API gateway / reverse proxy.
- Communication between services: synchronous protocol and/or message broker.
- Mapping library.
- Infrastructure tools.

## Repositories

- Each microservice is `ms-<name>`, with `<name>` in lowercase kebab-case. Never `<name>-service`.
- `api-gateway` is the single entry point for clients. It follows the service root layout with `<Name>` `ApiGateway`, its `WebApi/` has no `Data/`, and it counts as a service wherever these rules list services (solution, root compose files, `.vscode`).
- `infrastructure` holds the Docker setup of infrastructure tools and nothing else.

## Infrastructure

- One folder per tool, containing only `docker-compose.yml`, `.env.example`, and config files the tool needs.
- A root `docker-compose.yml` includes every tool, so all tools start and stop with docker compose.
- When I ask to add a tool, add only its Docker setup and include it in the root `docker-compose.yml`.
- No OS-specific scripts. Everything runs with docker compose commands, which work the same on every OS.

## Service root

```
ms-<name>/
├── src/
│   └── WebApi/
├── .vscode/                 # one repo per service only
├── .dockerignore            # one repo per service only
├── .editorconfig            # one repo per service only
├── .env.example
├── .gitignore               # one repo per service only
├── <Name>.<sln|slnx>
├── Dockerfile
├── docker-compose.yml
├── docker-compose.dev.yml
└── README.md
```

- `<Name>` is the service name in PascalCase without the `ms-` prefix. Solutions are `.slnx` on .NET 10 and later, `.sln` on older versions.
- The project is always `src/WebApi/WebApi.csproj` with root namespace `WebApi`.
- `.editorconfig`, `.gitignore`, and `.vscode/` (`launch.json`, `settings.json`, `tasks.json`) live in the service root when each service has its own repo, and only in the repository root in a monorepo. In a monorepo, `.vscode/` contains:
  - `settings.json`: `dotnet.defaultSolution` points to the root solution.
  - `tasks.json`: one `build ms-<name>` task per service that builds its `WebApi.csproj`.
  - `launch.json`: one `ms-<name> (Local)` configuration per service that reads port and environment from the service's `launchSettings.json` (`launchSettingsFilePath` + `launchSettingsProfile`) instead of repeating them, and an `All services (Local)` compound with every service and `stopAll: true`.
- `.env.example` is committed; `.env` is not.
- The internal port is `8080`, and every container joins the shared external network `<project>_network`.

## Docker

- In a monorepo, images are built with the repository root as context (`docker build -f services/ms-<name>/Dockerfile .`), so shared projects outside the service folder can be copied in. Service compose files point `build.context` to the repository root.
- `.dockerignore` lives in the service root when each service has its own repo, and only in the repository root in a monorepo. A service that needs different rules adds `services/ms-<name>/Dockerfile.dockerignore`, which replaces the root file entirely for that build.
- In a monorepo, the repository root has `docker-compose.yml` (Production) and `docker-compose.dev.yml` (Development) that list every service compose file under `include:`.
- Every compose file sets `name: <project>`, so the root and service files form one Compose project.
- Compose files set no `container_name`. Compose names containers `<project>-ms-<name>-<n>`, so a service can run several replicas.
- Before running Docker commands, the assistant checks that `<project>_network` exists and creates it when it is missing. The root `README.md` states that the network is required on every machine that runs the containers and gives `docker network create <project>_network`.
- Host ports: the first digit is the environment (`Local` `5xxx`, `Development` `6xxx`, `Production` `7xxx`) and the last digits are the service number, the same in every environment. `api-gateway` has service number `80`. `Local` sets each service's port in `launchSettings.json`, so services are reachable directly there. In `Development` and `Production` only `api-gateway` publishes a host port, mapped to `8080` in its compose files; microservices publish none and are reached through the gateway.

## Environments

- There are three environments (`ASPNETCORE_ENVIRONMENT`): `Local`, `Development`, `Production`.
- `Local` runs on the developer machine and uses the development database.
- `Development` runs from `docker-compose.dev.yml` and uses the development database.
- `Production` runs from `docker-compose.yml` and uses the production database.
- When development and production share one server, development resources on it end with `Dev`: database names, and per-environment resources inside a shared tool instance, such as identity provider realms. Production names have no suffix. SQL scripts quote these names so the suffix keeps its case.
- `appsettings.json` holds the base settings; `appsettings.Local.json` and `appsettings.Development.json` override them for their environment.
- Connection strings are secrets. The whole connection string lives in the secrets store as `ConnectionStrings:Default`, one entry per environment; `appsettings*.json` never contains one, not even without a password. An environment that reaches a resource by a different host (`Local` on the host machine, `Development` in Docker) has its own secret path.
- Launch profiles in `launchSettings.json` and `.vscode/launch.json` run in `Local` and never open a browser (`launchBrowser: false`).

## WebApi layout

```
WebApi/
├── Data/
│   ├── <Feature>/
│   │   ├── Commands/V1/
│   │   ├── Consumers/V1/
│   │   ├── Controllers/V1/
│   │   ├── Dtos/V1/
│   │   ├── Entities/V1/
│   │   ├── Enums/V1/
│   │   ├── Handlers/V1/
│   │   ├── Mappings/V1/
│   │   ├── Models/V1/
│   │   ├── Queries/V1/
│   │   ├── Repositories/V1/Base/
│   │   ├── Services/V1/Base/
│   │   └── Validators/V1/
│   └── Shared/
├── Exceptions/V1/
├── Extensions/V1/
├── Messaging/
│   ├── Events/V1/
│   └── Interfaces/
├── Middlewares/V1/
├── Persistence/
│   ├── Configurations/
│   └── Migrations/
├── Properties/launchSettings.json
├── appsettings.json
├── appsettings.Development.json
├── appsettings.Local.json
├── Program.cs
└── WebApi.csproj
```

- `<Feature>` is one feature or aggregate, in PascalCase singular.
- `Shared/` holds code used by more than one feature and follows the same sub-layout as a feature. The base entity lives in `Shared/Entities/V1/Base/`.
- `Models/` holds types that are not persisted: options, external API models, internal results.
- Create only the folders a service or feature needs. `Persistence/` exists only with a database, `Messaging/` only with a message broker.
- Feature sub-folders are versioned (`V1`, `V2`, ...). A new API version gets a new folder; the old one stays untouched.
- Interfaces and base classes live in a `Base/` folder next to their implementations.

## File naming

| Kind | Format |
|---|---|
| Controller | `<Entity>Controller` |
| Command | `<Action><Entity>Command` |
| Query (list) | `GetAll<Entities>Query` |
| Query (single) | `Get<Entity>By<Key>Query` |
| Handler | name of its command/query with `Handler` instead of `Command`/`Query` |
| Validator | name of its command/query with `Validator` instead of `Command`/`Query` |
| Request DTO | `<Action><Entity>Dto` |
| Response DTO | `Get<Entity>OkResponseDto` |
| Mapping | `<Source>_<Destination>` |
| Repository interface | `I<Entity>Repository` |
| Repository implementation | `<Entity><DataAccessTech>Repository` |
| Service | `I<Entity>Service` + `<Entity>Service` |
| Event | `<Subject><PastTenseVerb>Event` |
| Consumer | event name with `Consumer` instead of `Event` |
| Message publisher | `IMessagePublisher` + `<Broker>Publisher` |
| EF configuration | `<Entity>Configurations` |
| DbContext | `ApplicationDbContext`, with `ApplicationDbContextFactory` for design time |
| Exception | `<Reason>Exception` |
| Extensions class | `<Area>Extensions` |
| Middleware | `<Purpose>Middleware` |
| Global exception handler | `GlobalExceptionHandler` |

- Anything that also returns soft-deleted records gets a `WithDeleted` suffix on the query, handler, validator, and response DTO.
- File name equals the type name.

## Request flow

- Controller → `mediator.Send(command/query)` → Handler → `validator.ValidateAndThrowAsync` → mapping → Service → Repository.
- The mediator is MediatR 12.5.0, the last free (Apache-2.0) release. Never upgrade to 13 or later; those require a commercial license.
- Commands and queries are `sealed record`s implementing `IRequest` / `IRequest<T>` that wrap the request DTO.
- Validators validate the command/query, not the DTO.
- One mapping per file in `Mappings/V1`, written with the project's mapping library.
- Errors are thrown as exceptions from `Exceptions/V1` and turned into HTTP responses by `GlobalExceptionHandler` in `Middlewares/V1`.

## Controllers

- `[ApiController]`, `[ApiVersion]`, route `api/v{version:apiVersion}/[controller]`.
- Actions: `POST Create`, `PUT Update/{id}`, `DELETE Delete/{id}`, `PATCH Restore/{id}`, `GET {id}`, `GET` for the list, plus `WithDeleted` variants of the two `GET`s.
- Deletes are soft deletes; `Restore` brings a record back.
- Write actions and `WithDeleted` actions require the admin role.
- Every action declares `[ProducesResponseType]` and takes `CancellationToken ct`.
- Actions that return a body return `ActionResult<T>`; actions without a body return `IActionResult`.

## Code style

- File-scoped namespaces, enforced in `.editorconfig` with `csharp_style_namespace_declarations = file_scoped`; the namespace matches the folder path.
- Constructor injection into `private readonly` fields without an underscore prefix, assigned with `this.`.
- Async methods take `CancellationToken ct` (`= default` on interfaces).
- `.csproj` enables `Nullable`, `ImplicitUsings`, `TreatWarningsAsErrors`.

## Program.cs

- Stays short. If the project uses a secrets store, load secrets into configuration first: with Vault, `VaultExtensions.InitVaultAsync`.
- Register everything through `Init<Area>()` extension methods from `Extensions/V1`, one class per area.
- `LibraryExtensions.InitLibraries` registers third-party libraries that need only library-level setup. Anything with its own configuration gets its own `<Area>Extensions` class.
- `DatabaseExtensions.InitDatabase` registers the `ApplicationDbContext`.
- `RepositoryExtensions.InitRepositories` registers every repository and `ServiceExtensions.InitServices` every service; `Program.cs` calls both. Repositories and services are not registered anywhere else, such as in a per-feature or per-area class.
- Swagger only in `Local` and `Development`.
- Health check at `/health`.

## Boundaries

- A service never imports another service's code. Services communicate only through the method chosen for the project.
- Start small: build the minimum working set of services first, add the rest one proposal at a time.
