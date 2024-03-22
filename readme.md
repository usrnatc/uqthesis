::: titlepage
::: center

![logo](UQlogo.png)

:::

::: center
**Project Proposal\
Draft\
*by\
Nathan Corcoran\
School of Information Technology and Electrical Engineering,\
The University of Queensland.\
Submitted for the degree of\
Bachelor of Engineering\
in the field of Software Engineering\
March 2024.***
:::
:::

::: flushright
Nathan Corcoran\
n.corcoran@uqconnect.edu.au\
2024-03-22
:::

::: flushleft
Prof Michael Bruenig\
Head of School\
School of Information Technology and Electrical Engineering\
The University of Queensland\
St Lucia, Q 4072\
Dear Professor Bruenig,
:::

In accordance with the requirements of the degree of Bachelor of
Engineering in the division of Software Engineering, I present the
following thesis entitled "Investigating Arithmetic Re-Association
within the GraalVM Compiler". This work was performed under the
supervision of A/Prof. Mark Utting, Prof. Ian J. Hayes and, Mr. Brae
Webb.

I declare that the work submitted in this thesis is my own, except as
acknowledged in the text and footnotes, and has not been previously
submitted for a degree at The University of Queensland or any other
institution.

::: flushright
Yours sincerely,\
*--INSERT SIGNATURE--*\
Nathan Corcoran.
:::

# Acknowledgments

Acknowledge your supervisor, preferably with a few short and specific
statements about his/her contribution to the content and direction of
the project. If you collaborated with another student, acknowledge your
partner's contribution, including any parts of the thesis of which s/he
was the principal author or co-author; this information can be
duplicated in footnotes to the chapters or sections to which your
partner has contributed. Briefly describe any assistance that you
received from technical or administrative staff. Support of family and
friends may also be acknowledged, but avoid sentimentality---or hide it
in the dedication.

# Abstract

# Introduction {#intro}

