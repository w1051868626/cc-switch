# AtomCode Integration Fixes Summary

## Issues Fixed

### 1. ProviderList.tsx - isInConfig Logic ✅

**Problem**: The default case in `isInConfig` switch returned `true`, causing atomcode providers to always show as "in config" regardless of their actual state.

**Fix**: Made the switch exhaustive by explicitly listing all switching-mode apps (claude, codex, gemini, etc.) that return `true`, and changed default to return `false`. AtomCode now correctly checks `meta.liveConfigManaged` flag like other coexist-mode apps.

**Location**: `src/components/providers/ProviderList.tsx:185-217`

### 2. Backend - Default Model Deletion Edge Case ✅

**Problem**: When editing the current account, users could delete or rename the default model, causing AtomCode's `default_model` to point to a non-existent model key.

**Fix**: Added validation in `apply_provider` to check if the default model belongs to the account being edited. If the new config doesn't include the default model key, the operation is rejected with a clear error message.

**Location**: `src-tauri/src/atomcode_config.rs:225-256`

**Test**: Added `cannot_delete_or_rename_default_model` test case

### 3. Documentation - AtomCode Version and Config Source ✅

**Problem**: No documentation explaining which AtomCode version this integration targets, or where the `provider_accounts` format comes from (it's not in official docs).

**Fix**: Created comprehensive documentation covering:
- Supported AtomCode version (v0.11.0)
- Configuration format and source references
- Architecture and implementation details
- Edge cases handled
- Testing instructions
- Known limitations

**Location**: `docs/atomcode-integration.md`

## Issues Already Correct (No Fix Needed)

### 4. Update Path Ordering ✅

**Status**: Already correct - no changes needed

The backend already writes live config first, then commits to database on success:

```rust
crate::atomcode_config::save_and_commit(
    &provider.id,
    Some(provider.settings_config.clone()),
    create,
    || {
        state.db.save_provider(app_type.as_str(), provider)?;
        Ok(true)
    },
)
```

If live config write fails, the commit callback is never called, so the database stays intact. This prevents the next startup's backfill from overwriting the database with stale live config.

**Locations**: 
- `src-tauri/src/atomcode_config.rs:285-318` - `save_and_commit`
- `src-tauri/src/services/provider/mod.rs:7142-7150` - `switch_and_commit`

### 5. App.tsx Integration ✅

**Status**: Already correct - atomcode is properly integrated

App.tsx already has atomcode in all necessary whitelists:
- Switch handler: calls `handleAddToLiveConfig` (line 1677)
- Remove handler: `removeFromConfig` available (line 1690)  
- SetAsDefault handler: calls `switchProvider` (line 1706)
- Duplicate: sets `addToLive: false` (line 1324)

AtomCode is a coexist-mode app, so it doesn't need `switchMode` prop - it uses `buildAdditiveSections` automatically.

## Issues Deferred

### 6. i18n Translations (ko/ru)

**Status**: Deferred - will fallback to English

According to AGENTS.md, ko and ru are community translations that are allowed to be incomplete and fallback to English. The core five languages (zh, zh-TW, en, ja, tr) already have complete atomcode translations.

**Future**: Community contributors can add ko/ru translations later.

### 7. ProviderCard Actions Whitelist

**Status**: Not needed - already handled by presentation.ts

The whitelist logic is in `presentation.ts:buildAdditiveSections` (lines 715-950), not in ProviderCard or ProviderCardActions. AtomCode shares the kimi logic (lines 727-784):
- Shows "Add" button when not in config
- Shows "Set as Default" menu item + "Remove" button when in config
- Prevents removing the default provider

## Testing

### Backend Tests

```bash
cd src-tauri
CC_SWITCH_TEST_HOME="$(mktemp -d)" cargo test atomcode
```

New tests added:
- `cannot_delete_or_rename_default_model` - validates edge case protection

Existing tests verify:
- Switch preserves comments and unrelated TOML sections
- Add writes account without changing default_model
- Remove rejects current account
- Model key collisions are rejected
- Native provider types work without base_url

### Frontend Tests

```bash
pnpm typecheck
pnpm test:unit
```

## Files Changed

1. `src/components/providers/ProviderList.tsx` - Fixed isInConfig default case
2. `src-tauri/src/atomcode_config.rs` - Added default model protection + test
3. `docs/atomcode-integration.md` - Created comprehensive documentation
4. `docs/atomcode-fixes-summary.md` - This summary

## Conclusion

The AtomCode integration now correctly handles:
- Provider config state detection
- Default model lifecycle protection
- Documentation and version tracking
- Atomic live config writes with proper ordering
- All App.tsx integration points

No breaking changes were made - all fixes are backward compatible with existing AtomCode provider data.
