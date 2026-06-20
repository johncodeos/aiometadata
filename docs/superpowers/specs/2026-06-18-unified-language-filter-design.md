# Plan: Unified Language List with Role Toggle

## Context

The current TMDB Discover Builder has two separate sections for language filtering:
- "Original Language" (include) — Select + Add button + Badge chips
- "Exclude Original Languages" (exclude) — Select + Add button + Badge chips

**Problems:**
1. Both sections look identical — users confuse include vs exclude
2. Adding a language to include silently removes it from exclude with no warning
3. 100+ languages in a flat dropdown with no search — hard to find specific languages
4. 4 clicks per language: open dropdown → scroll → click language → click "Add"

**Goal:** Replace with a single unified section using color-coded role toggles.

## Scope

Single file change: `configure/src/components/sections/DiscoverBuilderDialog.tsx`

No backend changes. Backend still receives `excludedOriginalLanguages: string[]` and `originalLanguages: string[]` via query params.

---

## Step 1: Replace State Variables

**Remove** (lines 773-777):
```tsx
const [originalLanguage, setOriginalLanguage] = useState('');
const [originalLanguages, setOriginalLanguages] = useState<string[]>([]);
const [pendingOriginalLanguage, setPendingOriginalLanguage] = useState('');
const [excludedOriginalLanguages, setExcludedOriginalLanguages] = useState<string[]>(() => [...DEFAULT_EXCLUDED_ORIGINAL_LANGUAGES]);
const [pendingExcludedOriginalLanguage, setPendingExcludedOriginalLanguage] = useState('');
```

**Add:**
```tsx
const [languageRoles, setLanguageRoles] = useState<Record<string, 'include' | 'exclude'>>(() => {
  const initial: Record<string, 'include' | 'exclude'> = {};
  for (const code of DEFAULT_EXCLUDED_ORIGINAL_LANGUAGES) {
    initial[code] = 'exclude';
  }
  return initial;
});
const [languageSearch, setLanguageSearch] = useState('');
```

**Rationale:** Single source of truth for language state. Each language code maps to exactly one role ('include', 'exclude') or is absent (unselected). Default excluded languages still pre-populate as 'exclude'.

---

## Step 2: Replace `availableOriginalLanguages` and `availableExcludedOriginalLanguages` useMemo

**Remove** (lines 973-987):
```tsx
const availableOriginalLanguages = useMemo(...)
const availableExcludedOriginalLanguages = useMemo(...)
```

**Add:**
```tsx
const filteredLanguages = useMemo(() => {
  const search = languageSearch.toLowerCase().trim();
  if (!search) return sortedLanguages;
  return sortedLanguages.filter(lang =>
    lang.english_name?.toLowerCase().includes(search) ||
    lang.name?.toLowerCase().includes(search) ||
    lang.iso_639_1.toLowerCase().includes(search)
  );
}, [sortedLanguages, languageSearch]);
```

**Rationale:** We no longer need two separate "available" lists since all languages are always visible in the unified view. Search filter is applied to the full list.

---

## Step 3: Replace Handlers

**Remove** (lines 2548-2568):
```tsx
const handleAddOriginalLanguage = () => { ... };
const handleAddExcludedOriginalLanguage = () => { ... };
```

**Add:**
```tsx
const handleToggleLanguageRole = (code: string) => {
  setLanguageRoles(prev => {
    const current = prev[code];
    const next = { ...prev };
    if (!current) {
      next[code] = 'include';
    } else if (current === 'include') {
      next[code] = 'exclude';
    } else {
      delete next[code];
    }
    return next;
  });
};

const handleClearAllLanguages = () => {
  setLanguageRoles({});
};
```

**Rationale:** Three-state cycle: Off → Include → Exclude → Off. Clear all resets everything.

---

## Step 4: Update `getLanguageLabel`

**No change needed** — keep as-is (line 2543-2546). It works with any language code.

---

## Step 5: Replace the Two UI Sections

**Remove** (lines 3885-4001): Both the Include and Exclude sections for TMDB.

