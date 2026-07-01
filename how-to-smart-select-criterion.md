# How to create a Smart Select criterion

A Smart Select criterion filters the content tree in the translation UI. It has two parts: a **C# backend** that evaluates each content item, and a **TypeScript frontend** that renders the criterion's controls. Both live in the MP.LanguageMaster source.

## 1. Create the TypeScript criterion

Add a file to `src/OptiLanguageMaster.Cms/ClientResources/src/translation/selection/criteria/`:

```tsx
// criteria/wordCount.tsx
import React from 'react';
import { defineCriterion } from '../types';
import type { CriterionProps } from '../types';

type Value = {
  minWords: number;
};

function WordCountCriterion({ value, onChange, disabled }: CriterionProps<Value>) {
  return (
    <input
      type="number"
      className="olm-criterion__input"
      min={0}
      value={value.minWords}
      disabled={disabled}
      onChange={(e) => onChange({ minWords: parseInt(e.target.value) || 0 })}
    />
  );
}

export const wordCount = defineCriterion<Value>({
  id: 'word-count',           // must match CriterionId in the C# class
  label: 'Minimum Word Count',
  section: 'workflow',        // 'gap' | 'workflow' | 'structural'
  sortIndex: 50,
  defaultValue: { minWords: 0 },
  Component: WordCountCriterion,
  // toQueryParam controls what is sent to the C# backend.
  // Return {} to send nothing — the backend treats an absent parameter as "no filter".
  toQueryParam: (v) => v.minWords > 0 ? { minWords: v.minWords } : {},
});
```

Then register it in `registry.ts`:

```ts
import { wordCount } from './criteria/wordCount';

export const CRITERIA: SelectionCriterion[] = [
  // ...existing criteria...
  wordCount,
];
```

Run `npm run build` from `ClientResources/` to rebuild the bundle.

## 2. Create the C# criterion

```csharp
using EPiServer.Core;
using OptiLanguageMaster.Cms.Interfaces;

public sealed class WordCountCriterion : ISelectionCriterion
{
    // Must exactly match the `id` in the TypeScript defineCriterion() call.
    public string CriterionId => "word-count";

    public bool Matches(IContent content, SelectionQueryContext ctx)
    {
        if (!ctx.Parameters.TryGetProperty("minWords", out var p))
            return true; // no parameter sent — include everything

        var minWords = p.GetInt32();
        if (minWords <= 0) return true;

        // ctx.ContentRepository, ctx.SourceLanguage, and ctx.TargetLanguage are
        // also available for more complex evaluations.
        var wordCount = CountWords(content);
        return wordCount >= minWords;
    }

    private static int CountWords(IContent content)
    {
        if (content is not PageData page) return 0;
        var text = page.Property
            .OfType<PropertyLongString>()
            .Select(p => p.Value as string ?? string.Empty);
        return string.Join(' ', text)
            .Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    }
}
```

## Key rules

- **Id must match**: `CriterionId` must exactly match the `id` field in the TypeScript `defineCriterion()` call.
- **Return true for absent parameters**: when `toQueryParam` returns `{}` the parameter is not sent, so check for its absence and return `true` (include all content).
- **`ctx.Parameters`** is the JSON object emitted by `toQueryParam()` on the frontend — use `TryGetProperty` to read values safely.
- **Stateless**: criteria are singletons; use constructor injection for services rather than storing state on the class.
- **Available context**: `ctx.SourceLanguage`, `ctx.TargetLanguage`, `ctx.ContentRepository`, `ctx.RootContentId` (the selected subtree root, if any), `ctx.IsInlineBlock`, `ctx.CatalogContentType`.

## 3. Register in `Program.cs`

```csharp
services.AddOptiLanguageMaster(o => { ... });
services.AddSelectionCriterion<WordCountCriterion>();
```

The criterion appears automatically in the Smart Select panel under the section specified in the TypeScript definition.
