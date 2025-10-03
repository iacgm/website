---
date: '2025-05-15'
draft: false
title: 'Building a Program Synthesizer'
---

Program synthesis is an incredibly general, mostly futile, and deeply fascinating endeavor. 

As the name implies, it's the study of programs which generate (or synthesize) other programs, usually after having been given some formal requirements or sample data to draw from.

Last year, as part of my university coursework, I decided to build an enumerative program synthesizer (see [here](https://github.com/iacgm/kolmogorov)). That is, one which works via naive trial-and-error, possibly informed by programming language semantics and/or statistical guidance. 

In many settings, the search space of possible computer programs is too massive for this to work well (with some famous exceptions, such as Excel's autofill feature), but I took a simple idea and pushed it to an extreme, with more success than I had anticipated. 

The tool I built was an incredibly fun toy, and it let me play with different kinds of programming languages and see what they're capable of, which is often more than you might expect.

If this sounds interesting to you, I encourage you to skim [my report](/synthesizer_report.pdf) (or even to read it, at your own risk), especially chapters 3-6, but you can probably drop in and out at-will without getting too lost. It's shorter than it seems, and you might learn something new.
