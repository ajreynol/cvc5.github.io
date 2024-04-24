---
layout: blog-post
categories: blog
excerpt_separator: <!--more-->
title: "Interfaces for Understanding cvc5"
author: Andrew Reynolds
date: 2024-04-15
---

This blog post focuses on new interfaces for understanding the behavior of cvc5 and what to do when things go right and more importantly when they go wrong.
We review the numerous diagnotic features of cvc5 and give tips on what to do when you are stuck.

<!--more-->

## The Information Gap between SMT solvers and their Users

State-of-the-art SMT solvers like cvc5 can be used to solve constraints in a growing number of application domains.
These applications often rely on constraints in undecidable logics, for example logics with quantified formulas, non-linear arithmetic and strings.
While there are no termination guarantees for such constraints, SMT solvers are nevertheless surprisingly effective at solving constraints in these logics.

In a perfect world, SMT solvers can be seen as black boxes that reliably respond to every user query in a reasonably small amount of time.
However, this is far from a guarantee due to the increased complexity of constraints handled by SMT solver
The solver may return "unknown", or depending on the patience of the user may be perceived as "going off into the woods".
It can be highly frustrating to users, since seemingly no information can be extracted when the solver times out.
In fact, it has been said that users of SMT solvers spend a *majority* of their time dealing with the case where the solver is unsuccessful.

This blog post focuses on improving the user experience of SMT solvers.
In particular, we focus on bridging the information gap between users and the internals of cvc5, particularly for when it is *unable* to solve a user query in a reasonable amount of time.

Many of the features explained in this post were originally used by cvc5 developers for purposes of internal debugging.
As these features matured, many were polished so that users could understand and benefit from them.
We hope this post will foster further dialogue between users and developers of cvc5.
In particular, we welcome any further suggestions for diagnostic information that would benefit your application.

