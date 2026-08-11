# LLM Judge Calibrator

Paste an LLM judge's confusion matrix from a human-labeled calibration set and get a prediction-powered corrected pass rate with a confidence interval, beside Rogan-Gladen and its refusal states.

## Live demo

https://0xelitesystem.github.io/llm-judge-calibrator/

## Features

**The headline estimator is prediction-powered inference, not Rogan-Gladen.** This is the whole point of the tool and it is a deliberate reordering. `theta_PP = mean_unlabeled(f) - mean_labeled(f - Y)`, reported with a 95 percent interval. PPI needs no assumption whatsoever about how the judge errs. It works under correlated error, under asymmetric error, and under error that depends on which model wrote the response, because the correction is arithmetic on your own labels rather than a model of the judge's behaviour. What it does require, absolutely, is that the labeled subset be a random draw from the pool you want the number for.

**Rogan-Gladen stays on screen beside it, with its assumption printed and its breaking points demonstrated.** The classical screening-test correction, `(p_app + spec - 1) / (sens + spec - 1)`, assumes the judge's sensitivity and specificity are fixed properties that transfer unchanged from the calibration set to production, which means judge error independent of the system under test. The tool shows the number and shows exactly where it dies: below zero when the apparent rate falls under `1 - specificity`, above one when the apparent rate exceeds sensitivity, and undefined when `sens + spec` is at or under 1. One-click buttons load a real confusion matrix for each of those three states. **The divergence between the two estimators is the lesson,** and it is a better lesson than either number alone.

**A hard gate, not a warning, on how the calibration set was built.** You must declare whether the labeled subset was randomly sampled. If you say it was hand-picked, stratified, or adversarially selected, or that you do not know, the tool refuses to print PPI or Rogan-Gladen and explains why in full. Calibration sets in practice get built by someone choosing interesting cases, and a corrected rate transported from an interesting-case error rate is worse than the uncorrected rate because it looks rigorous. Cohen's kappa and the descriptive statistics still render, because those are properties of the table you entered and make no claim about production.

**A sampling consistency check that can catch a set that is not the random draw someone believes it is.** The tool computes a two-proportion z between the judge's pass rate on the calibration set and its pass rate on the unlabeled pool. Under genuine random sampling those should agree. When they do not, the headline is suspect no matter what the radio button says.

**Cohen's kappa computed from the observed marginals, never hardcoded to 0.5.** That is the entire difference between kappa and a decorative number. A judge that says pass to everything, on a set that is 85 percent pass, scores 85 percent raw agreement and a kappa of exactly 0.00. That case is one of the load buttons, and the page prints `p_e` with its four marginal totals written out so you can see where it came from.

**Uncertainty propagated through the estimated sensitivity and specificity, not binomial error on the apparent rate alone.** This is the most useful output on the page. On the built-in sample, a 200-example calibration set against a 5,000-item run gives a naive interval of plus or minus 0.66 points on the judge's raw rate and plus or minus 4.57 points on the corrected rate, a factor of 6.9 wider. A 3-point regression that looks unmissable on the raw number sits inside the corrected interval on the level. The page states that distinction explicitly rather than over-applying it: a difference between two systems scored by the same judge is a different quantity, and part of the calibration error is common to both and cancels.

**The attenuation identity, presented as a ceiling rather than a correction.** Under judge error that is independent of the system under test and symmetric between the two error types, a true gap `G` is reported as `G * (1 - 2e)`, and `1 - 2e` turns out to be exactly `sens + spec - 1`, the same Youden index Rogan-Gladen divides by. The tool prints `measured_gap / (1 - 2e)` as the most the identity licenses you to claim, with a floor of zero, because error correlated with which system produced the output can manufacture a gap from nothing. It also prints the judge's two error rates separately so you can see whether the symmetry precondition holds on your own data before you look at the number. It never tells you to replace your reported gap with the corrected one.

**Refusal states you can load in one click.** Rogan-Gladen below zero, Rogan-Gladen above one, Rogan-Gladen degenerate at `sens + spec = 1`, kappa exactly 0.00 on 85 percent raw agreement, and the hard sampling gate firing. Watching a tool refuse is more informative than watching it always answer.

**Housekeeping.** Dark by default with a light toggle that persists. Responsive to 360 pixels with every wide table and code block scrolling inside its own container. Keyboard accessible with visible focus. Copy buttons for the full summary and for the raw arithmetic. No result is truncated or capped, and the page says so on screen. Single file, no external dependencies, no network requests, no API key, no backend.

