# How to create a translation provider

A translation provider connects MP.LanguageMaster to a translation API. Providers are singletons — credentials are passed per-call, not stored on the class.

## 1. Implement `ITranslationProvider`

```csharp
using OptiLanguageMaster.Core.Interfaces;
using OptiLanguageMaster.Core.Models;

public sealed class MyTranslationProvider : ITranslationProvider
{
    // Stable key used in routing rules and job logs — never change this.
    public string ProviderId => "my-provider";
    public string DisplayName => "My Translation Service";

    // True if your API accepts HTML markup directly.
    public bool SupportsHtml => false;

    // False if your API must be called once per segment (e.g. an LLM).
    // When true (the default), all segments for a job are sent in one call.
    public bool SupportsBatching => true;

    // Fields shown in the Settings UI connection form.
    public IReadOnlyList<ProviderCredentialField> CredentialFields =>
    [
        new() { Key = "apiKey", Label = "API Key", IsSecret = true },
        new() { Key = "region", Label = "Region", Placeholder = "us-east-1" },
    ];

    // Optional extra fields shown below the connection form.
    public IReadOnlyList<ProviderCredentialField> ConfigurationFields =>
    [
        new()
        {
            Key       = "formality",
            Label     = "Formality",
            FieldType = "select",
            Options   =
            [
                new() { Value = "default", Label = "Default" },
                new() { Value = "formal",  Label = "Formal"  },
            ],
        },
    ];

    public async Task<IEnumerable<SupportedLanguage>> GetSupportedLanguagesAsync(
        ProviderCredentials credentials, CancellationToken ct)
    {
        // Call your API to fetch supported language pairs.
        // Return an empty list on error — the UI will show no languages.
        var apiKey = credentials.GetRequired("apiKey");
        // ... call API ...
        return [new SupportedLanguage { Code = "fr", Name = "French" }];
    }

    public async Task<ProviderValidationResult> ValidateCredentialsAsync(
        ProviderCredentials credentials, CancellationToken ct)
    {
        try
        {
            var apiKey = credentials.GetRequired("apiKey");
            // ... test API call ...
            return ProviderValidationResult.Ok();
        }
        catch (Exception ex)
        {
            return ProviderValidationResult.Fail(ex.Message);
        }
    }

    public async Task<TranslationResult> TranslateAsync(
        TranslationRequest request, CancellationToken ct)
    {
        try
        {
            var apiKey = request.Credentials.GetRequired("apiKey");

            // Translate all segments. Results must be in the same order as request.Segments.
            var translated = new List<string>();
            foreach (var segment in request.Segments)
            {
                // segment.Content is the text or HTML to translate.
                // segment.ContentType is SegmentContentType.Html or PlainText.
                var result = await CallMyApi(
                    segment.Content, request.SourceLanguage, request.TargetLanguage, apiKey, ct);
                translated.Add(result);
            }

            return TranslationResult.Success(ProviderId, translated);
        }
        catch (HttpRequestException ex) when (IsTransient(ex))
        {
            // isTransient: true triggers routing-rule fallback to the next provider.
            return TranslationResult.Failure(ProviderId, ex.Message, isTransient: true);
        }
        catch (Exception ex)
        {
            return TranslationResult.Failure(ProviderId, ex.Message, isTransient: false);
        }
    }
}
```

## Key rules

- **Never throw** from `TranslateAsync` or `ValidateCredentialsAsync` — always return a failure result.
- **Segment order**: `TranslatedSegments` must be the same length and in the same order as `request.Segments`.
- **Transient errors**: set `isTransient: true` on network or rate-limit errors so the job falls back to the next routing rule.
- **Missing credentials**: `credentials.GetRequired(key)` throws `InvalidOperationException` if the key is missing — catch it and return `Failure`.
- **Stateless**: providers are registered as singletons; never store per-request state on the class.

## 2. Register in `Program.cs`

```csharp
services.AddOptiLanguageMaster(o => { ... });
services.AddTranslationProvider<MyTranslationProvider>();
```

The provider appears automatically in the Settings UI and is available for selection in routing rules.