In this blog post, we first recap the standard SMT-LIB interfaces for how to interact with the SMT solver when things go right, and then turn our attention to when things go wrong.
In the following, we frequently reference:
- SMT-LIB version 2.6 commands in the text interface (e.g. `(get-unsat-core)`).
- Command line options (e.g. `produce-unsat-cores`).
- Output tags, which are a family of command line options (e.g. `-o unsat-core-benchmark`). These do not impact the behavior of the solver apart from printing diagnostic information. 
For more details, see our documentation of [output tags](https://cvc5.github.io/docs-ci/docs-main/output-tags.html).

While the discussion will focus on the command line interface and *.smt2 text interface, most of the features mentioned in this post are available in each of our APIs (C++, Java, Python).
Many of these features are new and are under active development.
All features are available in the latest development version of cvc5 on its [main branch](https://github.com/cvc5/cvc5).

## Interfaces for when things go right

### Unsat cores

cvc5, like most SMT solvers, supports the SMT-LIB standard command `(get-unsat-core)`, which returns a subset of the user assertions that are sufficient for showing "unsat".
This may help narrow down what the relevant portion of the problem was that led to the refutation.
In cvc5, unsat cores are available immediately after an "unsat" response and when the option `produce-unsat-cores` is enabled.

```
% cat test.smt2
(set-logic ALL)
(set-option :produce-unsat-cores true)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (! (and (> x 2) (< x 0)) :named A))
(assert (! (< y 0) :named B))
(check-sat)
(get-unsat-core)

% cvc5 test.smt2
unsat
(
A
)
```

We support additional options pertaining to the way unsat cores are computed and printed:
- `minimal-unsat-cores`, which uses a greedy algorithm to drop assertions from the unsat core.
This option may induce additional performance overhead, but is useful if the user prioritizes smaller unsat cores.
The cores returned from this option are locally minimal (in that dropping any formula from the unsat core does not lead to an "unsat" response), although they are not guaranteed to be globally minimal.
- `print-cores-full`, which makes unsat cores can also be made agnostic to the smt2 attribute `:named`.
- `dump-unsat-cores`, which issues a command to print the unsat core automatically after every unsatisfiable response.
- `-o unsat-core-benchmark`, which prints the unsat core as a standalone benchmark.
- `check-unsat-cores`, which internally double checks the correctness of the given unsat core.

cvc5 additionally supports more fine-grained variants of unsatisfiable cores which we describe in the following.

#### Unsat core of lemmas

cvc5 supports a custom SMT-LIB command `(get-unsat-core-lemmas)` which prints the theory lemmas that were relevant to showing unsatisfiable.
This is a list of valid formulas that were generated internally by theory solvers involving atoms not necessarily from the user input.
The lemmas used in the unsat core are available immediately after an "unsat" response and only when proofs are enabled (command line option `produce-proofs`).

```
% cat test.smt2
(set-logic ALL)
(set-option :produce-proofs true)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (! (and (> x 2) (< x 0)) :named A))
(assert (! (< y 0) :named B))
(check-sat)
(get-unsat-core-lemmas)

% cvc5 test.smt2
unsat
(
(or (not (>= x 3)) (>= x 0))
)
```

In the above example, a single theory lemma was generated which was required for showing the first assertion to be unsatisfiable.
Note that the literals `(>= x 3)` and `(>= x 0)` in this lemma correspond to the original two atoms after preprocessing.
The preprocessed form of the input is also available, which we will describe later in this post.

Theory lemmas may involve symbols that were introduced internally by cvc5 during solving, which we call "Skolems".
A classic example is the "division by zero" Skolem introduced to reason about the possibility of division with a zero denominator.
For details on all the documented Skolems cvc5 supports, see our documentation for [Skolem identifiers](https://cvc5.github.io/docs-ci/docs-main/skolem-ids.html).

We also support dumping the unsat core along with the lemmas used as a standalone benchmark via the output flag `-o unsat-core-lemmas-benchmark`.

#### Unsat core of instantiations

As a further refinement, one can ask specifically the shape of the quantifier instantiation lemmas used by cvc5 when solving a query.
These are available via the command line option `dump-instantiations`.
When proofs are enabled, this list is refined to only include the instantiations that were used in the proof of unsatisfiability.

```
% cat test.smt2
(set-logic UFLIA)
(declare-fun P (Int) Bool)
(assert (forall ((x Int)) (P x)))
(assert (or (not (P 3)) (not (P 4))))
(check-sat)

% cvc5 test.smt2 --dump-instantiations
unsat
(instantiations (forall ((x Int)) (P x))
  ( 3 )
  ( 4 )
)
```

A further refinement to this feature prints the "source" of the instantiation lemma, which is a unique identifier (which we call an "inference identifier") that indicates which part of cvc5's code base was responsible for the lemma.
This is available by the command line option `dump-instantiations-debug`.
In the future, we are planning to export more information about the meaning of the inference identifiers.

### Proofs

When cvc5 answers "unsat", a fine-grained account of its reasoning can be obtained via the SMT-LIB command `(get-proof)`.
Analogous to unsat cores, this requires the option `produce-proofs` to be enabled.
This section gives a cursory overview of the interface for getting proofs.

A proof is a step-by-step account of how a refutation can be derived from the input.
For documentation on the proof rules supported by cvc5, see documentation of our [proof rules](https://cvc5.github.io/docs-ci/docs-main/proofs/proof_rules.html).

The following outputs control how proofs are computed and printed:
- `proof-granularity=X` controls the granularity of the generated proof. This can range from allowing large informal "macro" steps to requiring each small-step theory rewrite to be justified.
- `check-proofs`, which runs an internal proof checker in cvc5 on the final proof.
- `proof-format=X` which impacts the format of the proof. By default, cvc5 generates proofs in the Alethe LF format, which is based on the SMT-LIB version 3.0 proposal. For details on generating Alethe LF proofs in cvc5, see [AletheLF](https://cvc5.github.io/docs-ci/docs-main/proofs/output_alf.html).
- `dump-proofs`, which issues a command to the get the proof automatically after every unsatisfiable response.

cvc5 additionally provides interfaces for getting only part of the entire proof.
In particular, our `get-proof` command takes an optional "component" identifier, indicating the part of the proof that the user is interested in.
For instance, the user can ask for only the proofs of theory lemmas with the command `(get-proof :theory_lemmas)`.
More details on proof components can be found [here](https://cvc5.github.io/docs-ci/docs-main/api/cpp/modes.html#_CPPv4N4cvc55modes14ProofComponentE).

Proofs can be checking externally using the proof checker `alfc` for the Alethe LF format (see this [user manual](https://github.com/cvc5/alfc/blob/main/user_manual.md) for details).
We provide a [script](https://github.com/cvc5/cvc5/blob/main/contrib/get-alf-checker) for getting started with this proof checker.
Further efforts are in progress towards exporting proofs to trustworthy proof assistants like Lean or Isabelle.

### Models

cvc5 supports models via the SMT-LIB command `(get-model)` which is available immediately after a "sat" response.
This requires the option `produce-models` to be set to true.
Again, analogous to other features `dump-models` can be used to issue a command to print the model after every satisfiable response.
Models can be double checked for correctness internally using the option `check-models`.

#### Model cores

cvc5 supports a refinement to models where only the variables that were relevant for satisfying the user input.
We call this a "model core".
```
% cat test.smt2
(set-logic QF_UFLIA)
(set-option :produce-models true)
(declare-fun x () Int)
(declare-fun y () Int)
(declare-fun z () Int)
(assert (or (> z 5) (> y 5)))
(check-sat)
(get-model)

% cvc5 test.smt2 --model-cores=simple
sat
(
(define-fun y () Int 6)
)
```

In detail, a model core is an interpretation of a subset of the user variables where all extensions of that interpretation are also a model.
In the above example, if `y` is assigned the value `6`, then the input assertion evaluates to true no matter what the value of `x` or `z`.

## Interfaces for when things go wrong

Now that we have seen the information that is available after the SMT solver answers, consider the case where the solver times out or answers unknown.
Unfortunately, users are often unaware of how to obtain useful information in such cases.
In the following, we first focus on information that documents what the solver is doing. 
We then focus on newer features that are tailored towards aiding the user in assessing what went wrong, and in some cases, how to fix it.

### Timeout cores

cvc5 has experimental support for SMT-LIB command `(get-timeout-core)`.
A "timeout core" is a subset of the user input that is sufficient for making cvc5 timeout in a given timeout (configurable by `timeout-core-limit=X`).
It is generally most useful for undecidable quantifier-free logics such as non-linear arithmetic or strings.

```
% cat test.smt2
(set-logic ALL)
(set-option :produce-unsat-cores true)
(declare-fun x () Int)
(declare-fun w () Int)
(declare-fun u () Int)
(assert (! (> u 0) :named u0))
(assert (! (< w 0) :named w0))
(assert (! (= (* x x x) 564838384999) :named hard))
(get-timeout-core)

% cvc5 test.smt2
unknown
(
hard
)
```

Unlike other interfaces, timeout cores are *only* available when using the extended SMT-LIB command `get-timeout-core`.
In other words, timeout cores are not impacted by previous calls to `check-sat`.

The expected result of a `get-timeout-core` is "unknown" followed by a list of formulas corresponding to a timeout core.
However, it may be the case that cvc5 stumbles upon a model or is able to prove unsat.
In these cases, it may respond "sat" or "unsat" to a `get-timeout-core` command.
In the latter case of "unsat", it will report the unsatisfiable core that it computed, which was a candidate timeout core that happened to be solvable "unsat" within the given timeout.

The following options are available which impact how timeout cores are computed and printed:
- `timeout-core-limit=X`, which impacts the timeout (in milliseconds) that the core is based on.
- `print-cores-full`, which analogous to unsat cores makes cvc5 agnostic to named assertions.

cvc5 additionally supports a variant of the timeout core command where assumptions are provided.
In particular, the command `(get-timeout-core-assuming (a1 ... an))` asks cvc5 to find a subset of the formulas `a1, ..., an` that when combined with the input assertions cause a timeout.
This is helpful if the user wants to focus on a particular subset of the input while implicitly assuming that all other assertions are part of the timeout core. 

The algorithm for computing timeout cores tries to build subsets of the given assertions until one is either solved or times out.
It may be the case that the timeout core itself takes a long time to compute, in particular
if it tries many different subsets of the input all of which are "sat" within the timeout.

### Difficulty

cvc5 also supports an experimental custom SMT-LIB command `(get-difficulty)`, which returns a mapping from user assertions to natural number values, where the higher the value, the more likely the assertion was the cause of the higher runtime of cvc5.

```
% cat temp.smt2
(set-logic ALL)
(set-option :finite-model-find true)
(set-option :fmf-mbqi none)
(set-option :produce-difficulty true)
(declare-sort U 0)
(declare-fun a () U)
(declare-fun b () U)
(declare-fun c () U)
(declare-fun P (U U) Bool)
(declare-fun Q (U) Bool)
(assert (distinct a b c))
(assert (forall ((x U) (y U)) (P x y)))
(assert (forall ((x U)) (Q x)))
(check-sat)
(get-difficulty)

% cvc5 temp.smt2
sat
(
((distinct a b c) 16)
((forall ((x U) (y U)) (P x y)) 9)
((forall ((x U)) (Q x)) 3)
)
```

The command `get-difficulty` can be used to get the difficulty mapping after *any* response ("unsat", "sat" or "unknown") to a satisfiability query.
It requires that the option `produce-difficulty` is enabled.

In the above example, we declare an uninterpreted sort `U` with (at least) three distinct elements `a,b,c`.
We then assert two quantified formulas over `U`.
In this example, we configure `cvc5` to perform exhaustive instantiation based on finite model finding.
The difficulty measurement is tracked for all three assertions, which is printed in response to `get-difficulty`.
For the latter two, we can see that the first quantified formula was assigned a difficulty of `9`, which in this case corresponds directly to the number of instantiation lemmas that were required for expand its definition, whereas the second was assigned a difficulty of `3`.

The difficulty of a user assertion can be measured based on several criteria which is configurable via the option `difficulty-mode=X`.
By default, the difficulty measurement roughly corresponds to clause activity, where if literals from a user input participate in a lemma or conflict, the difficulty of the assertion is incremented by one.

### Incompleteness

When cvc5 answers "unknown", the user may ask for an explanation of why cvc5 gave up using the output tag `-o incomplete`.
Reasons can include resource limiting, incomplete heuristics for quantifier instantiation, unsupported combinations of theories, options misconfiguration, and so on.

```
% cat temp.smt2
(set-logic ALL)
(declare-fun f (Int) Int)
(assert (forall ((x Int)) (= (f x) (* x (f (- x 1))))))
(check-sat)

% cvc5 temp.smt2
(incomplete INCOMPLETE QUANTIFIERS)
unknown
```

In the above example, cvc5 gave up because it could not determine the satisfiability of the given quantified formula.
The first identifier `INCOMPLETE` captures the high-level reason for why "unknown" was returned, which is an [unknown explanation](https://cvc5.github.io/docs-ci/docs-main/api/cpp/unknownexplanation.html#unknownexplanation).
This is a high-level categorization of why cvc5 answered unknown, which may be due to theory solvers being incomplete, resource or time limits being reached, or whether the user specified an option that make cvc5 intentionally answer unknown (e.g. `--preprocess-only`).
The second identifier is a finer grained internal explanation of why the theory solvers were incomplete.
Note these identifiers are not currently part of our API, but are documented internally [here](https://github.com/cvc5/cvc5/blob/main/src/theory/incomplete_id.h).
In the above example, the second identifier `QUANTIFIERS` indicates that cvc5 gave up due to the presence of quantified formulas.
Indeed, in the above example, cvc5 is unable to determine a model for `f`.

### More Information to Understand what the Solver is Doing

#### Statistics

The most basic form of information one can retrieve from cvc5 is its statistics.
These are dumped when cvc5 terminates when the option `stats` is enabled.
A more detailed account of statistics, including timing information and histograms of the inference identifiers used during solving is available when the option `stats-internal` is enabled.
An example output is shown below:

```
% cat test.smt2
(set-logic ALL)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (and (> x 2) (< x 0)))
(assert (> (+ y 1) y))
(check-sat)

% cvc5 test.smt2 --stats 
unsat
driver::filename = test.smt2
global::totalTime = 5ms
theory::arith::inferencesLemma = { ARITH_UNATE: 1 }
cvc5::CONSTANT = { integer type: 2 }
cvc5::TERM = { AND: 1, GT: 1, LT: 2 }
cvc5::VARIABLE = { Boolean type: 1 }
```

The statistics output gives various information about the file solved, the global time, the set of inferences used to solve the problem (as given by their *inference identifiers*), as well as the kinds of terms that were constructed.

#### Diagnostics for Preprocessing

cvc5 implements a number of preprocessing passes that are applied to the input formulas prior to solving.
It is often useful to see how these passes modify the user input, prior to when the solving begins.
Let us revisit our earlier example:

```
% cat test.smt2
(set-logic ALL)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (and (> x 2) (< x 0)))
(assert (> (+ y 1) y))
(check-sat)

% cvc5 test.smt2 -o post-asserts
;; post-asserts start
(set-logic ALL)
(declare-fun y () Int)
(declare-fun x () Int)
(assert (>= x 3))
(assert (not (>= x 0)))
(assert true)
(assert true)
(check-sat)
;; post-asserts end
unsat
```

Here, we can see that cvc5 replaces strict inequalities with non-strict inequalities.
Furthermore, the constraint involving `y` was replaced by `true` since it holds in all models.
This output can be helpful for narrowing down what the exact form of an input assertion is after preprocessing.
It can also help contextualize further information that cvc5 provides, e.g. the lemmas that cvc5 provides will often be over atoms contained in the input after preprocessing.

The set of input assertions prior to preprocessing are also available via `-o pre-asserts`.
cvc5 also has other output tags that pertain to important things learned during preprocessing, 
for example, `-o subs` will output the substitutions corresponding to variable elimination inferred during preprocessing.
We also have support for "learned literals", which we describe next.

#### Learned Literals

cvc5 supports the custom SMT-LIB command `(get-learned-literals)`, which prints the set of unit clauses that were learned by cvc5 during preprocessing and solving.
It is available when the option `produce-learned-literals` is enabled.

```
% cat test.smt2
(set-logic ALL)
(set-option :produce-learned-literals true)
(declare-fun x () Int)
(declare-fun y () Int)
(declare-fun z () Int)
(assert (> x 5))
(assert (< y 4))
(assert (or (< x y) (>= z 1)))
(check-sat)
(get-learned-literals)

% cvc5 test.smt2
sat
(
(>= z 1)
(>= (+ x (* (- 1) y)) 0)
)
```
In particular, for this example, cvc5 learns that `x` must be greater than or equal `y`, and hence by the third constraint, `z` must be greater than or equal to one.

With `-o learned-lits`, cvc5 will print the literals it learns as they are learned.
For the above example, cvc5 prints:

```
% cvc5 test.smt2 -o learned-lits
(learned-lit (>= x 6) :preprocess)
(learned-lit (not (>= y 4)) :preprocess)
(learned-lit (>= (+ x (* (- 1) y)) 0) :input)
(learned-lit (>= z 1) :input)
sat
(
(>= z 1)
(>= (+ x (* (- 1) y)) 0)
)
```

Learned literals can be further classified based on whether the literal occurs in the original input problem (after preprocessing), and when it was learned.
In the above example, the first two literals were learned as part of preprocessing, whereas the latter two literals were input literals learned during solving.
By default, cvc5 will only report input literals that are not learned as part of preprocessing.
More documentation about the classification of learned literals is available [here](https://cvc5.github.io/docs-ci/docs-main/api/cpp/modes.html#_CPPv4N4cvc55modes14LearnedLitTypeE).

#### Candidate Models

cvc5 supports commands such as `get-model` after an "unknown" response, e.g. after reaching a time or resource limit.
The model given to the user reflects the most recent *candidate* model that cvc5 computed.
A candidate model is one that satisfies (decidable) quantifier-free constraints but could not be confirmed to satisfy quantified or undecidable constraints.
In a sense, the most recent candidate model typically indicates cvc5's best approximation of what a model of the constraints are.

#### Quantifier Triggers and Instantiations

Problems with quantified formulas can be notoriously sensitive and difficult to debug.
cvc5 supports various output tags to understand how quantified formulas are been handled internally.
In particular, note the following output tags:
- `-o inst`: This tag prints the number of instantiations tried for each quantified formula.
- `-o trigger`: This tag prints the triggers chosen for quantified formulas via outputs of the form `(trigger <quantified formula> <trigger>)`. It also prints the quantified formulas for which *no* triggers could be inferred via `(no-trigger <quantified formula>)`. The latter can be indicative of a performance issue, since quantified formulas with no triggers cannot be handled by traditional quantifier instantiation techniques, i.e. E-matching.

Note that the formatting of quantified formula in the above output traces can be refined for clarity by assigning names to quantified formulas.
This can be accomplished by the SMT attribute `:qid`.

#### Lemmas

The main solving loop in SMT consists of a propositional SAT solver in combination with a set of theory solvers, the latter of which we call the "theory engine".
The latter sends a stream of theory lemmas to the SAT solver until the input and these lemmas is propositionally unsatisfiable, or a model is found.
The set of theory lemmas that are generated internally in cvc5 is available via `-o lemmas`.

```
% cat test.smt2
(set-logic ALL)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (and (> x 2) (< x 0)))
(assert (< y 0))
(check-sat)

% cvc5 test.smt2 -o lemmas
(lemma (or (not (>= x 3)) (>= x 0)) :source ARITH_UNATE)
unsat
```

In the above example, `-o lemmas` prints the set of lemmas that are generated when solving this problem.
In this case, a single lemma sufficed for showing the input is unsatisfiable, which was given the identifier `ARITH_UNATE`.

#### Diagnostics for Syntax-guided Synthesis

cvc5 supports other kinds of constraints beside satisfiability.
In particular, we support syntax-guided synthesis problems, for details see [CAV2015](https://homepage.divms.uiowa.edu/~ajreynol/cav15a.pdf) and [CAV2019](https://homepage.divms.uiowa.edu/~ajreynol/cav19b.pdf).
Our syntax-guided synthesis solver also supports diagnostic output flags, including `-o sygus` which output the list of candidate solutions as they are tried, `-o sygus-grammar` which outputs auto-generated grammars for functions-to-synthesize, `-o sygus-enumerator` outputs a summary of the enumeration strategy, and `-o sygus-sol-gterm` which indicates which production rules of a grammar were used in constructing the final solution.

#### Auto-configuration of Options

cvc5 has a vast array of configurable options that have accumulated over many years of development and research.
In an ideal world, users of cvc5 should not need to worry about which options to use, and trust that the default configuration of cvc5 will always do the optimal thing.
For this purpose, cvc5 automatically configures its options based on the provided logic, as well as resolving any dependencies between options and reporting if there was an incompatibility of options.
The configuration routine is run prior to solving, after which options are fixed for the remainder of the run.
The output tag `-o options-auto` prints which options were automatically configured, as well as the reason for why the value of the option was modified.

```
% cat test.smt2
(set-logic ALL)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (and (> x 2) (< x 0)))
(assert (< y 0))
(check-sat)

% cvc5 test.smt2 -o options-auto
(options-auto strings-exp true :reason "logic including strings")
(options-auto bv-propagate false :reason "bitblast solver")
(options-auto heuristic-pivots 5 :reason "logic")
(options-auto decision justification :reason "logic")
(options-auto cegqi true :reason "logic")
(options-auto dt-share-sel false :reason "quantified logic without SyGuS")
(options-auto minisat-simplification clause-elim :reason "non-basic logic")
```

Above, cvc5 lists the changes it made to its default options, along with a reason for why it made this change.
Note that in the original benchmark, the logic was set to `ALL`, indicating that any kind of constraint may be present in the input.
For this reason, certain options were configured to account for this possibility, including using a more conservative simplification policy in Minisat.
In contrast, if the user had set the logic to `QF_LIA`, the automatic configuration would be different:

```
% cat test.smt2
(set-logic QF_LIA)
(declare-fun x () Int)
(declare-fun y () Int)
(assert (and (> x 2) (< x 0)))
(assert (< y 0))
(check-sat)

% cvc5 test.smt2 -o options-auto
(options-auto bv-propagate false :reason "bitblast solver")
(options-auto arith-rewrite-equalities 1 :reason "logic")
(options-auto heuristic-pivots 5 :reason "logic")
(options-auto standard-effort-variable-order-pivots 200 :reason "logic")
(options-auto nl-ext-tplanes-interleave true :reason "pure integer logic")
(options-auto nl-rlv-assert-bounds 1 :reason "non-quantified logic")
(options-auto quant-dsplit none :reason "non-datatypes logic")
```

Above, more specific options were configured that are specific to `QF_LIA`, for example the option `arithRewriteEq` (which eagerly replaces arithmetic equalities by a conjunction of inequalities) is set to true.
Furthermore we do not require modifying the Minisat simplification mode in this case.

When given no command line options, cvc5 generally will perform close to optimal in most use cases.
However, some expert users may require manually configuring options that are tailored to their use of cvc5.
Notably, the set of options for solving quantified formulas may vary significantly based on the logic and the needs and priorities of the user. 
We plan to publish a followup post on recommendations how to find the best combination for your application.

## Conclusion

This post covers many of the diagnostic features of cvc5 that are intended to be useful both to users and developers of cvc5.
We have shown various interfaces that are available in the *.smt2 interface and available via our APIs.

Many other diagnostics are available internally, some of which we are in the process of making available to users as well.
If you have a request for more information to understand what cvc5 is doing, don't hesitate to contact us, or to post a question on our [discussions page](https://github.com/cvc5/cvc5/discussions).

#### [Andrew Reynolds](https://homepage.cs.uiowa.edu/~ajreynol/) is a research scientist at the University of Iowa in the Computational Logic Center ([CLC](https://clc.cs.uiowa.edu/site/index.shtml)) and one of the main developers of cvc5. His research is focused on several aspects of SMT solving, including quantified formulas, strings and regular expressions, syntax-guided synthesis and proofs.
