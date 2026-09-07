# sloth.dev/contrib/denominator_corrected_rules/v1

## High level explanation

Plugin ported from [#459 Sloth PR][PR]. Full details are in the PR.

**Note:** This plugin replaces all SLI recording rules and adds new metadata rules.

This plugin adjusts SLOs for services with seasonal traffic patterns (for example: high traffic during the day, very low traffic at night).
Normally, SLOs treat all burn rates the same. But during low-traffic periods, even a few failed requests can cause false alerts and pages.

To fix this, the plugin applies a correction factor based on request volume. The burn rate impact scales with traffic levels, higher traffic means failures weigh more, and lower traffic means they weigh less.

If your service experiences low request volumes at certain times and you see noisy alerts, this plugin can help.

More details in the original [PR].

## How it works

For every alert window, the plugin generates a `slo:numerator_correction:ratio{window}` recording rule that holds the correction factor `k`, computed as the ratio of the request rate over the short window to the request rate over the total (e.g. 30d) window:

```promql
clamp_max(
  (
    <total query over the short window>
  )
  /
  (
    <total query over the total window>
  ),
  1.0
)
```

The generated SLI error recording rules then substitute this factor into the SLI formula, effectively becoming `(1 - k * bad/total)` instead of the usual `(1 - bad/total)`.

### Correction factor upper bound (`clamp_max`, k ≤ 1.0)

The correction factor is capped at `1.0` by wrapping the ratio expression in `clamp_max(..., 1.0)`, i.e. `k ≤ 1.0` is always guaranteed.

**Problem:** On services with historically low (or absent) traffic that suddenly start receiving load, the SLI calculation could produce absurd negative values (in practice, values around ~-52000% were observed).

**Root cause:** The plugin computes the correction factor as `k = rate(5m) / rate(30d)`. If `rate_30d ≈ 0.0001 req/s` and a traffic spike brings `rate_5m ≈ 1.0 req/s`, then `k` skyrockets to `10000`. Substituting such a value into the SLI formula (`1 - k * bad/total`) overflows and produces negative percentages.

**Solution:** An upper bound was added to the generated expression: `clamp_max((numerator)/(denominator), 1.0)`, i.e. `k ≤ 1.0`.

**Impact:**

- The plugin now only does what it was designed for: it lowers the significance of errors during anomalously low traffic (`k < 1.0`) and can no longer amplify errors during spikes after idle periods.
- `+Inf`/giant-number artifacts are eliminated, so dashboards and alerts are predictable.
- Backward compatibility: for services with stable traffic the behavior does not change (`k` was already ≤ 1.0).

## Config

- `disableOptimized`(**Optional**, `bool`): If `true`, disables optimized rule generation for long SLI windows. Optimized rules use short-window recording rules to derive long-window SLIs with lower Prometheus resource usage, at the cost of reduced accuracy. Defaults to `false`.

## Env vars

None

## Order requirement

This plugin should run after rule generation plugins.

## Usage examples

### Regular usage

```yaml
  sloPlugins:
    chain:
      - id: "sloth.dev/contrib/denominator_corrected_rules/v1"
```

### Disable optimization

```yaml
sloPlugins:
  chain:
    - id: "sloth.dev/contrib/denominator_corrected_rules/v1"
      config:
        disableOptimized: true
```


[PR]: https://github.com/slok/sloth/pull/459
