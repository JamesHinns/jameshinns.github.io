---
layout: post
title: "What is counterfactual cherry-picking and how can we detect it?"
date: 2026-06-24
permalink: /blog/cherry-picking/
description: ""
---

<p style="border-left: 3px solid var(--global-theme-color); padding-left: 0.85rem; margin: 0 0 1.75rem; color: var(--global-text-color-light);">
  This post accompanies our paper <em>On the Definition and Detection of Cherry-Picking in Counterfactual Explanations</em>, which will appear in <em>Data Mining and Knowledge Discovery</em> and be presented at <em>ECML PKDD 2026</em>. You can already read the <a href="https://arxiv.org/abs/2601.04977">pre-print</a>.
</p>

A counterfactual explanation identifies a minimal change to an instance that would change the model’s prediction {% cite martens2014explaining --file blog-references %}.

<div class="note note-info" style="margin: 1.5rem 0;">
  <strong>Counterfactual explanation example for a rejected loan application</strong><br>
  “If your income had been €10,000 higher, your application would have been accepted.”
</div>

The €10,000 increase is one change that would lead to the application being accepted, but the definition leaves “minimal” open to interpretation. In practice, judging whether one counterfactual is preferable to another requires some way of comparing them, and in the paper we focus on three such measures. Sparsity counts how many features are changed, proximity measures the distance from the original instance, and plausibility assesses how well the counterfactual fits patterns in the training data. These measures may guide the search for counterfactuals, restrict which candidates are allowed, or be used afterwards to choose between the counterfactuals that were found.

There are often many valid counterfactuals. A higher income might change the loan decision, but so might a longer employment history, a smaller requested loan, or some combination of changes. Different methods, parameter settings, and random seeds can all produce different explanations. This is part of the broader **disagreement problem** in counterfactual explanations {% cite brughmans2024disagreement --file blog-references %}.

<img src="/assets/img/blog/pcherry/cf_example.svg" alt="Several possible counterfactual explanations for the same instance." style="display: block; width: 100%; height: auto; margin: 1.5rem auto 0.5rem;">

*One decision can have several valid counterfactual explanations. Which one is “best” depends on how the explanation specification ranks them.*

Different people prioritise different aspects of an explanation, and different uses call for different kinds of counterfactual, so this variety can be useful. However, because counterfactual explanations are often presented one at a time, the selection process matters. An auditor needs to know whether the returned counterfactual was in fact the top-ranked option under the procedure the provider claims to have used, or whether another counterfactual should have been returned instead.

## A definition of cherry-picking

Imagine Alice was denied a loan by an automated system used by a bank. The bank wants to explain why the model rejected her application, and it reports one counterfactual to her. It could return either of the following explanations.

- “If you earned €10,000 more and had worked for two more years, your application would have been accepted.”
- “If your recorded gender had been male, your application would have been accepted.”

The first explanation is likely more actionable. The second raises a more serious concern, because it suggests that the model’s decision can be changed by altering a protected attribute that should not affect access to credit.

Suppose the lender claims to rank explanations by sparsity. Under that ranking, the gender-based explanation should be returned, since it changes only one feature. If the lender nevertheless presents the income-and-employment explanation to hide the model’s reliance on gender, that is cherry-picking.

We call everything that went into choosing the returned counterfactual the **explanation specification**. This includes the data and models, counterfactual generation methods, constraints, parameter settings, random realisations, utility function, and rule used to break ties.

Whether an explanation has been cherry-picked can only be judged against this specification. The same counterfactual may rank first under one specification, rank lower under another, or not be admissible under another at all. The specification determines both which explanations are possible and how they are ranked.

For an instance $x$, the **admissible explanation space** contains every possible explanation under the models and methods included in the specification.

$$
\mathcal{E}_{\mathcal{F},\mathcal{A}}(x)
= \{A_f(x) \mid f \in \mathcal{F},\ A \in \mathcal{A}\}.
$$

Here, $\mathcal{F}$ is the set of models considered and $\mathcal{A}$ is the set of explanation methods considered. A fixed realisation of any stochastic model or method is represented by a separate element of $\mathcal{F}$ or $\mathcal{A}$. The utility and tie-breaking rule induce a strict ranking over the admissible explanation space. A reported explanation is **cherry-picked if it is not the top-ranked admissible explanation under the real specification**.

This definition does not say whether the explanation specification is itself sound. A returned counterfactual might be top-ranked under an unsound specification, just as a counterfactual might be cherry-picked under a defensible one. Soundness is a separate question about whether the models, methods, constraints, and ranking rule are appropriate for the context. Cherry-picking concerns whether the returned counterfactual is the one that should have been returned under the true specification.

The practical problem is that manipulation does not require a false explanation. A provider may only need to search through many valid explanations until it finds one that supports the story they want to tell {% cite goethals2023manipulation --file blog-references %}.

## How can we detect cherry-picking?

Detection depends on what the auditor can observe.

