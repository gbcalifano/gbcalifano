---
layout: post
title: "The latent bias in latent class analysis"
date: 2026-06-01 00:00:00 +0100
description: >
  After you fit a latent class model, linking the classes to outcomes is
  trickier than it looks. The standard approach biases your estimates toward zero.
tags: [methods, stata, latent-class]
categories: notes
toc:
  sidebar: left
math: true
giscus_comments: false
related_posts: false
---

<style>
/* ── scoped styles for this post ───────────────────────────────── */
.t-muted { opacity: 0.65; }
.cell-hi { fill: rgba(99, 179, 237, 0.2); }

.callout {
  background: var(--global-code-bg-color, #f4f6f8);
  border-left: 3px solid var(--global-theme-color, #4a90d9);
  padding: 0.85rem 1.1rem;
  margin: 1.25rem 0;
  border-radius: 0 4px 4px 0;
}
.callout p { margin: 0.4rem 0 0; }

.fig-cap {
  font-size: 0.875rem;
  color: var(--global-text-color-light, #6c757d);
  margin-top: 0.4rem;
  font-style: italic;
}

.methods-table {
  width: 100%;
  border-collapse: collapse;
  margin: 1rem 0;
  font-size: 0.93rem;
}
.methods-table th, .methods-table td {
  border: 1px solid var(--global-divider-color, #dee2e6);
  padding: 0.5rem 0.75rem;
}
.methods-table th {
  background: var(--global-code-bg-color, #f8f9fa);
  font-weight: 600;
}

.badge {
  display: inline-block;
  padding: 0.15em 0.5em;
  border-radius: 3px;
  font-size: 0.82em;
  font-weight: 600;
}
.badge-accepted { background: #2d8a4e; color: #fff; }
.badge-rejected { background: #c0392b; color: #fff; }

/* Stata input — default code-block styling, driven by highlight.js later */
.hljs-stata-code {
  font-family: var(--font-mono, "SFMono-Regular", Consolas, monospace);
  font-size: 0.85rem;
}

/* Stata output — dark terminal look */
.stata-log > .highlight > pre {
  background: #1e2127 !important;
  color: #abb2bf;
  font-size: 0.82rem;
  border: none;
}
.stata-log > .highlight > pre > code { color: #abb2bf; }

.faq h3 { font-size: 1rem; font-weight: 600; margin-top: 1.25rem; }
</style>

Latent class analysis is a method for discovering hidden subgroups of observations in your data. *Latent* means not directly observed: the classes are inferred from patterns in measured variables (such as survey responses, test items, behavioural indicators) rather than assigned by the researcher. Feed it enough items and it tells you that your sample breaks into — say — three distinct profiles, each with a characteristic response pattern.

Once you have those classes, the natural next question is: *do they differ on some outcome I care about?* Or: *what predicts membership in each group?* The standard approach to answering these questions introduces a systematic bias that shrinks your estimates.

Here is what goes wrong, why it matters, and how to fix it in Stata.

## The three-step workflow

In Step 1, you estimate the latent class model using only the observed indicators (no external variables yet). The model returns *posterior probabilities* for every observation: one probability per class, summing to 1. If your model has two classes, person *i* might come out 85% likely to belong to Class 1 and 15% likely to belong to Class 2.

In Step 2, you assign each person to their *modal* class — the one with the highest posterior. Person *i* at 85%/15% gets hard-assigned to Class 1. An alternative is *proportional* assignment, where each person contributes fractionally to every class weighted by their posteriors.[^1]

In Step 3, you run a regression, ANOVA, or multinomial logit using these class labels as a variable.

The problem is between Steps 2 and 3, wherein we treat the *latent* class variable as if it were *observed*.

<div class="fig">
<svg width="100%" viewBox="0 0 720 170" role="img" aria-label="Diagram of the three-step procedure">
<defs>
  <marker id="arr" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
    <path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
  </marker>
</defs>
<rect x="10" y="42" width="190" height="82" fill="none" stroke="currentColor" stroke-width="1.5"/>
<rect x="246" y="42" width="190" height="82" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<rect x="482" y="42" width="228" height="82" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="105" y="70" text-anchor="middle" font-size="22" font-weight="700">Step 1</text>
<text x="105" y="92" text-anchor="middle" font-size="14" class="t-muted">Fit the mixture model</text>
<text x="105" y="110" text-anchor="middle" font-size="14" class="t-muted">without external variables</text>
<text x="105" y="28" text-anchor="middle" font-size="13" class="t-muted">Estimate class structure</text>
<text x="341" y="70" text-anchor="middle" font-size="22" font-weight="700">Step 2</text>
<text x="341" y="92" text-anchor="middle" font-size="14" class="t-muted">Assign each observation</text>
<text x="341" y="110" text-anchor="middle" font-size="14" class="t-muted">to a class</text>
<text x="341" y="28" text-anchor="middle" font-size="13" class="t-muted">Modal or proportional</text>
<text x="596" y="70" text-anchor="middle" font-size="22" font-weight="700">Step 3</text>
<text x="596" y="92" text-anchor="middle" font-size="14" class="t-muted">Relate classes to</text>
<text x="596" y="110" text-anchor="middle" font-size="14" class="t-muted">covariates or outcomes</text>
<text x="596" y="28" text-anchor="middle" font-size="13" class="t-muted">Regression / ANOVA</text>
<rect x="191" y="59" width="64" height="48" fill="var(--global-bg-color, white)"/>
<line x1="200" y1="83" x2="244" y2="83" stroke="currentColor" stroke-width="1" fill="none" marker-end="url(#arr)"/>
<text x="222" y="73" text-anchor="middle" font-size="12" class="t-muted">posterior</text>
<text x="222" y="98" text-anchor="middle" font-size="12" class="t-muted">probs</text>
<rect x="427" y="59" width="64" height="48" fill="var(--global-bg-color, white)"/>
<line x1="436" y1="83" x2="480" y2="83" stroke="currentColor" stroke-width="1" fill="none" marker-end="url(#arr)"/>
<text x="458" y="73" text-anchor="middle" font-size="12" class="t-muted">class</text>
<text x="458" y="98" text-anchor="middle" font-size="12" class="t-muted">labels</text>
<text x="458" y="144" text-anchor="middle" font-size="13">↑</text>
<text x="458" y="160" text-anchor="middle" font-size="13" class="t-muted">classification error enters here</text>
</svg>
</div>

---

## The bias: class differences get diluted

When you assign someone to a class, you treat an uncertain membership as though it were known with certainty. People who truly belong to Class 1 get counted in Class 2, and vice versa. The groups look more similar than they actually are, hence the estimated difference between them shrinks.

<div class="callout">
  <strong>How large is the bias?</strong>
  <p>Simulation studies show naive three-step estimates are typically 20–45% smaller than the truth: largest when classes overlap, smallest when they are more cleanly separated.[^2]</p>
</div>

The mechanism is easy to see. Suppose two equal-sized classes have true outcome means of 1 and 0, and 20% of observations land in the wrong class. The "Class 1" bucket now contains 80 genuine members (mean 1) and 20 interlopers from Class 2 (mean 0), giving a bucket average of **0.80**. By symmetry the "Class 2" bucket averages **0.20**, and the naive estimate of the group difference is 0.60 rather than the true 1.00.

More generally, with a misclassification rate of $\varepsilon$ (say, 20%), the naive estimator recovers only $(1 - 2\varepsilon)$ of the true effect. At 10% error you keep 80% of the signal; at 30% error you keep only 40%. Models with moderate class overlap routinely sit between $\varepsilon = 0.15$ and $\varepsilon = 0.30$, which is well within the range where the bias is practically meaningful.

**A glass of intuition.** Suppose we want to measure how much more alcohol there is in *red wine* than in *water*. If we could sort the glasses perfectly, the comparison is stark: a deep-red glass of wine beside a clear glass of water. But modal assignment never hands us the true glasses: it hands us two *buckets*, each contaminated by the misclassified members of the other. The wine bucket gets a splash of water; the water bucket gets a splash of wine. Both come out more or less *pink*. Of course the gap between two pinks is narrower than the gap between red and clear, and the more the classes overlap (the lower the entropy), the more alike those two pinks become, until the measured difference all but disappears.

<div class="fig">
<svg width="100%" viewBox="0 0 720 212" role="img" aria-label="Wine and water glasses: as class overlap grows, the two estimated buckets both turn pink and the measured alcohol gap shrinks">
<defs>
  <marker id="arrw" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
    <path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
  </marker>
</defs>
<text x="360" y="20" text-anchor="middle" font-size="16" font-weight="700">What modal assignment actually compares</text>
<line x1="150" y1="38" x2="585" y2="38" stroke="var(--global-divider-color, #aaa)" stroke-width="1" marker-end="url(#arrw)"/>
<line x1="240" y1="50" x2="240" y2="190" stroke="var(--global-divider-color, #aaa)" stroke-width="1" stroke-dasharray="4 3"/>
<line x1="480" y1="50" x2="480" y2="190" stroke="var(--global-divider-color, #aaa)" stroke-width="1" stroke-dasharray="4 3"/>
<text x="120" y="62" text-anchor="middle" font-size="13" class="t-muted">Clean separation</text>
<text x="360" y="62" text-anchor="middle" font-size="13" class="t-muted">Some overlap</text>
<text x="600" y="62" text-anchor="middle" font-size="13" class="t-muted">Heavy overlap</text>
<!-- Column 1: clean separation -->
<g transform="translate(78,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#7c1a33" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<g transform="translate(162,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#eef3f7" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<text x="120" y="184" text-anchor="middle" font-size="14" font-weight="700">&#916; alcohol &#8776; 12</text>
<!-- Column 2: some overlap -->
<g transform="translate(318,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#b25c74" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<g transform="translate(402,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#e7bcca" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<text x="360" y="184" text-anchor="middle" font-size="14" font-weight="700">&#916; alcohol &#8776; 7</text>
<!-- Column 3: heavy overlap -->
<g transform="translate(558,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#cf9fab" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<g transform="translate(642,78)">
  <path d="M-18 0 Q-18 36 0 36 Q18 36 18 0 Z" fill="#d6a8b5" stroke="currentColor" stroke-width="1.3"/>
  <ellipse cx="0" cy="0" rx="18" ry="4" fill="none" stroke="currentColor" stroke-width="1.3"/>
  <line x1="0" y1="36" x2="0" y2="62" stroke="currentColor" stroke-width="1.3"/>
  <line x1="-13" y1="64" x2="13" y2="64" stroke="currentColor" stroke-width="1.3"/>
</g>
<text x="600" y="184" text-anchor="middle" font-size="14" font-weight="700">&#916; alcohol &#8776; 1</text>
</svg>
</div>

<p class="fig-cap">With 20% misclassification each bucket trades a fifth of its contents with the other: the wine bucket keeps only 80% of its colour, the water bucket picks up 20%, and a true gap of 12 is measured as roughly 7. Push the classes closer together (right) and the two buckets converge on the same pink: the estimated difference collapses toward zero even though nothing about the real wine and water has changed.</p>

---

## The classification error matrix

Think of D as a receipt that summarises how much Step 2 distorted things. The entry in row *s*, column *t* is the probability of being assigned to class *t* when your true class is *s*. Diagonal entries capture correct classification; off-diagonal entries capture misclassification.

Formally, D is an $S \times S$ matrix — for two classes, a 2 × 2 table. When classes are well separated, the diagonals approach 1 and the off-diagonals approach 0. When classes overlap, the off-diagonals grow, and so does the bias. The key point: you do not need to guess D. It can be computed directly from the posterior probabilities saved in Step 1, by averaging across observations within each true-class group.

<div class="fig">
<svg width="100%" viewBox="0 0 720 210" role="img" aria-label="Classification error matrix comparison">
<text x="200" y="22" text-anchor="middle" font-size="16" font-weight="700">Well-separated classes</text>
<text x="72" y="88" text-anchor="end" font-size="13" class="t-muted">True 1</text>
<text x="72" y="148" text-anchor="end" font-size="13" class="t-muted">True 2</text>
<text x="122" y="188" text-anchor="middle" font-size="13" class="t-muted">Pred. 1</text>
<text x="228" y="188" text-anchor="middle" font-size="13" class="t-muted">Pred. 2</text>
<rect x="78" y="58" width="88" height="52" class="cell-hi"/>
<rect x="78" y="58" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="122" y="89" text-anchor="middle" font-size="20" font-weight="700">0.94</text>
<rect x="184" y="58" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="228" y="89" text-anchor="middle" font-size="20" class="t-muted">0.06</text>
<rect x="78" y="118" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="122" y="149" text-anchor="middle" font-size="20" class="t-muted">0.05</text>
<rect x="184" y="118" width="88" height="52" class="cell-hi"/>
<rect x="184" y="118" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="228" y="149" text-anchor="middle" font-size="20" font-weight="700">0.95</text>
<line x1="360" y1="16" x2="360" y2="185" stroke="var(--global-divider-color, #aaa)" stroke-width="1" stroke-dasharray="4 3" fill="none"/>
<text x="538" y="22" text-anchor="middle" font-size="16" font-weight="700">Overlapping classes</text>
<text x="408" y="88" text-anchor="end" font-size="13" class="t-muted">True 1</text>
<text x="408" y="148" text-anchor="end" font-size="13" class="t-muted">True 2</text>
<text x="458" y="188" text-anchor="middle" font-size="13" class="t-muted">Pred. 1</text>
<text x="618" y="188" text-anchor="middle" font-size="13" class="t-muted">Pred. 2</text>
<rect x="414" y="58" width="88" height="52" class="cell-hi"/>
<rect x="414" y="58" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="458" y="89" text-anchor="middle" font-size="20" font-weight="700">0.72</text>
<rect x="574" y="58" width="88" height="52" fill="none" stroke="currentColor" stroke-width="1.5"/>
<text x="618" y="89" text-anchor="middle" font-size="20" font-weight="700">0.28</text>
<rect x="414" y="118" width="88" height="52" fill="none" stroke="currentColor" stroke-width="1.5"/>
<text x="458" y="149" text-anchor="middle" font-size="20" font-weight="700">0.31</text>
<rect x="574" y="118" width="88" height="52" class="cell-hi"/>
<rect x="574" y="118" width="88" height="52" fill="none" stroke="var(--global-divider-color, #aaa)" stroke-width="1"/>
<text x="618" y="149" text-anchor="middle" font-size="20" font-weight="700">0.69</text>
</svg>
</div>

---

## The fix: two bias-corrected methods

**Bolck–Croon–Hagenaars (BCH; [Bolck et al., 2004](https://doi.org/10.1093/pan/mph001)).** BCH takes the *inverse* of D and uses it as a set of correction weights. The dataset is expanded into one row per class per observation, each row weighted by an entry of $D^{-1}$. A standard weighted regression on this expanded dataset recovers unbiased estimates without ever touching the mixture model again, meaning that class definitions from Step 1 stay frozen.

**Maximum Likelihood (ML; [Vermunt, 2010](https://doi.org/10.1093/pan/mpq025)).** ML re-estimates the mixture model in Step 3, but holds the entries of D fixed as constraints. This lets the model account for classification error while estimating the relationship with the external variable, and it reports class-specific variances that BCH does not provide. The trade-off: it assumes normality within classes, which BCH does not require.[^2]

<div class="callout">
  <strong>When to use which</strong>
  <p>Use BCH when normality within classes is uncertain, or when you want class definitions to stay fixed between steps. Use ML for categorical outcomes or when classes are well separated and distributional assumptions are plausible.</p>
</div>

---

## Using `step3` in Stata

Computing and manipulating the D matrix is not exactly a piece of cake. Fortunately, the `step3` module implements both BCH and ML methods in Stata. Let's see how it works with a simple example.

Let's install the command and dataset first:

<pre><code class="hljs-stata-code">. net install st0801</code></pre>

We can now `sysuse height`, which contains fictional data on 4,071 American high school seniors. Our variables are: person ID (`pid`), student height (cm), sex (1 = female, 2 = male).

We also have a variable `y`, generated to follow a normal distribution with $\mu = 0$, $\sigma = 1$ for male students (48% of the sample), and $\mu = 0.5$, $\sigma = 1$ for female students (52% of the sample). Assume that the nature of `y` is unknown, and our goal is to test whether female students have higher values of `y` than male students. Further, suppose that our funny colleague dropped the sex variable from our dataset while you were in the toilet, making it our latent categorical variable in this example.[^3]

Fortunately for us, we still observe student height, which can serve as a useful proxy for sex. Heights may be represented as a mixture of two Gaussian components, corresponding to female and male students, with males expected to be, on average, about 5%–10% taller.

{% include figure.liquid loading="eager" path="assets/img/posts/note_1/fig_1.svg" alt="Histogram with overlaid kernel density of student heights, showing a bimodal distribution" caption="Figure 1. Distribution of student heights. The bimodal shape hints at two underlying subpopulations." class="img-fluid" %}

<pre><code class="hljs-stata-code">. sysuse height
. twoway hist height || kdensity height</code></pre>

More formally, we aim to fit the following model:

$$f(\text{height}) = \pi_1 \mathcal{N}(\mu_1, \sigma_1^2) + \pi_2 \mathcal{N}(\mu_2, \sigma_2^2)$$

where $f(\text{height})$ is the overall probability density of height, $\pi_1$ and $\pi_2$ are the mixing proportions ($\pi_1 + \pi_2 = 1$), and $\mathcal{N}(\mu_j, \sigma_j^2)$ are Gaussian distributions with mean $\mu_j$ and variance $\sigma_j^2$, representing the two subpopulations of female and male students. In Stata, we can fit this model with `fmm` or `gsem`, and predict the posterior class membership probabilities for each observation as follows:

<pre><code class="hljs-stata-code">. fmm 2: regress height
. predict cpost*, classposteriorpr</code></pre>

<div class="stata-log">
{% highlight text %}
Finite mixture model                                     Number of obs = 4,071
Log likelihood = -15016.15

------------------------------------------------------------------------------
             | Coefficient  Std. err.      z    P>|z|     [95% conf. interval]
-------------+----------------------------------------------------------------
1.Class      |  (base outcome)
-------------+----------------------------------------------------------------
2.Class      |
       _cons |  -.2149548   .2166024    -0.99   0.321    -.6394878    .2095781
------------------------------------------------------------------------------

Class:    1
Response: height
Model:    regress

-------------------------------------------------------------------------------
              | Coefficient  Std. err.      z    P>|z|     [95% conf. interval]
--------------+----------------------------------------------------------------
height        |
        _cons |   163.0747   .6220492   262.16   0.000     161.8555    164.2939
--------------+----------------------------------------------------------------
 var(e.height)|   42.44258   3.455455                      36.18273    49.78542
-------------------------------------------------------------------------------

Class:    2
Response: height
Model:    regress

-------------------------------------------------------------------------------
              | Coefficient  Std. err.      z    P>|z|     [95% conf. interval]
--------------+----------------------------------------------------------------
height        |
        _cons |   177.2601   .9649576   183.70   0.000     175.3688    179.1514
--------------+----------------------------------------------------------------
 var(e.height)|   51.78815   5.937079                      41.36635     64.8356
-------------------------------------------------------------------------------
{% endhighlight %}
</div>

The first component has an estimated mean height of $\hat{\mu}_1 = 163.07$ cm (about 5′4″ for the metrically resistant) with variance $\hat{\sigma}_1^2 = 42.44$, while the second has $\hat{\mu}_2 = 177.26$ cm (about 5′10″) with variance $\hat{\sigma}_2^2 = 51.79$. Since the second component corresponds to taller students, we label it "Male" and the first "Female". To plot the fitted model, we also need the mixing-proportion estimates $\hat{\pi}_1$ and $\hat{\pi}_2$. We will overlay the true sex-specific height distributions to assess how well the model has recovered them.

<pre><code class="hljs-stata-code">. estat lcprob, nose</code></pre>

<div class="stata-log">
{% highlight text %}
Latent class marginal probabilities                      Number of obs = 4,071

--------------------------------------------------------------
             |     Margin
-------------+------------------------------------------------
       Class |
          1  |   .5535327
          2  |   .4464673
--------------------------------------------------------------
{% endhighlight %}
</div>

{% include figure.liquid loading="eager" path="assets/img/posts/note_1/fig_2.svg" alt="Fitted mixture components overlaid on the true sex-specific height distributions" caption="Figure 2. Estimated mixture components (dashed) overlaid on the true sex-specific distributions (solid). The model recovers the two subpopulations well, though with some leakage near the overlap region." class="img-fluid" %}

<pre><code class="hljs-stata-code">. twoway (function .52*normalden(x,162.9026,6.526993), range(height)) ///
         (function .48*normalden(x,176.3803,7.747659), range(height)) ///
         (function .55*normalden(x,163.0747,sqrt(42.44258)), range(height)) ///
         (function .45*normalden(x,177.2601,sqrt(51.78815)), range(height)), ///
         legend(label(1 "Female") label(2 "Male") label(3 "Class 1") label(4 "Class 2")) ///
         ytitle("Density") xtitle("Height (cm)")</code></pre>

As shown in the figure above, the two components align fairly well, though not perfectly, with the true distributions. Our model estimates 55% female and 45% male students in the population. We can infer that the imperfect classification arising from using height alone is due to relatively tall female students, and relatively short male students, for whom the classification probabilities are not very separated (again, low entropy).

Nevertheless, a common approach at this stage is to assign each student to the class with the highest posterior membership probability and then compare means of `y` across classes using ANOVA, a t-test, or similar methods. If the mixture model had perfectly recovered the sex variable, the differences in `y` obtained via modal assignment would match those from the true sex variable. However, this is rarely the case. As a result, we expect the relationship between class membership and `y` to be somewhat *diluted* (recall the wine example above). To assess this, we estimate a regression model:

<pre><code class="hljs-stata-code">. generate Class = 1
. replace Class = 2 if cpost2 &gt; cpost1
. regress y i.Class</code></pre>

<div class="stata-log">
{% highlight text %}
      Source |       SS           df       MS      Number of obs   =     4,071
-------------+----------------------------------   F(1, 4069)      =    109.50
       Model |  112.081626         1  112.081626   Prob > F        =    0.0000
    Residual |  4164.85525     4,069  1.02355745   R-squared       =    0.0262
-------------+----------------------------------   Adj R-squared   =    0.0260
       Total |  4276.93687     4,070  1.05084444   Root MSE        =    1.0117

------------------------------------------------------------------------------
           y | Coefficient  Std. err.      t    P>|t|     [95% conf. interval]
-------------+----------------------------------------------------------------
     2.Class |   -.335732   .0320835   -10.46   0.000    -.3986332   -.2728308
       _cons |   .3984043   .0208967    19.07   0.000     .3574354    .4393732
------------------------------------------------------------------------------
{% endhighlight %}
</div>

The estimated difference between the two components — the coefficient of `2.Class` — is ≈ −0.336, about 33% smaller than the true difference of −0.5. Exactly the kind of dilution we expected. To correct for it, we use `step3` in BCH mode with the `outcome` option, which tells the command to treat `y` as a distal outcome of the latent class. We also supply the prefix of our posterior probabilities (`cpost`) and a variable name for the modal assignment (`W`).

<pre><code class="hljs-stata-code">. step3 y, pr(cpost) lclass(W) outcome bch</code></pre>

<div class="stata-log">
{% highlight text %}
STEP3: Distal outcome analysis

BCH Mean estimation                  Number of obs =     4,071
--------------------------------------------------------------
             | Coefficient  Std. err.     [95% conf. interval]
-------------+------------------------------------------------
y            |
           W |
          1  |   .4678218    .026197      .4164765     .519167
          2  |  -.0066626    .028869     -.0632448    .0499196
--------------------------------------------------------------
Note: Linearized std. err.
{% endhighlight %}
</div>

By default, `step3` for continuous distal outcomes reports class-specific means of the outcome. Postestimation commands such as `pwcompare` are also supported:

<pre><code class="hljs-stata-code">. pwcompare W</code></pre>

<div class="stata-log">
{% highlight text %}
Pairwise comparisons of marginal linear predictions

Margins: asbalanced

--------------------------------------------------------------
             |                                 Unadjusted
             |   Contrast   Std. err.     [95% conf. interval]
-------------+------------------------------------------------
y            |
           W |
     2 vs 1  |  -.4744844   .0450046     -.5626919   -.3862769
--------------------------------------------------------------
{% endhighlight %}
</div>

The BCH method indeed provided less biased estimates. Next, we try the ML method by removing the option `bch`. Recall that the ML method uses the entries of the matrix D to re-estimate the mixture model while incorporating the external variable(s). Consequently, the ML approach may alter the composition of the components in this third step, potentially changing the meaning of the latent classes themselves. To monitor such changes, we can include the option `detail`, which provides statistics on the switching of observations between classes during the procedure. If more than 20% of observations switch classes, `step3` will automatically display these statistics with a warning message.

<pre><code class="hljs-stata-code">. step3 y, pr(cpost) lclass(W) outcome detail</code></pre>

<div class="stata-log">
{% highlight text %}
Performing ML estimation...

STEP3: Distal outcome analysis

ML Mean estimation                   Number of obs =     4,071
--------------------------------------------------------------
             | Coefficient  Std. err.     [95% conf. interval]
-------------+------------------------------------------------
y            |
           W |
          1  |   .4674262    .026677      .4151401    .5197122
          2  |  -.0061706   .0293066     -.0636105    .0512693
-------------+------------------------------------------------
sigma2       |
       W#c.y |
          1  |   1.045716   .0405707      .9661987    1.125233
          2  |   .9324711   .0414606      .8512097    1.013732
--------------------------------------------------------------
Note: Robust std. err.
      Unequal variance across classes assumed.

Class composition (%) before and after Step 3

--------------------------------------------------
       Class |     Step 1      Step 3 |     Change
-------------+------------------------+-----------
           1 |      55.35       55.35 |      -0.00
           2 |      44.65       44.65 |       0.00
--------------------------------------------------

Observations in Step 1 class moved to a different class in Step 3

+------------------------+
|          n           % |
|------------------------|
|          1           0 |
+------------------------+
Note: results might be inconsistent for % > 20
{% endhighlight %}
</div>

The difference between the two components on `y`, estimated using the ML method, is quite similar to that obtained with the BCH method, both being only about 5% smaller than the true difference. From the `detail` output, we also see that just 1 out of 4,071 observations switched class during the ML procedure. In addition, the ML method reports variance estimates (`sigma2`) for each distal outcome within each class, which by default are allowed to differ across classes, though this can be constrained with the `eqvar` option.

---

## Choosing between methods

<table class="methods-table">
<thead>
  <tr>
    <th>Situation</th>
    <th>Method</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>Continuous distal outcome, normality uncertain</td>
    <td><span class="badge badge-accepted">BCH</span></td>
  </tr>
  <tr>
    <td>Continuous distal outcome, classes well separated</td>
    <td><span class="badge badge-accepted">ML</span></td>
  </tr>
  <tr>
    <td>Categorical outcome or covariate</td>
    <td><span class="badge badge-accepted">ML</span></td>
  </tr>
  <tr>
    <td>Low entropy, need stable class definitions</td>
    <td><span class="badge badge-accepted">BCH</span></td>
  </tr>
  <tr>
    <td>Naive modal or proportional assignment</td>
    <td><span class="badge badge-rejected">Avoid</span></td>
  </tr>
</tbody>
</table>

---

## Frequently asked questions

<div class="faq">

<h3>What is the three-step problem in latent class analysis?</h3>
<p>The three-step approach fits a latent class model, assigns each observation to a class, then relates those classes to outcomes or predictors. The problem arises between the second and third steps: the assigned class labels are treated as if they were observed without error, when they are actually estimates. Ignoring this classification error biases the Step-3 estimates toward zero, diluting real differences between classes.</p>

<h3>How much bias does the naive three-step approach introduce?</h3>
<p>With moderate entropy, the attenuation typically falls in the 20–45% range, meaning genuine class differences can be understated by up to nearly half. The bias shrinks as entropy (classification certainty) approaches 1 and grows as classes become harder to separate.</p>

<h3>How do you correct the bias — BCH or ML?</h3>
<p>Two bias-adjusted methods fix it by carrying the classification error into Step 3. Use BCH for continuous distal outcomes when normality is uncertain or when you need stable class definitions at low entropy; use the ML (maximum-likelihood) correction for categorical outcomes and covariates, or for continuous outcomes when classes are well separated. In Stata, both are available through the <code>step3</code> command.</p>

</div>

---

[^1]: This is biased as well. See [Vermunt (2010)](https://doi.org/10.1093/pan/mpq025).

[^2]: Entropy is a summary of classification quality. High entropy means most posterior probabilities cluster near 0 or 1; low entropy means they spread across classes. The 20–45% bias range assumes moderate entropy. When entropy approaches 1 the bias is negligible.

[^3]: Never leave your unlocked laptop unattended. Or do, and then write a methods paper about it.

---

**Cite as:** Califano, G. & Fabbricatore, R. (2026). Relating latent class membership to covariates and outcomes: Two bias-adjusted methods in Stata. *Stata Journal, 26*(2), 153–176. <https://doi.org/10.1177/1536867X261449931>

---

<script type="module">
/**
 * Load highlight.js with the Stata language and apply it to
 * all .hljs-stata-code blocks that Rouge left unstyled.
 */
import hljs from 'https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/es/highlight.min.js';
import stata from 'https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/es/languages/stata.min.js';

hljs.registerLanguage('stata', stata);

document.querySelectorAll('code.hljs-stata-code').forEach(el => {
  el.removeAttribute('data-highlighted');
  hljs.highlightElement(el);
});
</script>
