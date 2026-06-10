# model-price-repo

Filtered model pricing data for CRS and sub2api projects. Syncs from the upstream [litellm](https://github.com/BerriAI/litellm) pricing file on a schedule, applying configurable prefix filters to keep only the models you actually use.

## How it works

A GitHub Actions workflow runs every 10 minutes (and on manual trigger):

1. Downloads the full `model_prices_and_context_window.json` from litellm
2. Filters models by the prefix rules in `config.json`
3. Merges new models into the existing output (additive — never removes)
4. Applies alias mappings and custom model definitions
5. Optionally fills flat pricing fallbacks from `tiered_pricing`
6. Writes the output JSON + SHA-256 hash, commits only if content changed

## Configuration

All settings live in [`config.json`](config.json):

| Field | Description |
|---|---|
| `upstream_url` | URL to the upstream litellm pricing JSON |
| `output_file` | Output filename (default: `model_prices_and_context_window.json`) |
| `hash_file` | SHA-256 hash filename for change detection |
| `sync_mode` | `"additive"` (only add new) or `"full"` (replace each run) |
| `update_existing` | Whether to update pricing data for models already in the output |
| `prefix_filters` | List of prefixes — a model key must start with one to be included |
| `keyword_filters` | Case-insensitive keywords matched against model keys and `litellm_provider` |
| `exclude_patterns` | Substring patterns to exclude (applied before inclusion matching) |
| `tiered_pricing_fallback` | Optional compatibility fallback that copies first-tier prices into missing top-level pricing fields |
| `aliases` | Map alias model keys to existing source models (deep copy pricing) |
| `custom_models` | Manually defined pricing objects, always injected |

### Adding model families

Edit `prefix_filters` for direct model-key prefixes and `keyword_filters` for
provider-prefixed models:

```json
{
  "prefix_filters": [
    "deepseek-",
    "qwen-"
  ],
  "keyword_filters": [
    "deepseek",
    "qwen",
    "dashscope"
  ]
}
```

### Adding aliases

Aliases create copies of an existing model's pricing under a new key:

```json
{
  "aliases": {
    "claude-opus-4-6-thinking": {
      "source": "claude-opus-4-6",
      "description": "Thinking variant, same pricing"
    }
  }
}
```

If the source model doesn't exist in the filtered data, the alias is skipped with a warning.

### Tiered pricing fallback

Some upstream models, especially DashScope/Qwen models, publish range-based
pricing in `tiered_pricing` and leave top-level price fields empty. Enable
`tiered_pricing_fallback` to copy the first tier into missing flat fields for
legacy consumers:

```json
{
  "tiered_pricing_fallback": {
    "enabled": true,
    "strategy": "first_tier",
    "only_when_missing": true
  }
}
```

`tiered_pricing` remains the authoritative source for exact range-based billing.
The copied flat fields are only a compatibility fallback.

## Running locally

```bash
python3 scripts/sync_prices.py --config config.json --repo-root .
```

No pip dependencies — uses Python standard library only.

## CRS integration

Point CRS to the raw output file from this repo:

```
MODEL_PRICES_URL=https://raw.githubusercontent.com/<owner>/model-price-repo/main/model_prices_and_context_window.json
```

The output JSON structure is identical to what litellm produces (model key -> pricing object), so CRS `pricingService.js` works without changes.

## License

[MIT](LICENSE)
