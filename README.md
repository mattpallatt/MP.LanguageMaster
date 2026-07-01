# MP.LanguageMaster

Multi-provider translation add-on for Optimizely CMS 12 and 13. An alternative to the native Labs.LanguageManager with a content tree UI, multiple translation backends, per-item publish control, routing rules, async jobs with real-time progress, translation memory, and a full audit log.

**Providers**: DeepL · Azure Cognitive Services · OpenAI · Google Cloud · Anthropic Claude · ModernMT · Amazon Translate · Smartling · Custom

---

## Install

```
dotnet add package MP.LanguageMaster.Cms
```

Commerce catalog support (optional):

```
dotnet add package MP.LanguageMaster.Commerce
```

---

## Setup

**1. Register services** in `StartUp.cs`

```csharp
services.AddOptiLanguageMaster(o =>
{
    o.SqlConnectionString = configuration.GetConnectionString("EPiServerDB");
});

// Prevents credential loss on app restart
services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(@"C:\keys\olm"))
    .SetApplicationName("OptiLanguageMaster");

// Optional — Commerce catalog support
services.AddOptiLanguageMasterCommerce();
```

**2. Map the SignalR hub** in `StartUp.cs`

```csharp
app.UseEndpoints(e =>
{
    e.MapOptiLanguageMaster();
    // ...existing mappings...
});
```

**3. Register the Shell module**

```csharp
services.Configure<ProtectedModuleOptions>(pm =>
{
    if (!pm.Items.Any(x => x.Name.Equals("OptiLanguageMaster", StringComparison.OrdinalIgnoreCase)))
        pm.Items.Add(new ModuleDetails { Name = "OptiLanguageMaster" });
});
```

**4. Ensure Hangfire is running** — OLM queues translation jobs through your existing Hangfire setup; a background worker must be active.

**5. Start the app** — `olm_*` SQL tables are created automatically on first run (requires `CREATE TABLE` permission on the connection string user).

**6. Open the UI** — navigate to **Language Master** in the CMS left-hand menu, configure your translation providers, set up routing rules, and run translations.

---

## Access control

Requires `CmsAdmins` by default. Override with any role:

```csharp
services.AddOptiLanguageMaster(o =>
{
    o.SqlConnectionString = ...;
    o.AuthorizedRoles = ["Editors"];
});
```

---

## CMS compatibility

| CMS | .NET target |
|-----|-------------|
| Optimizely CMS 12 | net6.0 / net8.0 / net10.0 |
| Optimizely CMS 13 | net10.0 |

---
