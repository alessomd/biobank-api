## Summary

<!-- What does this change do and why? -->

## Changes

-

## How to test

```bash
pytest
ruff check . && ruff format --check .
```

## Checklist

- [ ] `pytest` and `ruff` pass locally
- [ ] Tests cover the success path and error paths
- [ ] No secrets, real `.env` files or real personal/health data
- [ ] `.env.example` updated if a setting was added
- [ ] README updated if behaviour, configuration or commands changed
