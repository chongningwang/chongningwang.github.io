---
layout: page
title: Classification of geometrically non-regular regular K-trivial projective surfaces
description: Geometric integrality, normality and smoothness over imperfect fields, canonical torsion, and explicit examples.
importance: 1
category: research
---

This project studies the classification of regular projective surfaces with numerically trivial canonical class whose geometric base change fails to be regular.
Let $$X$$ be a regular projective surface over a field $$k$$ of characteristic $$p > 0$$ with $$H^0(X, \mathcal O_X) = k$$ and $$K_X \equiv 0$$.
When $$k$$ is imperfect, the base change $$X_{\bar k}$$ may be non-normal, or even non-reduced.

The [GitHub repository](https://github.com/chongningwang/regular-surfaces-over-imperfect-fields) collects the papers, PDFs and computational verification scripts.
The [Lean code](https://github.com/chongningwang/regular-surfaces-over-imperfect-fields/tree/main/lean) treats the geometrically non-reduced example in characteristic five.

**[Geometric singularities of regular surfaces with nef anti-canonical divisors over imperfect fields](https://arxiv.org/abs/2604.05293)**, with Lei Zhang.
We show that $$X$$ is geometrically integral if $$p \geq 7$$ and construct a geometrically non-reduced example in characteristic five, showing that the bound is optimal.
This paper is available on arXiv.

**[Effective nonvanishing of $$K$$-trivial regular surfaces over imperfect fields](https://arxiv.org/abs/2610.09542)**, with Jingshan Chen and Lei Zhang.
We prove the optimal uniform bound $$60K_X \sim 0$$, with sharper bounds according to the characteristic and geometric properties of $$X$$.
An example with canonical divisor of exact order five shows that the factor five is necessary.
This paper is available on arXiv.

**Geometric singularities of $$K$$-trivial regular surfaces.**
The [draft](https://github.com/chongningwang/regular-surfaces-over-imperfect-fields/blob/main/papers/geonormal/main.pdf) gives the sharp bounds
$$p \geq 13$$ for geometric normality and $$p \geq 23$$ for smoothness over $$k$$.
There is an example over $$\mathbb F_{11}(c)$$ that is not geometrically normal, and for $$p \in \{13, 17, 19\}$$ the surface
$$X_{\bar k}$$ is either smooth or has exactly one singular point, of type $$\mathrm A_{p-1}$$, where the singular case does occur.
This manuscript is a work in progress: I am still checking and revising its AI-assisted proofs.

**Frobenius length of $$K$$-trivial regular surfaces in positive characteristic.**
A further manuscript on Frobenius length is in preparation.
The Frobenius length $$\ell_F(X/k)$$, in the sense of Tanaka, is the smallest $$e$$ such that after replacing $$k$$ by $$k^{1/p^e}$$ and $$X$$ by the
normalization of its reduced base change, the surface becomes geometrically normal.
We bound it by a constant depending only on $$p$$, independently of $$k$$: $$\ell_F(X/k) \leq 1$$ for $$p \in \{5, 7, 11\}$$, which is sharp, with
explicit bounds for $$p = 2$$ and $$3$$. We also bound the corresponding length for geometric reducedness.
