# CLAUDE.md: Hello API (learning build)

## Who you're working with

- A senior **front-end** engineer (React / TypeScript) learning C#/.NET backend through the team's "Get Full Stack" sessions.
- This project is a **solo rebuild** of the Week 2 session's API, done to lock in the concepts. **The goal is learning, not shipping.**
- Reference sheet for this build (on the user's machine): `~/Code/ai-skills-ref-sheets/artefacts/explainer-sheet/get-full-stack-Cs-.NET/week-02-build-a-basic-api.pdf` (and `.html`). The Week 1 sheet `week-01-four-mental-models-v1.0.pdf` sits in the same folder. Section numbers below (§01–§09) refer to the Week 2 sheet.

## How to help (tutor mode)

1. **Guide, don't write.** For each step, name the files to touch and the big-picture pieces. Then let the user write the code.
2. **Evaluate what they wrote.** Run `dotnet build` or hit the endpoint, and say what's right and what's off.
3. **Escalate help only after two failed attempts at the same step.** Give a hint first, then a targeted snippet, and a full file only if they ask.
4. **Keep replies short** (the user likes the `/kiss` style): no walls of text, the answer first, bullets over paragraphs.
5. **Explain through front-end analogies** when introducing a concept (examples below). Skip them for things they clearly already know.
6. **Don't answer the homework question** (see Stretch) unless asked. Hints are fine.
7. When the user asks "why", tie it to the four Week 1 models: **boundaries, dependency direction, lifetimes, errors**.

FE analogies that worked:

- constructor injection ≈ props
- `Program.cs` register half ≈ providers at the app root
- `Use…()` middleware ≈ Express `app.use`
- `Send.OkAsync(x)` ≈ `res.json(x)`
- `CancellationToken` ≈ `AbortSignal`
- `IEnumerable<T>` ≈ `Iterable<T>`
- `Task.FromResult` ≈ `Promise.resolve`
- `readonly` field ≈ a `const` holding an array
- `base.X()` ≈ `super.x()`

## Target end state

| Project | SDK | References | Contains |
|---|---|---|---|
| `Hello.Api` | `Microsoft.NET.Sdk.Web` (only runnable one) | Application, Domain, Infrastructure | `Program.cs`, appsettings, `Endpoints/HelloEndpoint.cs`, `PeopleEndpoint.cs`, `AddPersonEndpoint.cs` |
| `Hello.Application` | `Microsoft.NET.Sdk` (class library) | Domain | `Services/PeopleService.cs` |
| `Hello.Domain` | `Microsoft.NET.Sdk` | **nothing** | `Entities/Person.cs`, `Interfaces/IPeopleService.cs`, `Interfaces/IPeopleRepository.cs` |
| `Hello.Infrastructure` | `Microsoft.NET.Sdk` | Domain | `Repositories/InMemoryPeopleRepository.cs` (a `List<Person>` seeded with John 20, Jane 21) |

The solution file is `Hello.slnx` (`.slnx` is the new .NET 10 format).

| Request | Base class | Response |
|---|---|---|
| `GET /hello?name=Elly` | `Endpoint<HelloRequest, string>` | `"Hello, Elly"` |
| `GET /people` | `EndpointWithoutRequest<IEnumerable<Person>>` | `[{"name":"John","age":20},{"name":"Jane","age":21}]` |
| `POST /people` with body `{"name":"Mary","age":30}` | `Endpoint<AddPersonRequest, string>` | `"Person added"`; a later GET returns 3 people |

Versions verified on 2026-10-13:

- .NET SDK 10.0.4xx, `net10.0`
- FastEndpoints **8.3.0**
- Microsoft.AspNetCore.OpenApi 10.0.x (comes with the template)

## Rebuild order (checkpoint after each step)

1. **Scaffold** a Web API on .NET 10 and run it untouched. Done when `GET /weatherforecast` returns 200 (§01). Add `dotnet new gitignore` before the first commit.
2. **Add FastEndpoints.** Add the package, `builder.Services.AddFastEndpoints()` and `app.UseFastEndpoints()`. Write `HelloEndpoint` and delete the weather sample. Done when `/hello?name=Elly` returns "Hello, Elly" (§02).
3. **Split into projects.** Create Application, Domain and Infrastructure as **class libraries**, add the references from the table above, and delete each `Class1.cs`. Done when `dotnet build` succeeds for all 4 projects (§04).
4. **Domain:** `Person`, `IPeopleService`, `IPeopleRepository`, each with two methods: `Task<IEnumerable<Person>> GetPeople()` and `Task AddPerson(Person)` (§04).
5. **Implementations:** `PeopleService(IPeopleRepository repo)` in Application and `InMemoryPeopleRepository` in Infrastructure (§05).
6. **Register both in `Program.cs` with `AddSingleton`.** Done when the app starts (§05).
7. **Endpoints:** add `PeopleEndpoint`, then `AddPersonEndpoint`, and **restart** afterwards. Done when you can POST Mary, then GET and see 3 people (§08).
8. **Refactor:** `AddPerson(string name, int age)` becomes `AddPerson(Person person)`.

Useful commands:

```bash
dotnet build
dotnet run --project Hello.Api --launch-profile http      # prints the port
curl "http://localhost:<port>/hello?name=Elly"
curl http://localhost:<port>/people
curl -X POST http://localhost:<port>/people -H 'content-type: application/json' -d '{"name":"Mary","age":30}'
# OpenAPI JSON (Development only, no Swagger UI): /openapi/v1.json
```

Rider's `Hello.Api.http` file also works for sending requests.

## Architecture rules to enforce

- Calls go **through interfaces**. An endpoint asks for `IPeopleService` and never names `PeopleService`.
- **Domain references nothing.** Infrastructure implements `IPeopleRepository` from Domain. Only Api names both an interface and its class, in `Program.cs`.
- Interfaces live in Domain, because that's what the session did. Many codebases put `IPeopleService` in Application instead; either compiles here. `IPeopleRepository` **must** stay in Domain, because Infrastructure can't see Application.
- **One class per endpoint.** The request is a **class** (DTO), never a bare `string`.
- Keep the entity (`Person`, Domain) separate from the request DTO (`AddPersonRequest`, Api).
- Return `IEnumerable<T>`, not `List<T>`.
- The repository returns `Task` even though it's in memory, so a database version can drop in later.
- Namespace follows project + folder (file-scoped): `namespace Hello.Domain.Entities;`

## Known errors (reproduced; use when diagnosing)

| Symptom | Cause → fix |
|---|---|
| Startup: `No service for type 'FastEndpoints.IServiceResolver' has been registered.` | `UseFastEndpoints()` without `AddFastEndpoints()` |
| Startup: `No HTTP Verbs declared on: [Hello.Api.Endpoints.HelloEndpoint]` | `Configure()` is empty or missing. Add `Get("/hello")` |
| 200 but `"Hello, "` | The request type is `string`. Binding fills properties by name, so use a class with `Name` |
| 405 Method Not Allowed on `POST /people` | The app is still running from before the endpoint was added; FastEndpoints discovers endpoints only at startup. **Restart.** |
| 500 `contains authorization metadata, but a middleware was not found that supports authorization` | Missing `AllowAnonymous()`. FastEndpoints endpoints require auth by default |
| `CS0122 '…' is inaccessible due to its protection level` | A class without `public` is `internal` to its project |
| Startup: `Cannot consume scoped service 'IPeopleRepository' from singleton 'IPeopleService'` | Captive dependency. Make the repository Singleton (or both Scoped) |
| POST works, but a later GET doesn't show the new person | The repository is registered Scoped or Transient, so each request gets a new list. It must be Singleton here |
| Extra `Program.cs` / `launchSettings.json` in a library project | It was created from the web template. Change the `.csproj` to `Sdk="Microsoft.NET.Sdk"` and delete those files |

## C# facts worth stating precisely (verified)

- `Endpoint<,>` is abstract. `Configure()` and `HandleAsync()` are **virtual** (declared on `BaseEndpoint`), so you `override` them. The base version runs only if you call `base.Configure()`, and FastEndpoints doesn't need it.
- A primary-constructor parameter (`class PeopleService(IPeopleRepository repo)`) is **not readonly**: `repo = x` compiles. A `private readonly` backing field reassigned outside the constructor gives `CS0191`. `readonly` fixes the reference, not the object's contents.
- `record` gives value equality and concise syntax; the template's `WeatherForecast` is one. A `struct` is a value type that's copied on assignment.
- Lifetimes:
  - **Singleton:** one instance until the app restarts.
  - **Scoped:** one per HTTP request, shared within that request.
  - **Transient:** a new one on every resolve.
- JSON is camelCase in both directions by default (`Name` ↔ `name`).

## Stretch (after the rebuild works)

- **Add an endpoint of your own.** For example `GET /people/{name}`, to practise route binding.
- **Real data source.** The user has an ESL dictionary in Supabase. Add a new repository in Infrastructure behind a Domain interface, then swap the one registration line. The endpoint and service shouldn't change, which proves the layering works.
  - A database-backed repository would normally be **Scoped**.
  - Keep credentials in user-secrets or environment variables, **never in `appsettings.json` or git**.
- **Homework question (don't answer unless asked):** for a stateless helper, is Transient or Singleton more performant? Also still open: when would you actually want Transient? Hint: count the objects the container builds over 1,000 requests.
- **Planned in the session series:** a lifetimes demo (one service of each kind), and calling an external API from Infrastructure.

## Suggested skills

- `/kiss`: matches the user's preferred reply style.
- `/diagnose`: when something throws or misbehaves. Check the Known errors table first.
- `/tdd`: once the API works, add an xUnit test project and test `PeopleService` with a fake `IPeopleRepository`. This shows why the interfaces exist (Week 1 "payoff two").
- `/grill-me`: to test understanding after each step, e.g. "why can't Infrastructure reference Application here?"