**Replace with single unified section:**
```tsx
{discoverSource === 'tmdb' && (
  <div className="space-y-3">
    <Label>Original Language</Label>
    {/* Search input */}
    <Input
      placeholder="Search languages..."
      value={languageSearch}
      onChange={(e) => setLanguageSearch(e.target.value)}
    />
    {/* Language grid */}
    <div className="max-h-[200px] overflow-y-auto rounded-lg border border-white/[0.06] bg-muted/30 p-2">
      <div className="flex flex-wrap gap-1.5">
        {filteredLanguages.map(lang => {
          const role = languageRoles[lang.iso_639_1];
          return (
            <Badge
              key={lang.iso_639_1}
              variant="outline"
              className={cn(
                'cursor-pointer select-none transition-colors',
                role === 'include' && 'bg-emerald-500/20 text-emerald-300 border-emerald-500/30 hover:bg-emerald-500/30',
                role === 'exclude' && 'bg-red-500/20 text-red-300 border-red-500/30 hover:bg-red-500/30',
                !role && 'hover:bg-muted'
              )}
              onClick={() => handleToggleLanguageRole(lang.iso_639_1)}
            >
              {lang.english_name || lang.name || lang.iso_639_1}
            </Badge>
          );
        })}
      </div>
    </div>
    {/* Selected languages summary */}
    {Object.keys(languageRoles).length > 0 && (
      <div className="space-y-2">
        <div className="flex flex-wrap gap-1.5">
          {Object.entries(languageRoles).map(([code, role]) => (
            <Badge
              key={code}
              variant="secondary"
              className={cn(
                'gap-1 pl-2 pr-1 py-1',
                role === 'include' && 'bg-emerald-500/20 text-emerald-300 border border-emerald-500/30',
                role === 'exclude' && 'bg-red-500/20 text-red-300 border border-red-500/30'
              )}
            >
              <span className="max-w-[180px] truncate">{getLanguageLabel(code)}</span>
              <button
                type="button"
                onClick={() => handleToggleLanguageRole(code)}
                className="rounded-sm p-0.5 hover:bg-background/50"
                aria-label={`Remove ${getLanguageLabel(code)}`}
              >
                <Trash2 className="h-3 w-3" />
              </button>
            </Badge>
          ))}
        </div>
        <button
          type="button"
          className="text-xs text-muted-foreground hover:text-foreground"
          onClick={handleClearAllLanguages}
        >
          Clear all
        </button>
      </div>
    )}
  </div>
)}
```

**Non-TMDB sources:** Keep the existing single-select (lines 3934-3951) unchanged.

---

## Step 6: Update `resetState`

**Change** (lines 1181-1185):
```tsx
// OLD
setOriginalLanguage('');
setOriginalLanguages([]);
setPendingOriginalLanguage('');
setExcludedOriginalLanguages([...DEFAULT_EXCLUDED_ORIGINAL_LANGUAGES]);
setPendingExcludedOriginalLanguage('');
```

**To:**
```tsx
// NEW
const initial: Record<string, 'include' | 'exclude'> = {};
for (const code of DEFAULT_EXCLUDED_ORIGINAL_LANGUAGES) {
  initial[code] = 'exclude';
}
setLanguageRoles(initial);
setLanguageSearch('');
```

---

## Step 7: Update `buildFormState`

**Change** (lines 2358-2371):
```tsx
// OLD
if (discoverSource === 'tmdb' || discoverSource === 'tvdb') {
  Object.assign(state, {
    ...
    originalLanguage,
    originalLanguages,
    ...
  });
}
if (discoverSource === 'tmdb') {
  Object.assign(state, {
    ...
    excludedOriginalLanguages,
    ...
  });
}
```

**To:**
```tsx
if (discoverSource === 'tmdb' || discoverSource === 'tvdb') {
  const includeLangs = Object.entries(languageRoles)
    .filter(([_, role]) => role === 'include')
    .map(([code]) => code);
  const excludeLangs = Object.entries(languageRoles)
    .filter(([_, role]) => role === 'exclude')
    .map(([code]) => code);

  Object.assign(state, {
    ...
    originalLanguage: includeLangs[0] || '',
    originalLanguages: includeLangs,
    languageRoles,
  });
}
if (discoverSource === 'tmdb') {
  const excludeLangs = Object.entries(languageRoles)
    .filter(([_, role]) => role === 'exclude')
    .map(([code]) => code);

  Object.assign(state, {
    ...
    excludedOriginalLanguages: excludeLangs,
    ...
  });
}
```

