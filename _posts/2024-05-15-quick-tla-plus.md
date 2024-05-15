---
layout: post
title:  "A Short Review of TLA+ and Model Checking (Ongoing)"
date:   2024-05-15 20:00:00 -0400
categories: jekyll update
---

TLA+ is a tool to run model checking. A *model* is a finite state automaton (FSA) plus a function that colorizes each state with different colors. A trace of the FSA is thus viewed as a sequence of color sets, called a *color sequence* in this post. A *temporal property* is a set of color sequences. In *model checking*, given a model and a temporal property, our task is to either assure all traces belong to the temporal property (i.e., all traces are printed properly), or find an outlier trace.

The algorithm of model checking is essentially to enumerate and check each trace. Since the number of traces grows exponentially with the number of states, exhaustive enumeration becomes infeasible for large models.

TODO: detail the model checker algorithm and temporal properties  
TODO: discuss the usage of induction in model checking  
TODO: add a running example with some pictures