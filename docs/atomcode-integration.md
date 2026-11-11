# AtomCode Integration

## Supported AtomCode Version

This implementation is based on **AtomCode v0.11.0** configuration format as documented in:

- Official Configuration Reference: https://github.com/atomcode-ai/atomcode/blob/main/docs/configuration.md
- Third-party Integration Examples:
  - magpie PR #736: https://github.com/some-repo/magpie/pull/736
  - dsh-connect-atomcode: https://github.com/example/dsh-connect-atomcode

## Configuration Format

AtomCode uses a shared-provider model where all configured accounts coexist in `~/.atomcode/config.toml`:

```toml
default_provider = "model-key"
default_model = "model-key"

[provider_accounts.account-id]
provider = "openai"  # or "claude", "deepseek", "gemini", "ollama"
base_url = "https://api.example.com/v1"
api_key = "sk-..."  # Optional: when empty, uses AtomCode's native OAuth (auth.toml)

[models."model-key"]
account = "account-id"
model = "model-id"
context_window = 262144  # Optional
```

### Supported Provider Types

According to official documentation:
- `openai` - OpenAI-compatible APIs
- `claude` / `anthropic` (legacy alias) - Anthropic Claude
- `deepseek` - DeepSeek
- `gemini` - Google Gemini
- `ollama` - Ollama
- `openai_responses` (third-party extension)

### Environment Variables

- `ATOMCODE_HOME` - Custom data directory (default: `~/.atomcode`)

## CC Switch Implementation

### Architecture

- **Mode**: Coexist (like Kimi/Dsh/Mcode) - all accounts present in live config simultaneously
- **Live Config**: `~/.atomcode/config.toml`
- **Operations**:
  - **Add**: Write `provider_accounts.<id>` and `models.*` nodes, set `liveConfigManaged: true`
  - **Switch**: Update `default_provider` and `default_model` to point to the account's model
  - **Remove**: Delete the account's nodes from config.toml
  - **Edit**: Update in live config first, then commit to database

### Edge Cases Handled

1. **Default Model Protection**: Cannot delete or rename the currently selected default model
2. **Account Removal**: Cannot remove an account if its model is currently selected as default
3. **Key Collision**: Model keys must not collide with models owned by other accounts
4. **Atomic Writes**: All config.toml writes use temp file + rename for atomicity

### Whitelist Integration

AtomCode is added to the following whitelists:

- **ProviderList.tsx**: `isInConfig` - checks `meta.liveConfigManaged` flag
- **App.tsx**: 
  - Switch handler - calls `handleAddToLiveConfig` (not `switchProvider`)
  - Remove handler - `removeFromConfig` available
  - SetAsDefault handler - calls `switchProvider` (changes default_provider/default_model)
  - Duplicate - sets `addToLive: false` (don't auto-add copy to config.toml)

### Database Schema

```sql
CREATE TABLE providers (
  id TEXT PRIMARY KEY,
  app TEXT NOT NULL,
  name TEXT NOT NULL,
  settings_config JSON NOT NULL,
  meta JSON,  -- { liveConfigManaged: boolean }
  ...
);
```

## Known Limitations

1. **Import from Official Config**: Users who configured AtomCode following only the official documentation (using `[providers.*]` tables instead of `[provider_accounts.*]`) will get zero imports. The `provider_accounts` format appears only in third-party integration examples.

2. **Version Compatibility**: This implementation is tested against AtomCode v0.11.0. Future versions may introduce breaking changes to the configuration format.

3. **OAuth Accounts**: AtomCode's native OAuth (stored in `auth.toml`) is not directly managed by CC Switch - only the API key-based accounts in `provider_accounts`.

## Testing

Run backend tests with isolated home directory to avoid polluting real config:

```bash
cd src-tauri
CC_SWITCH_TEST_HOME="$(mktemp -d)" cargo test atomcode
```

Frontend unit tests:

```bash
pnpm test:unit atomcode
```

## Future Improvements

1. Document the exact AtomCode version/commit this integration targets
2. Add support for official `[providers.*]` format import
3. Improve OAuth account detection and display
4. Add config validation against AtomCode's schema
