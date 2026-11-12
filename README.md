# Typesafe.Env

A C# source generator that turns the keys in your `.env` into a typed `Env` class, so configuration is used as
`env.PGHOST` instead of `configuration["PGHOST"]`:

- adding a key to `.env` makes it available in code right away,
- removing or renaming a key breaks the build everywhere it is used,
- `env.Validate()` at startup stops the app if a required key is missing or a number isn't a number.

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
  <TypesafeEnvVersion>v1.0.0</TypesafeEnvVersion>
  <TypesafeEnvDir>$(BaseIntermediateOutputPath)typesafe-env/$(TypesafeEnvVersion)/</TypesafeEnvDir>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="DotNetEnv" Version="3.2.0" />
  <CompilerVisibleProperty Include="RootNamespace" />
  <AdditionalFiles Include="$(MSBuildProjectDirectory)/.env" Condition="Exists('$(MSBuildProjectDirectory)/.env')" />
  <AdditionalFiles Include="$(MSBuildProjectDirectory)/../.env" Condition="Exists('$(MSBuildProjectDirectory)/../.env')" />
</ItemGroup>
<Target Name="TypesafeEnv" BeforeTargets="CoreCompile">
  <DownloadFile Condition="!Exists('$(TypesafeEnvDir)Typesafe.Env.Generator.dll')"
                SourceUrl="https://github.com/uldahlalex/Typesafe.Env/releases/download/$(TypesafeEnvVersion)/Typesafe.Env.Generator.dll"
                DestinationFolder="$(TypesafeEnvDir)" />
  <ItemGroup>
    <Analyzer Include="$(TypesafeEnvDir)Typesafe.Env.Generator.dll" />
  </ItemGroup>
</Target>
```

What each part does:

- `TypesafeEnvVersion` is the release tag. To upgrade, change it and build again.
- The first build downloads the DLL into `obj/typesafe-env/<version>/`. Later builds reuse it, so they work offline.
- `DotNetEnv` loads `.env` into the environment at runtime (see step 2). The generator itself only runs at build time.
- The `.env` is picked up next to the project or one folder up (the repository root, for the usual `repo/src/App` layout).

### 2. Load `.env` and register `Env` in `Program.cs`

```csharp
DotNetEnv.Env.NoClobber().TraversePath().Load(); // before CreateBuilder; real env vars (e.g. production secrets) win
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<Env>();

var app = builder.Build();
app.Services.GetRequiredService<Env>().Validate(); // fail at startup, not on first use

// anywhere else: inject Env
app.MapGet("/", (Env env) => $"{env.PGHOST}:{env.PGPORT}");
```

`Env` is generated in the project's root namespace and takes an `IConfiguration`, so it reads every configuration
source ASP.NET Core sets up (environment variables, `appsettings.json`, ...).

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
- **The downloaded DLL runs inside the compiler**, so only point `TypesafeEnvVersion` at releases you trust.

## Developing the generator

```bash
dotnet build -c Release   # output: src/Typesafe.Env.Generator/bin/Release/netstandard2.0/Typesafe.Env.Generator.dll
```

To try a local change in an app without releasing, point the app's `Analyzer` item at that path instead of the
downloaded one.

### Releasing

```bash
git tag v1.0.1
git push origin v1.0.1
```

`.github/workflows/release.yml` builds the generator and creates a GitHub release with `Typesafe.Env.Generator.dll`
attached. It uses the built-in `GITHUB_TOKEN`, so there are no keys to manage. Apps then switch by changing
`TypesafeEnvVersion`.
