# Typesafe.Env

Generates a typed `Env` class from your `.env` file. Use `env.PGHOST` instead of `configuration["PGHOST"]`, and a
removed or renamed key breaks the build.

## Setup

### 1. Add a `.env` next to your `.csproj`

```bash
PGHOST=localhost
PGPORT=5432
```

### 2. Add this inside `<Project>` in your `.csproj`

```xml
<PropertyGroup>
  <!-- Empty: latest release. Or pin a tag, e.g. v1.0.1 -->
  <TypesafeEnvVersion></TypesafeEnvVersion>
  <TypesafeEnvRepo>https://github.com/uldahlalex/Typesafe.Env</TypesafeEnvRepo>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="DotNetEnv" Version="3.2.0" />
  <CompilerVisibleProperty Include="RootNamespace" />
  <AdditionalFiles Include="$(MSBuildProjectDirectory)/.env" Condition="Exists('$(MSBuildProjectDirectory)/.env')" />
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
  <DownloadFile Condition="$(_TypesafeEnvLatest) Or !$(_TypesafeEnvCached)"
                SourceUrl="$(_TypesafeEnvUrl)" DestinationFolder="$(_TypesafeEnvDir)"
                ContinueOnError="$(_TypesafeEnvCached)" />
  <ItemGroup>
    <Analyzer Include="$(_TypesafeEnvDir)Typesafe.Env.Generator.dll" />
  </ItemGroup>
</Target>
```

### 3. Use it in `Program.cs`

```csharp
DotNetEnv.Env.NoClobber().TraversePath().Load();

var builder = WebApplication.CreateBuilder(args);
var env = new Env(builder.Configuration); // throws at startup if a value is missing
builder.Services.AddSingleton(env);

var app = builder.Build();
app.MapGet("/", (Env env) => $"{env.PGHOST}:{env.PGPORT}");
app.Run();
```

Don't add `using DotNetEnv;`. It also has a class named `Env`, which clashes with the generated one.

### 4. Run

```bash
dotnet run
```

## Types

| `.env` | Property |
|---|---|
| `PGPORT=5432` | `int PGPORT` |
| `PGHOST=localhost` | `string PGHOST` |
| `SENTRY_DSN=` | `string? SENTRY_DSN` (optional) |
| `Jwt__Secret=abc` | `string Jwt__Secret` (configuration key `Jwt:Secret`) |

Real environment variables override `.env`, so set production values as environment variables. The `.env` must exist
wherever the app is built, including CI and Docker.

## Releasing

Every push to `master`/`main` that changes `src/` publishes the next version (`v1.0.0`, `v1.0.1`, ...). To start a
new series, change `<ReleaseSeries>` in `src/Typesafe.Env.Generator/Typesafe.Env.Generator.csproj`.