---

## Step 8: Update Form Restore Logic

**Change** (lines 1317-1332):
```tsx
// OLD
if (fs.originalLanguage) setOriginalLanguage(fs.originalLanguage);
if (Array.isArray(fs.originalLanguages)) {
  setOriginalLanguages(fs.originalLanguages);
} else if (fs.originalLanguage) {
  setOriginalLanguages([fs.originalLanguage]);
} else {
  setOriginalLanguages([]);
}
const storedExcludedLanguages = editingCatalog.metadata?.discover?.excludedOriginalLanguages;
if (Array.isArray(fs.excludedOriginalLanguages)) {
  setExcludedOriginalLanguages(fs.excludedOriginalLanguages);
} else if (Array.isArray(storedExcludedLanguages)) {
  setExcludedOriginalLanguages(storedExcludedLanguages);
} else {
  setExcludedOriginalLanguages([]);
}
```

**To:**
```tsx
// Support both new format (languageRoles) and legacy format
if (fs.languageRoles && typeof fs.languageRoles === 'object') {
  setLanguageRoles(fs.languageRoles);
} else {
  // Legacy: convert old format to new
  const roles: Record<string, 'include' | 'exclude'> = {};
  const includeLangs = Array.isArray(fs.originalLanguages)
    ? fs.originalLanguages
    : fs.originalLanguage ? [fs.originalLanguage] : [];
  const excludeLangs = Array.isArray(fs.excludedOriginalLanguages)
    ? fs.excludedOriginalLanguages
    : Array.isArray(editingCatalog?.metadata?.discover?.excludedOriginalLanguages)
      ? editingCatalog.metadata.discover.excludedOriginalLanguages
      : [...DEFAULT_EXCLUDED_ORIGINAL_LANGUAGES];

  for (const code of includeLangs) {
    roles[code] = 'include';
  }
  for (const code of excludeLangs) {
    if (!roles[code]) roles[code] = 'exclude';
  }
  setLanguageRoles(roles);
}
```

---

## Step 9: Update Preview Request

**No change needed** — lines 1976-1978 still work. The `excludedOriginalLanguages` variable is derived from `languageRoles` in `buildFormState` and passed through formState. But we also need the raw array for preview.

**Add before preview fetch** (around line 1976):
```tsx
const previewExcludedLangs = Object.entries(languageRoles)
  .filter(([_, role]) => role === 'exclude')
  .map(([code]) => code);
if (previewExcludedLangs.length > 0) {
  queryParams.set('excludedOriginalLanguages', previewExcludedLangs.join(','));
}
```

**Remove** the old:
```tsx
if (excludedOriginalLanguages.length > 0) {
  queryParams.set('excludedOriginalLanguages', excludedOriginalLanguages.join(','));
}
```

---

## Step 10: Handle Non-TMDB `originalLanguage` Fallback

The old code had `originalLanguage` state for non-TMDB sources. Since we removed it, we need to ensure non-TMDB sources still work.

**Add back** a minimal state for non-TMDB:
```tsx
const [legacyOriginalLanguage, setLegacyOriginalLanguage] = useState('');
```

And restore the non-TMDB Select section using this state (lines 3934-3951 stay as-is, just using `legacyOriginalLanguage`).

Actually — looking more carefully, the old code had both `originalLanguage` (for non-TMDB single-select) AND `originalLanguages` (for TMDB multi-select). The non-TMDB section at line 3934 uses `originalLanguage`. We need to keep that state variable for non-TMDB sources.

**Revised approach:** Keep `originalLanguage` state for non-TMDB sources only:
```tsx
const [originalLanguage, setOriginalLanguage] = useState('');
```

---

## Verification

After implementation, verify:
1. `vite build` passes
2. `tsc` passes (backend)
3. ESLint: no new errors
4. Manual test: Open Discover Builder → TMDB source → verify unified language section
5. Manual test: Click language → cycles through include/exclude/off
6. Manual test: Search filters languages
7. Manual test: Save catalog → edit → verify form restores correctly
8. Manual test: Preview works with excluded languages