Optimising compilers are a crucial tool for building efficient modern
programs. They allow developers to write highly abstracted code without
sacrificing program performance. High-level abstractions are relied upon
when programming complex models that many of modern software requires.
The GraalVM compiler is one such compiler that aims to achieve
high-levels of optimisation. Arithmetic expressions allow for a plethora
of optimisation opportunities and it is important that expressions are
represented in such a way to ease methods of optimisation. Arithmetic
re-association plays a pivotal role in this. The method is not used to
optimise the code by itself, but rather to create opportunities for
further optimisations. Consequently, to maximise the effectiveness of an
optimising compiler, we must ensure effectiveness of re-association
methods. Currently, the GraalVM compiler optimises expressions by way of
graph comparisons and manipulations. Although GraalVM currently employs
a method of re-association, its stated purpose is to aid only
loop-invariant and constant-folding optimisations [@graalsrc].
Expressions outside of loops are not re-associated, potentially losing
further optimisations within a program.\
The purpose of this report is to investigate the effectiveness of
current method of arithmetic re-association within the GraalVM compiler.
The report will attempt to determine short-comings of the compiler by
way of missed further expression optimisations. A alternative method of
re-association will be investigated to maximise the effectiveness of
optimisations within the GraalVM compiler. The report will have the
following structure: Section [4.1](#ara){reference-type="ref"
reference="ara"} will discuss arithmetic re-association following on
with Section [4.2](#opt){reference-type="ref" reference="opt"} covering
some of the further optimisations it aids,
 Section [4.3](#ir){reference-type="ref" reference="ir"} will cover how
a compiler can represent source code using graphs,
 Section [4.4](#graal){reference-type="ref" reference="graal"} will
discuss GraalVM and how re-association methods are applied to this
representation,  Section [5](#litrev){reference-type="ref"
reference="litrev"} will review some previous work on optimising
compilers, and finally, Section [6](#methods){reference-type="ref"
reference="methods"} will present a plan to investigate shortcomings of
current methods within the GraalVM compiler as well as a plan to rectify
them.

# Background {#bg}

## Arithmetic Re-Association {#ara}

Arithmetic re-association utilises the associative and commutative
properties of certain arithmetic operators to re-arrange
expressions [@redund]. This may allow simplifications that were not
easily identified in the original expression to be made evident.
Re-association methods use associative, distributive, and commutative
properties of some operators to re-order arithmetic expressions. Using
re-association, a compiler would transform the expression
`x = 23 + i + y + 43 + i` into the form `x = i + i + y + 23 + 43`. It is
now easier for the compiler to reason that the sub-expression `23 + 43`
can be optimised to a single constant. Arithmetic re-association only
alters the representation of a given expression, and not its resulting
value. Hence, it can only be applied to operators that are distributive.
For this reason, re-association methods are generally not applied to
floating point expressions [@floats].

## Optimisations {#opt}

### Algebraic Simplification {#as}

Algebraic simplification can allow for large performance optimisations
in programs. It is the process of simplifying expressions which can
produce smaller but equivalent expressions, or replace operators with
which are faster to compute. This process utilises both the mathematical
properties of associativity, commutativity, and distributivity, and
operator identities. Some simple examples of algebraic simplification
can be seen in Fig. [4.1](#simp){reference-type="ref" reference="simp"}.

<figure id="simp">
<pre data-frame="single" data-basicstyle="\ttfamily\small"
data-tabsize="1" data-columns="fullflexible"><code>-(-a)           --&gt;  a
    a - a           --&gt;  0
    a * 2           --&gt;  a &lt;&lt; 1
    2 * a + 3 * a   --&gt;  (2 + 3) * a</code></pre>
<figcaption>Examples of algebraic simplifications</figcaption>
</figure>

### Constant Folding {#cf}

Constant folding is the process of trivially merging sequences of
sub-expressions containing only constant values. Simply, the expression
$2 + 3 - 1$ would be transformed to be $4$.

### Loop Invariant Code Motion {#licm}

Loop invariant code motion, also referred to as *hoisting*, aims to
reduce the number of computations required within loop bodies. If a sub-
expression is reasoned to be constant, that is invariant, across all
loop iterations, this value can be stored elsewhere and its resulting
value can be referenced inside of the loop. A trivial example of this
can be seen in Fig. [4.2](#hoist){reference-type="ref"
reference="hoist"}. More complex scenarios arise when function calls are
hoisted, yeilding larger performance optimisations.

<figure id="hoist">
<pre data-frame="single" data-basicstyle="\ttfamily\small"
data-tabsize="1" data-columns="fullflexible"><code>
        for i = 1..10 {
            a = i + b * 3
        }</code></pre>
<pre data-frame="single" data-basicstyle="\ttfamily\small"
data-tabsize="1" data-columns="fullflexible"><code>b0 = b * 3
        for i = 1..10 {
            a = i + b0
        }</code></pre>
<figcaption>Example of before and after hoisting
respectively</figcaption>
</figure>

## Intermediate Representation {#ir}

Generally, compilers do not perform optimisations directly on high-level
code. Instead, an *intermediate representation* (IR) is used to
represent the program in such a way that optimisations are easily
implemented. Intermediate representations are split into three levels:
high, medium, and low. Each level generally represents the source code
in different ways, these differences are defined by the optimisations
expecting to be performed. Common forms of IR are graph-, tree-, and
stack-based structures. These are usually selected depending on how much
information is needed from the source code in order to apply an
optimisation. This report will focus on graph-based intermediate
representation.

## GraalVM {#graal}

The GraalVM compiler, released by Oracle Labs, is a polyglot optimising
compiler for languages that run on the Java Virtual Machine (JVM). It
focuses on aggresively optimising programs, with around 62 possible
methods [@graalenterprise] of doing so.\
The GraalVM compiler uses a graph-based structure as an intermediate
representation. Graphs are an efficient way to store program flow
contexts. Nodes represent basic code blocks or data values, and directed
edges represent dependencies between nodes. Lacking circular
dependencies [@gir], the graph is more accurately a directed acyclic
graph (DAG). DAGs minimise redundancy by sharing common sub-graphs. The
GraalVM IR also uses a single graph to model both control- and
data-flow, creating a *program-graph* [@understanding]. Traversing
control edges outlines the order in which a program must be run, whilst
traversing data edges outlines how data is passed and what instructions
are manipulating it throughout the program's run. This benefits the
implementation of optimisations as both contexts are easily accessed.
Though, this makes graphs with nested or complex control flows messy and
difficult to follow. Hence, it is often stated that GraalVM uses a *sea
of nodes* to represent program behaviour.\
Within GraalVM optimisations are referred to as *phases*. Execution
order of phases are defined as `PhaseSuite`s, an ordered list of
optimisations to be applied in sequence. `PhaseSuite`s can be scheduled
within other `PhaseSuite`s. The compiler uses three tiers to run phases;
low, mid, and high. Progressing from the high to the low tier, lower and
lower level optimisations are applied to the graph. After each tier has
completed, the graph is prepared for the next, called
*lowering* [@poly-graal]

# Literature Review {#litrev}

## Effective Partial Redundancy Elimination {#litrev1}

This report addresses methods to increase the effectiveness of partial
redundancy elimination within an optimising compiler. The report
proposes a method of *global reassociation* to re-arrange expressions
within a program.\
The report uses a system to sort sub-expressions by ranking the
operators within them. It uses these ranks as a hueristic to determine a
sub-expressions new location within an expression and to determine which
sub-expressions should be distributed.\
This report will investigate the methods proposed to determine any
benefit in a the GraalVM compiler, though some concerns are immediately
apparent. The report proposes the use of *expression trees* to store
sub-expressions and their associated ranks, and are forward-propagated.
This is generally a space-expensive operation. This report will
investigate the pay off from gained optimisations, if any, to the
excessive memory usage that may present itself when re-associating very
large and complex expressions. The report also explicitly states that
what it proposes is not an optimal solution, but rather one that
produces good results. This report will investigate extensions to the
method presented for optimal arithmetic re-association. [@effective-pre]

## Finding Missed Compiler Optimizations by Differential Testing {#litrev2}

This report investigates the usage of differential testing to detect
missed compiler optimisations. The report generates random programs with
well defined source code constraints and compares generated binaries
across 3 C compilers; GCC, Clang, and CompCert. The binaries are
compared with a Python tool called `optdiff`. The randomly generated
code was restricted by omitting any code that would (A) invoke undefined
behaviour from the language, and (B) rely on compiler-defined
implementations of certain features, order of evaluation of expressions
is presented as an example. The report correctly identifies some
difficulties in using differential analysis on generated binaries,
however, reducers are presented as a sound solution to these issues.\
The report scope is extremely broad in that it attempts to catch missed
opportunities from a large range of expected optimisations, some of
which are explicitly stated as begin architecture-specific. The report
also states that some initial results were found to be false-positive.\
This report will investigate the usage of differential analysis to find
missed optimisations within the GraalVM compiler. In an attempt to
maximise the validity of any findings, this report will narrow the scope
of missed optimisations to be searched for, and methods data used when
comparing. This report will investigate this testing method focusing on
finding missed optimisations relating to arithmetic expressions, and the
usage of generated IR graphs for comparison. A method to randomly
generate large complex expressions with corresponding simplified
equivalents will be investigated. In order to uphold correctness, a
verification method will also be investigated. [@missed-opt]

# Methodology {#methods}

In order to maintain a sound workflow and maximise findings, this
project will adhere to the following milestones:

::: {#milestones}
   **Milestone**    **Proposed Deadline**
  ---------------- -----------------------
      Research        Week 7 Semester 1
   Implementation    Week 12 Semester 1
    Optimisation      Week 7 Semester 2
       Report        Week 13 Semester 2
      Testing            Throughout

  : Proposed Milestones
:::

## Research

Research for the project will consist of the following:

-   Further investigate expression re-association techniques

-   Further investigate methods of testing compiler optimisations

-   Develop implementation plan for alternate expression re-association
    techniques

-   Develop implementation plan for testing alternate methods

-   Become familiar with the GraalVM compiler structure

## Implementation

Implementation for the project will consist of the following:

-   Develop expression re-association algorithm

-   Integrate expression re-association algorithm in to GraalVM compiler

-   Consistent meetings with supervisors to review implementation

## Optimisation

Optimisation for the project will be completed if deemed necessary and
will consist of the following:

-   Profile implementation to identify potential performance bottlenecks

-   Plan implementation performance fixes

-   Consistent meetings with supervisors to review code fixes

## Report

Report writing for the project will consist of the following:

-   Consistent upkeep of progress journal

-   Consistent logging of findings

-   Determining future improvements of findings

-   Determining further usages of re-association algorithm

-   Consistent meetings with supervisors to discuss potential edits

## Testing

Testing within the project will be continuous and will ultimately shape
the final implementation. Testing for the project will consist of the
following:

-   Develop testing of compiler optimisations that may be missed

-   Develop complex expressions to test compiler optimisation

-   Investigate method of automatic running of tests

-   Consistent meetings with supervisors and peers for recommendations
    of testing methods

## Risks

The following risks have been determined for the project along with
associated mitigation and prevention strategies:

::: {#project-risk-table}
  **Risk**                    **Likelihood**   **Consequence**   **Mitigation**                          **Prevention**
  --------------------------- ---------------- ----------------- --------------------------------------- --------------------------------------------------------------------------
  Loss of work                Unlikely         Severe            Manage multiple instances of progress   Store progress on UQ server and save and push regularly
  Milestone delays            Unlikely         Moderate          Create time management plan             Consistent checking of time management plan and meeting with supervisors
  Loss of project direction   Unlikely         Low               Create clear task plan                  Consistent meeting with supervisors to review work

  : Project Risks
:::

::: {#safety-risk-table}
  **Risk**                 **Likelihood**   **Consequence**   **Mitigation**          **Prevention**
  ------------------------ ---------------- ----------------- ----------------------- ---------------------------------------------
  Unsafe use of Computer   Unlikely         Moderate          Comfortable workspace   Ensure regular breaks during computer usage

  : Safety Risks
:::

# Results and discussion ...

# Conclusions

::: thebibliography
99

Oracle. 2024. *GraalVM: An advanced JDK with ahead-of-time Native Image
compilation*. https://github.com/oracle/graal Brae J. Webb, Ian J.
Hayes, and Mark Utting. 2023. *Verifying Term Graph Optimizations using
Isabelle/HOL*. In Proceedings of the 12th ACM SIGPLAN International
Conference on Certified Programs and Proofs (CPP '23), January 16--17,
2023, Boston, MA, USA. ACM, New York, NY, USA, 14 pages.
https://doi.org/10.1145/3573105.3575673 Gilles Duboscq, Lukas Stadler,
Thomas Wuerthinger, Doug Simon, Christian Wimmer, Hanspeter
Moessenboeck. 2013. *Graal IR: An Extensible Declarative Intermediate
Representation*. https://api.semanticscholar.org/CorpusID:52231504
OpenJDK Community. 2012. *Graal Project*.
https://openjdk.org/projects/graal Cliff Click, Michael Paleczny. 1995.
*A Simple Graph-Based Intermediate Representation*.
https://www.oracle.com/technetwork/java/javase/tech/c2-ir95-150110.pdf
Steven S. Muchnick. 1997. *Advanced Compiler Design and Implementation*.
Chapter 12. ISBN 1-55860-320-4 Oracle. 2024. *Oracle GraalVM Enterprise
Edition 21*.
https://docs.oracle.com/en/graalvm/enterprise/21/docs/reference-manual/java/compiler/#graal-compiler
Preston Briggs and Keith D. Cooper. 1994. *Effective partial redundancy
elimination*. SIGPLAN Not. 29, 6 (June 1994), 159--170.
https://doi.org/10.1145/773473.178257 M. Sipek, B. Mihaljevic, A.
Radovan. 2021. *Exploring Aspects of Polyglot High-Performance Virtual
Machine GraalVM*. https://doi.org/10.23919/MIPRO.2019.8756917 Gergoe
Barany. 2018. *Finding Missed Compiler Optimizations by Differential
Testing*. Compiler Construction (CC'18). ACM, New York, NY, USA, 11
pages. https://doi.org/10.1145/3178372.3179521 Chris Seaton. 2017.
*Understanding How Graal Works - a Java JIT Compiler Written in Java*.
https://chrisseaton.com/truffleruby/jokerconf17/ Martyn J. Corden, David
Kreitzer. 2018. *Consistency of Floating-Point Results using the Intel
Compiler or Why doesn't my application always give the same answer?*.
Software Services Group, Intel Corporation.
https://www.intel.com/content/dam/develop/external/us/en/documents/pdf/fp-consistency-121918.pdf
Keith Cooper, Jason Eckhardt, Ken Kennedy. 2008. *Redundancy Elimination
Revisited*. PACT'08, October 25--29, 2008, Toronto, Ontario, Canada.
https://cscads.rice.edu/p12-cooper.pdf Oracle. 2015-2022. *GraalVM
Reassociation Phase Source*.
https://github.com/oracle/graal/blob/master/compiler/src/\
jdk.graal.compiler/src/jdk/graal/compiler/phases/common/ReassociationPhase.java
David Monniaux, Cyril Six. 2022. *Formally Verified Loop-Invariant Code
Motion and Assorted Optimizations*. ACM Transactions on Embedded
Computing Systems (TECS). ff10.114/3529507ff.hal03628646
:::
