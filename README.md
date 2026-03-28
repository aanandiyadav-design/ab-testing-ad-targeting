# Ad Targeting A/B Testing Analysis

![R](https://img.shields.io/badge/R-276DC3?style=flat&logo=r&logoColor=white)
![dplyr](https://img.shields.io/badge/dplyr-1a1a2e?style=flat)
![ggplot2](https://img.shields.io/badge/ggplot2-1a1a2e?style=flat)
![Method](https://img.shields.io/badge/Method-A%2FB%20Experiment-2d6a4f?style=flat)
![Stats](https://img.shields.io/badge/Stats-Welch%20t--test%20%7C%20Power%20Analysis-2d6a4f?style=flat)

Evaluating the monetization and engagement tradeoffs of geographic ad constraints using a two-arm randomized experiment.

---

## Key Results

| Metric | Control | Treatment | Δ Change |
|---|---|---|---|
| Click-Through Rate | 68.0% | 60.3% | ▼ −7.7 pp |
| Conversion Rate | 75.9% | 85.7% | ▲ +9.8 pp |
| Sell-Through Rate | 94.3% | 74.0% | ▼ −20.3 pp |
| Total Revenue | — | — | ▼ −15.7% *(significant)* |

> **Bottom line:** Geographic targeting constraints improved conversion quality but reduced click volume and auction competition enough to cause a statistically significant 15.7% revenue decline.

---

## Overview

This project evaluates how restricting the set of ads shown to users affects engagement, conversion behavior, and revenue outcomes. The analysis compares a baseline ad system against a treatment condition where ads are limited to a narrower geographic range — a common product decision in ad platforms.

The core question: *does stricter targeting improve outcome quality without sacrificing revenue at scale?*

---

## Implementation

### Unified KPI Aggregation

All metrics were computed in a single `dplyr` pipeline to ensure internal consistency across engagement and monetization measures:

```r
metrics_summary <- data %>%
  mutate(
    click_flag    = AdClick  == "clicked",
    purchase_flag = Purchase == "yes"
  ) %>%
  group_by(Treatment) %>%
  summarise(
    impressions             = n(),
    clicks                  = sum(click_flag),
    purchases               = sum(purchase_flag),
    revenue                 = sum(Revenue),
    CTR                     = clicks / impressions,
    conversion_rate         = purchases / clicks,
    STR                     = impressions / max(impressions),
    revenue_per_impression  = revenue / impressions
  )
```

### Significance Testing & Power Analysis

Revenue impact was assessed with a **Welch t-test** (robust to unequal group variance), followed by power analysis to confirm the experiment was adequately sized to detect the observed CTR effect:

```r
# Revenue significance
t_test_revenue <- t.test(Revenue ~ Treatment, data = data)

# Cohen's h effect size for CTR (proportion comparison)
effect_size_ctr <- ES.h(
  p1 = metrics_summary$CTR[metrics_summary$Treatment == "control"],
  p2 = metrics_summary$CTR[metrics_summary$Treatment == "treatment"]
)

# Verify sufficient statistical power
power_analysis <- pwr.2p.test(
  h         = effect_size_ctr,
  n         = min(table(data$Treatment)),
  sig.level = 0.05
)
```

---

## Findings

**User behavior:** The CTR decline under geographic restriction indicates users engage with nearby-but-not-proximate ads when other attributes are favorable — proximity alone is not a binding constraint on purchase intent.

**Revenue mechanics:** Under a pay-per-click model, click volume drives revenue more than conversion efficiency. A 9.8 pp conversion rate lift was insufficient to offset the 7.7 pp CTR decline at scale.

**Auction dynamics:** A 20 pp drop in sell-through rate signals reduced auction competition. Fewer eligible advertisers suppress clearing prices and constrain fill — a compounding monetization risk beyond the direct click loss.

**Takeaway:** Tightening geographic targeting is not advisable without a compensating mechanism — such as expanded inventory sourcing or a hybrid relevance scoring model — to maintain monetizable reach. System performance depends on balancing relevance against volume rather than optimizing either in isolation.

---

## Tech Stack

| Area | Tools |
|---|---|
| Data transformation | R, dplyr |
| Visualization | ggplot2 |
| Statistical inference | Welch t-test, Cohen's h, `pwr` package |
| Experiment design | Two-arm A/B test, treatment-level aggregation |
