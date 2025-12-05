# Retry Logic Specification V2.1

## Overview

Defines retry logic with exponential backoff and jitter for all API calls.

## Jitter Formula

```yaml
backoff_type: exponential_with_jitter
jitter_formula: "delay = base * 2^retry * (0.5 + 0.5 * random())"
base_delay_ms: 1000
```

## Per-API Configurations

| API | Max Attempts | Timeout (ms) | Notes |
|-----|--------------|--------------|-------|
| Perplexity | 4 | 180000 | Deep Research is slow |
| Claude | 3 | 60000 | Standard config |
| ChatPRD | 2 | 30000 | Fast API |
| Devin | 2 | 300000 | Builds are slow |
| GitHub | 3 | 10000 | Critical dependency |
