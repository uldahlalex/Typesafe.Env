# Typesafe.Env

A C# source generator that turns the keys in your `.env` into a typed `Env` class, so configuration is used as
`env.PGHOST` instead of `configuration["PGHOST"]`:

- adding a key to `.env` makes it available in code right away,
- removing or renaming a key breaks the build everywhere it is used,
- creating the `Env` at startup stops the app if a required value is missing or a number isn't a number.

The compiler can only check key *names*, against your development `.env`. *Values* exist only at runtime: in
production they come from real environment variables, which the build never sees. The `Env` constructor checks them, so
create the `Env` at startup and a misconfigured deploy fails right away, listing every problem, instead of on first use.

The development value in `.env` decides the type:

| `.env` line | Generated property |
|---|---|
| `PGPORT=5432` (digits only) | `public int PGPORT` (required) |
| `PGHOST=localhost` (anything else) | `public string PGHOST` (required) |
| `SENTRY_DSN=` (empty) | `public string? SENTRY_DSN` (optional) |
| `Jwt__Secret=...` | `public string Jwt__Secret`, read from configuration key `Jwt:Secret` |

Keys must be valid C# identifiers (letters, digits, `_`), or the build fails with `TSENV001`.

There is no NuGet package. The generator is a single DLL attached to each
[GitHub release](https://github.com/uldahlalex/Typesafe.Env/releases), and consuming projects download it at build time.

## Using it in a project

### 1. Add this to the app's `.csproj`

```xml
<!-- Typesafe.Env: generates a typed Env class from .env (https://github.com/uldahlalex/Typesafe.Env) -->
<PropertyGroup>
  <!-- Empty: always the latest release. A tag like v1.0.3: pinned to that release. -->
  <TypesafeEnvVersion></TypesafeEnvVersion>
  <TypesafeEnvRepo>https://github.com/uldahlalex/Typesafe.Env</TypesafeEnvRepo>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="DotNetEnv" Version="3.2.0" />
  <CompilerVisibleProperty Include="RootNamespace" />
  <AdditionalFiles Include="$(MSBuildProjectDirectory)/.env" Condition="Exists('$(MSBuildProjectDirectory)/.env')" />
  <AdditionalFiles Include="$(MSBuildProjectDirectory)/../.env" Condition="Exists('$(MSBuildProjectDirectory)/../.env')" />
</ItemGroup>
<Target Name="TypesafeEnv" BeforeTargets="CoreCompile">
  <PropertyGroup>
    <_TypesafeEnvLatest>$([System.String]::IsNullOrEmpty('$(TypesafeEnvVersion)'))</_TypesafeEnvLatest>
    <_TypesafeEnvDir>$(BaseIntermediateOutputPath)typesafe-env/$(TypesafeEnvVersion)/</_TypesafeEnvDir>
    <_TypesafeEnvDir Condition="$(_TypesafeEnvLatest)">$(BaseIntermediateOutputPath)typesafe-env/latest/</_TypesafeEnvDir>
    <_TypesafeEnvUrl>$(TypesafeEnvRepo)/releases/download/$(TypesafeEnvVersion)/Typesafe.Env.Generator.dll</_TypesafeEnvUrl>
    <_TypesafeEnvUrl Condition="$(_TypesafeEnvLatest)">$(TypesafeEnvRepo)/releases/latest/download/Typesafe.Env.Generator.dll</_TypesafeEnvUrl>
    <_TypesafeEnvCached>$([System.IO.File]::Exists('$(_TypesafeEnvDir)Typesafe.Env.Generator.dll'))</_TypesafeEnvCached>
  </PropertyGroup>
  <!-- Pinned: download once. Latest: check every build (only re-downloads when it changed), and fall back to the
       cached copy with a warning when offline. -->
  <DownloadFile Condition="$(_TypesafeEnvLatest) Or !$(_TypesafeEnvCached)"
                SourceUrl="$(_TypesafeEnvUrl)" DestinationFolder="$(_TypesafeEnvDir)"
                ContinueOnError="$(_TypesafeEnvCached)" />
  <ItemGroup>
    <Analyzer Include="$(_TypesafeEnvDir)Typesafe.Env.Generator.dll" />
  </ItemGroup>
</Target>
```

What each part does:

- `TypesafeEnvVersion` picks the generator version:
  - **Empty (the default): latest release.** Every build checks GitHub and downloads only when there is a newer
    release. Offline, the build uses the cached copy and shows a warning (`MSB3923`).
  - **A tag such as `v1.0.3`: pinned.** It is downloaded once, and later builds never touch the network. Pin in apps
    that deploy to production, so a new release can't change a build of the same commit.
- The DLL is cached in `obj/typesafe-env/latest/` or `obj/typesafe-env/<tag>/`.
- `DotNetEnv` loads `.env` into the environment at runtime (see step 2). The generator itself only runs at build time.
- The `.env` is picked up next to the project or one folder up (the repository root, for the usual `repo/src/App` layout).

### 2. Load `.env` and register `Env` in `Program.cs`

```csharp
DotNetEnv.Env.NoClobber().TraversePath().Load(); // before CreateBuilder; real env vars (e.g. production secrets) win
var builder = WebApplication.CreateBuilder(args);
var env = new Env(builder.Configuration); // throws here, at startup, if a required value is missing or malformed
builder.Services.AddSingleton(env);

// use it while configuring services...
builder.Services.AddSingleton(new NpgsqlDataSourceBuilder(env.ConnectionStrings__Db).Build());

var app = builder.Build();
// ...or inject it anywhere
app.MapGet("/", (Env env) => $"{env.PGHOST}:{env.PGPORT}");
```

`Env` is generated in the project's root namespace, and the generated file adds a `global using` for that namespace.
That makes `Env` available in every file, including a top-level `Program.cs`, without a `using`. It takes an
`IConfiguration`, so it reads every configuration source ASP.NET Core sets up (environment variables,
`appsettings.json`, ...).

Create it yourself as shown, rather than only registering it with `AddSingleton<Env>()`. Dependency injection would
create it on first use, which moves the check away from startup.

Write the DotNetEnv call fully qualified as above, not with `using DotNetEnv;`. That namespace also has a class named
`Env`, and the import would make `Env` ambiguous.

### 3. Build

```bash
dotnet build   # or dotnet run
```

In Rider or Visual Studio, `Env` shows up in IntelliSense after the first build has downloaded the generator.

## Things to know

- **A `.env` must exist wherever the app is built**, including CI and Docker, because the generator reads it at build
  time. Without one, no `Env` class is generated and code using it won't compile. The values only decide the types, so
  a committed `.env` with development values works. Check that `.dockerignore` doesn't exclude it.
- **Only projects that see a `.env` get an `Env` class.** A test project that references the app doesn't get an empty
  duplicate.
- **Typing is based on the development value.** Only digits count as numbers, so `-1` or `3.5` give a `string`.
- **The downloaded DLL runs inside the compiler**, so only point `TypesafeEnvRepo` at a repository you trust.
- **The first build needs network access**, and so does every build of an app that uses the latest release. A fresh
  CI or Docker build always downloads.

## Developing the generator

```bash
dotnet build -c Release   # output: src/Typesafe.Env.Generator/bin/Release/netstandard2.0/Typesafe.Env.Generator.dll
```

To try a local change in an app without releasing, point the app's `Analyzer` item at that path instead of the
downloaded one.

### Releasing

Releases are automatic. Every push to `master` or `main` that changes `src/` (or the workflow) runs
`.github/workflows/release.yml`, which:

1. takes `ReleaseSeries` from `src/Typesafe.Env.Generator/Typesafe.Env.Generator.csproj` (e.g. `1.0`),
2. finds the highest existing tag in that series and releases the next patch (`v1.0.0`, then `v1.0.1`, ...),
3. creates the tag and a GitHub release with `Typesafe.Env.Generator.dll` attached.

Apps on the latest release get it on their next build. Pinned apps update by changing `TypesafeEnvVersion`.

- **README-only commits don't release**, because of the `paths` filter. To release anyway, run the workflow from the
  repository's Actions tab.
- **To start a new series**, for example after a breaking change, set `<ReleaseSeries>2.0</ReleaseSeries>` and push.
  The next release is `v2.0.0`.
- **No keys:** the workflow uses the built-in `GITHUB_TOKEN`.