If an auditor is given a specification containing all its components, they can reconstruct the admissible explanation space implied by that specification, apply the disclosed ranking, and verify whether the returned explanation is top-ranked.

The figure below illustrates this using one of our DiCE experiments {% cite mothilal2020explaining --file blog-references %}. In this case, the disclosed ranking rule selected the counterfactual with the lowest proximity score. We compared this with an alternative rule that ranked every explanation without a gender change above those with one, and used proximity only to break ties within each group. The blue points are the proximity-optimal explanations from the generated set, while the orange points are the explanations selected by the alternative rule. Where an orange point has a higher proximity score than the corresponding blue point, the selected explanation is not top-ranked under the disclosed rule and the cherry-picking would be detectable. This occurred for 4 of the 10 audited instances.

<img src="/assets/img/blog/pcherry/optimal-vs-cherry-picked-proximity.svg" alt="Proximity of optimal and cherry-picked explanations." style="display: block; width: 100%; height: auto; margin: 1.5rem auto 0.5rem;">

*The blue points are optimal under the disclosed proximity ranking. The orange points were selected using an alternative rule that prioritised avoiding gender changes.*

The figure above shows what verification can establish when the specification is provided. An auditor can check whether the stated specification returns the explanation that was presented. This verifies that the stated specification produces the given explanation, but it cannot show that the stated specification is the true specification. The provider may have explored other models, methods, seeds, constraints, utilities, or ranking rules before deciding what to reveal.

This is where **omittance** becomes central. Cherry-picking concerns the explanation that is returned, while omittance concerns the specification that is disclosed. It occurs when the disclosed specification does not match the true specification because components that influenced the provider’s choice were omitted. A provider could consider many models, methods, seeds, constraints, and utilities, then disclose only the specification under which its preferred explanation comes first. Since the true specification governs whether cherry-picking occurred, but an auditor sees only the disclosed specification, the omitted components are precisely what make the manipulation difficult to detect.

With partial procedural access, some consistency checks remain possible, but the auditor may be unable to reconstruct the admissible explanation space. With explanation-only access, there is usually no reliable way to know which alternatives were possible, let alone how they should have been ranked.

## Why quality measures are not enough

If the selection process cannot be fully observed, a natural fallback is to examine the explanations themselves. One might assume cherry-picked explanations should look suspicious under familiar quality measures. They might be less sparse, less plausible, or farther from the original instance than explanations produced by an ordinary run.

The experiments suggest that this is unreliable. Ordinary choices, such as changing a random seed, can create as much variation as deliberately restricting which features may be changed. A cherry-picked result can therefore sit comfortably among apparently normal runs.

<img src="/assets/img/blog/pcherry/proximity-plausibility-seed-variation.svg" alt="Variation in proximity and plausibility across seeds and feature restrictions." style="display: block; width: 100%; height: auto; margin: 1.5rem auto 0.5rem;">

*Variation caused by seeds and generation settings overlaps with the effect of restricting sensitive-feature changes, making outcome-based detection difficult.*

More surprisingly, cherry-picking does not always make familiar quality scores worse. Restricting a heuristic search can occasionally steer it towards a counterfactual with better sparsity or proximity, even while hiding behaviour that an auditor would want to see. A good metric score is not proof of an honest selection process.

## From detecting outcomes to constraining the process

The quality-measure results point to a more general problem. Cherry-picking can look ordinary under common quality measures, because variation introduced by seeds, constraints, methods, and rankings can overlap with the variation introduced by cherry-picking. Once a provider has enough flexibility over these choices, the final counterfactual can appear reasonable even when it was cherry-picked to hide behaviour that another admissible explanation would have revealed.

Without stronger standards for generating and reporting counterfactual explanations, this flexibility makes cherry-picking easy to carry out and difficult to detect. A provider does not need to invent a counterfactual or return one with obviously poor quality scores. It may only need to search through enough admissible explanations and report the one that presents the model in the most favourable light. In that setting, counterfactual explanations can be used to mask model behaviour rather than reveal it.

A practical way forward is to reduce these hidden degrees of freedom before explanations are generated and reported.

* Publish the complete explanation specification, including the utility and tie-breaking rule.
* Fix or pre-declare random seeds and use evaluation protocols that aggregate across seeds.
* Report feasibility constraints, parameter settings, and computational budgets.
* Evaluate explanations across multiple runs instead of presenting one convenient result.
* Establish domain-specific standards for how counterfactuals should be generated and selected.

These safeguards do not make cherry-picking impossible. They make it harder to quietly explore a large collection of explanations and reveal only the most convenient one.

Counterfactual explanations are useful because they can present model decisions through concrete alternatives. However, a simple reported explanation can hide a complicated selection process. The important questions are therefore which alternatives were admissible, how they were ranked, whether the specification is sound, and whether the reported explanation came first.

## References

<div class="publications">
{% bibliography --file blog-references --group_by none --cited_in_order %}
</div>