## How it works

Four counts go in: judge pass with human pass, judge pass with human fail, judge fail with human pass, judge fail with human fail. Then the size and pass count of the unlabeled run you actually want a number for.

    n     = TP + FP + FN + TN                    (labeled calibration set)
    N     = size of the unlabeled production run
    p_app = judge passes on the unlabeled run / N

    theta_PP = p_app - (FP - FN)/n

    s2_d = (n/(n-1)) * ( (FP+FN)/n - ((FP-FN)/n)^2 )     variance of judge-minus-human
    s2_f = (N/(N-1)) * p_app * (1 - p_app)               variance of the judge's verdict

    95% CI = theta_PP +/- 1.96 * sqrt( s2_d/n + s2_f/N )

Rogan-Gladen is computed alongside, with a delta-method interval that propagates the sampling error in both `sens` and `spec`:

    sens = TP/(TP+FN)      spec = TN/(TN+FP)      J = sens + spec - 1
    theta_RG = (p_app + spec - 1) / J

    Var(theta_RG) ~= [ p_app(1-p_app)/N
                       + theta_RG^2     * sens(1-sens)/(TP+FN)
                       + (1-theta_RG)^2 * spec(1-spec)/(TN+FP) ] / J^2

One property worth knowing, which falls out of the algebra rather than out of a citation: when the judge's pass rate on the calibration set exactly equals its pass rate on the unlabeled pool, PPI and Rogan-Gladen both collapse to the human pass rate of the calibration set, which is always inside zero to one. The two estimators only separate as those two apparent rates drift apart, which they normally do from sampling noise alone. A corollary is that an out-of-range Rogan-Gladen result is always driven by that mismatch, never by the calibration set on its own. You can watch it happen in the page: enter `TP 166, FP 22, FN 2, TN 10` against a run of 5,000 where the judge passed 4,700, and the divergence panel reads exactly zero because both apparent rates are 94 percent.

All core logic lives in pure functions (`calibrate`, `ppiEstimate`, `roganGladen`, `cohensKappa`, `attenuationCeiling`, `binomialInterval`, `twoProportionZ`) that take plain values and return plain result objects, with no document or window access. They can be lifted out of the page and run in Node unchanged.

### Sources

Every quoted sentence in the page was copied verbatim from the linked source on 2026-08-11 and is kept in visually separate, date-stamped blocks so that a later revision upstream can never make the calculator's own arithmetic look wrong. Paraphrases carry no quote marks.

- Prediction-powered inference: Angelopoulos, Bates, Fannjiang, Jordan, Zrnic, arXiv:2301.09633, and the reference implementation at github.com/aangelopoulos/ppi_py
- Rogan W J and Gladen B, *Estimating prevalence from the results of a screening test*, American Journal of Epidemiology, 1978 (PubMed 623091)
- Out-of-range behaviour of the Rogan-Gladen estimator: the R `epiR` reference page for `epi.prev`
- Judge error correlated with the system under test: arXiv:2604.06996v2 (22 Jul 2026) and arXiv:2604.22891 (24 Apr 2026). The first is cited and linked at its v2 URL on purpose. Its submission history is v1 8 Apr 2026, v2 22 Jul 2026, v3 3 Aug 2026; the sentence quoted on the page appears verbatim in v2 and v3, while v1 reads "up to 50%" rather than "more than 50%".

## Not to be confused with

[eval-significance-calculator](https://github.com/0xelitesystem/eval-significance-calculator) asks whether a delta between two eval runs is real or sampling noise. This tool asks whether the instrument that produced the delta is lying.

Run the significance calculator when you have two runs and want to know if the gap survives McNemar. Run this one when you have human labels on a sample of the judge's verdicts and want to know what the judge's headline number is actually worth.

## Privacy

Everything runs in your browser. There is no backend, no API key, no analytics, no cookies, no telemetry, and no network request of any kind. The page is one HTML file with all CSS and JavaScript inline and no external dependencies. Your confusion matrix and your pass counts never leave the tab, which is the point when the numbers describe an unshipped model.

## License

MIT. See [LICENSE](LICENSE).

## More

- More tools: https://0xelitesystem.github.io/
- https://elitesystem.ai
