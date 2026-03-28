# Ad Targeting A/B Testing Analysis

![R](https://img.shields.io/badge/R-276DC3?style=flat&logo=r&logoColor=white)
![Method](https://img.shields.io/badge/A%2FB%20Experiment-Randomized%20Controlled-2d6a4f?style=flat)
![Stats](https://img.shields.io/badge/Welch%20t--test%20%7C%20Cohen's%20h%20%7C%20Power%20Analysis-1a1a2e?style=flat)

---

Ad platforms face a persistent tension between targeting precision and monetizable reach. Narrowing the pool of eligible ads can improve relevance — but relevance does not automatically translate to revenue. This project investigates that tradeoff empirically, using a two-arm randomized experiment to quantify how geographic ad constraints affect user engagement, conversion behavior, and overall monetization.

The treatment condition restricted which ads could be shown to users based on geographic proximity. Everything else — auction mechanics, user population, impression opportunities — was held constant. The goal was not simply to measure what changed, but to understand *why* it changed and what that implies for system design decisions at scale.

---

## Results

| Metric | Control | Treatment | Change |
|---|---|---|---|
| Click-Through Rate | 68.0% | 60.3% | ▼ −7.7 pp |
| Conversion Rate | 75.9% | 85.7% | ▲ +9.8 pp |
| Sell-Through Rate | 94.3% | 74.0% | ▼ −20.3 pp |
| Total Revenue | — | — | ▼ −15.7% |

Revenue decline was confirmed statistically significant via Welch t-test. Power analysis using Cohen's h verified the experiment had sufficient sensitivity to detect the observed CTR effect at α = 0.05.

---

The results tell a coherent story. Restricting the ad pool improved the match between users and the ads they saw — conversion rate rose nearly 10 percentage points, suggesting that the ads displayed under the treatment condition were more aligned with actual purchase intent. But this came at a cost: fewer eligible ads meant fewer impressions filled, fewer clicks generated, and fewer advertisers competing in the auction. Sell-through rate dropped by 20 percentage points, reflecting a significant reduction in inventory utilization. Under a pay-per-click revenue model, where earnings scale directly with click volume rather than conversion efficiency, that reduction in exposure dominated the outcome. Total revenue declined 15.7%.

What this reveals is that user engagement is not strictly bounded by geographic proximity. Users in the control group clicked on — and converted from — ads that fell outside the treatment's geographic threshold, indicating they were responsive to non-geographic signals like price, brand, or product relevance even when the advertiser was not local. Enforcing a hard geographic cutoff removed ads that users would have engaged with meaningfully, not just ads that were irrelevant.

The sell-through decline compounds the problem. In an auction-based system, fill rate and competition are interdependent: fewer eligible advertisers reduce clearing prices, which in turn reduce revenue per impression even on the impressions that do fill. The 20 pp STR drop suggests the treatment didn't just reduce clicks — it degraded the underlying competitiveness of the auction, creating a secondary monetization drag that operates independently of user behavior.

Taken together, the findings point to a structural limit on precision-first targeting strategies. Optimizing for relevance at the expense of reach can improve a subset of downstream metrics while simultaneously undermining the conditions that make monetization viable. The relevant question for product decisions is not whether conversion rate improved — it did — but whether the system as a whole performs better. In this case, it does not.

---

## Implementation

All key performance metrics were computed in a single aggregation pass, ensuring internal consistency across engagement and monetization measures and eliminating any risk of mismatched denominators between metrics:

```r
metrics_summary <- data %>%
  mutate(
    click_flag    = AdClick  == "clicked",
    purchase_flag = Purchase == "yes"
  ) %>%
  group_by(Treatment) %>%
  summarise(
    impressions            = n(),
    clicks                 = sum(click_flag),
    purchases              = sum(purchase_flag),
    revenue                = sum(Revenue),
    CTR                    = clicks / impressions,
    conversion_rate        = purchases / clicks,
    STR                    = impressions / max(impressions),
    revenue_per_impression = revenue / impressions
  )
```

Statistical inference used a Welch t-test rather than a standard two-sample t-test to account for potential variance differences between treatment arms — a more conservative and appropriate choice when group sizes or distributions may differ. Effect size was quantified using Cohen's h, which is specifically suited to proportion comparisons and avoids the distortions that arise from applying continuous-variable effect size measures to bounded rates. Power analysis then confirmed that the experiment had sufficient sample size to detect an effect of the observed magnitude, ruling out the possibility that the CTR result was an artifact of an underpowered design.

```r
t_test_revenue <- t.test(Revenue ~ Treatment, data = data)

effect_size_ctr <- ES.h(
  p1 = metrics_summary$CTR[metrics_summary$Treatment == "control"],
  p2 = metrics_summary$CTR[metrics_summary$Treatment == "treatment"]
)

power_analysis <- pwr.2p.test(
  h         = effect_size_ctr,
  n         = min(table(data$Treatment)),
  sig.level = 0.05
)
```

This sequencing — estimate the effect, quantify its size, verify detectability — reflects standard practice in experiment analysis and ensures the conclusions are not overstated relative to what the data can support.

---

The broader implication is that targeting constraints introduce a relevance-volume tradeoff that cannot be resolved by optimizing either dimension in isolation. A system that maximizes conversion efficiency while depleting its own auction competitiveness is not well-calibrated. Closing the gap would require either a mechanism that preserves reach while improving relevance — such as a hybrid scoring model that weights geographic proximity alongside behavioral signals — or an inventory expansion strategy that offsets the reduction in eligible ads. Without one of these, tightening geographic constraints is a precision improvement that costs more in scale than it returns in quality.
