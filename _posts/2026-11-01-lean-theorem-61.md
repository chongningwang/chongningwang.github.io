---
layout: post
title: "Formalizing a Counterexample in Lean, and the Dualizing-Sheaf Detour"
date: 2026-11-01 20:00:00 +0800
description: An AI agent formalized Theorem 6.1 of our paper in Lean 4. The first run took days because it was building the general theory of the dualizing sheaf; with a concrete definition it finished within a day.
tags: lean ai
categories: blog
---

Theorem 6.1 of our paper with Lei Zhang,
[_Geometric singularities of regular surfaces with nef anti-canonical divisors over imperfect fields_](https://arxiv.org/abs/2604.05293v3),
is now checked in Lean 4. The formalization, built on mathlib, is public at
[chongningwang/geo-nonred-K-trivial-surface-char5](https://github.com/chongningwang/geo-nonred-K-trivial-surface-char5).
The Lean work was done by AI coding agents. This post records what Lean now checks, and one lesson from getting there: the
first run took far longer than expected, because the agent was building the general theory of the dualizing sheaf.

## The theorem

> **Theorem 6.1.** Let $$\ell = \mathbb F_5(s_1, s_2)$$. Then there exists a projective regular surface $$S$$ over $$\ell$$ with
> $$H^0(S, \mathcal O_S) = \ell$$ and $$K_S \equiv 0$$ such that $$S$$ is geometrically non-reduced over $$\ell$$.

The theorem is a counterexample. It shows that the bound $$p \geq 7$$ in our main result is optimal. The surface is constructed
explicitly: from affine charts, an invariant quotient by a $$p$$-closed derivation, and a contraction.

## What Lean checks

The main theorem gives, over $$k = \mathbb F_5(s_1,s_2)$$, a scheme $$Y$$ and a morphism $$f \colon Y \to \mathrm{Spec}\,k$$ with
these properties:

- $$f$$ is proper;
- $$Y$$ is integral, noetherian, of dimension 2, and all its local rings are regular;
- $$H^0(Y, \mathcal O_Y) = k$$, and $$Y$$ is not geometrically reduced;
- the sheaf of absolute differentials $$\Omega_{Y/\mathbb F_5}$$ is locally free of rank 4;
- the sheaf $$\omega = \mathcal{H}om\bigl(f^*\bigwedge^2\Omega_{k/\mathbb F_5}, \bigwedge^4\Omega_{Y/\mathbb F_5}\bigr)$$ is
  isomorphic to $$\mathcal O_Y$$, so it has degree 0 on every proper curve.

The only axioms used are Lean's standard three (`propext`, `Classical.choice`, `Quot.sound`), and there are no geometric
hypotheses. Two steps are not formalized:

- **Projectivity.** Lean proves only that $$f$$ is proper.
- **The comparison of $$\omega$$ with the canonical sheaf.** That $$\omega$$ is the canonical sheaf of $$Y$$ follows from the
  local freeness of $$\Omega_{Y/\mathbb F_5}$$ and the transitivity triangle of cotangent complexes for
  $$\mathbb F_5 \to k \to Y$$. This standard argument is on paper; the repository README spells it out.

## The detour: building the theory of the dualizing sheaf

The first run went on for about four days, with no end in sight. By the time I stopped it, shortly after midnight on 31 October, the
project had grown to 1,061 Lean files with more than 16,000 declarations, all checked. Yet the final statement, $$K_S \equiv 0$$,
was still open.

I asked the agent what it was doing. The answer was reasonable, and that was the problem. Lean needs a definition of the
canonical class, and mathlib does not have one. The agent had taken the most general definition available: the dualizing sheaf,
characterized by Serre duality. It was faithfully building the theory needed to recognize that sheaf. That meant internal
Hom and Ext sheaves of modules, injective resolutions, diagonal Ext groups and rigidity. Its own notes said it should not call
the explicit trivial line "canonical" before proving that recognition. Each piece was correct. Together they amounted to a
general theory of Grothendieck duality, far beyond the needs of one surface.

The fix was to give it a concrete definition that fits this surface. Our surface is regular, hence a local complete
intersection over $$k$$. For such schemes the canonical sheaf has a local description:
$$\mathcal{H}om\bigl(\bigwedge^{\mathrm{top}} I/I^2, \bigwedge^{\mathrm{top}}\Omega_P \otimes A\bigr)$$
for a smooth presentation $$A = P/I$$, glued along changes of presentation. With this definition, the agent dropped the duality
machinery, reused the determinant and volume computations it had already made, and finished within the same day.

Afterwards the definition was simplified once more, to the explicit formula for $$\omega$$ above. Since the surface lives in
characteristic $$p$$, the absolute differentials $$\Omega_{Y/\mathbb F_5}$$ already form a vector bundle of rank 4. The
determinant of the base, $$\bigwedge^2\Omega_{k/\mathbb F_5}$$, normalizes them, so that no presentation is needed at all.

## Cleaning up

Once the theorem was proved, most of the code turned out to be unnecessary. The earlier duality machinery was deleted, along
with unused declarations, redundant imports and build warnings. The rest was regrouped into a layout closer to mathlib's. The
public version has 42 files and about 21,000 lines in four layers:

- general mathematics: exterior powers, regularity, Kähler differentials, $$p$$-closed derivations, sheaves of differentials;
- the construction of the surface;
- the computation that $$\omega \cong \mathcal O_Y$$;
- the statement.

A verification script builds everything and audits the axioms of every declaration. It also compares a fingerprint of the
statement with a pinned value. This guards against a later edit, by a person or an agent, silently weakening what is proved.

## Lessons

- **An agent takes your definitions literally.** Asked for "the canonical class", it reached for the most general
  characterization and started to build its theory. The theorem only needed a concrete definition that is correct for this
  surface. Choosing that definition is the mathematician's job. It is a decision, not an implementation detail.
- **When a run takes much longer than expected, ask what it is building.** The agent was not stuck; it was making steady
  progress on the wrong problem. One question exposed this, and changing the target saved far more time than any speed-up would
  have.
- **Be explicit about what is not formalized.** Moving the comparison with the dualizing sheaf from Lean back to the paper is
  fine, provided the README says so clearly. A formal proof is only as convincing as the statement it proves.
