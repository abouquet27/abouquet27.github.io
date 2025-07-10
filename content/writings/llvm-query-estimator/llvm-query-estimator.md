---
title: "Implementing a Query Count Estimation"
description: "Bachelor Project @ Dependable Systems Lab, EPFL"
date: 2023-06-09
bibliography: references.bib
tags: ["Symbolic Execution", "LLVM"]
link-citations: true
csl: ieee.csl
block-headings: true
toc: true
---

*This paper was written under the supervision of Ph. D. candidates Solal Pirelli and Can Cebeci from DSLAB at EPFL and is available in PDF format on [this repository](https://github.com/abouquet27/BachelorProject/blob/main/project/Bachelor_report.pdf)*.

<html xmlns="http://www.w3.org/1999/xhtml" lang="" xml:lang="">

<meta charset="utf-8" />
<meta name="generator" content="pandoc" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes" />
<meta name="author" content="Adrien Alain Bouquet" />

<style>
    /* Warning */
    /* CSS for citations */
    div.csl-bib-body {}

    div.csl-entry {
        clear: both;
        margin-bottom: 0em;
    }

    .hanging-indent div.csl-entry {
        margin-left: 2em;
        text-indent: -2em;
    }

    div.csl-left-margin {
        min-width: 2em;
        float: left;
    }

    div.csl-right-inline {
        margin-left: 2em;
        padding-left: 1em;
    }

    div.csl-indent {
        margin-left: 2em;
    }
</style>
<script defer="" src="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.js"></script>
<script>document.addEventListener("DOMContentLoaded", function () {
        var mathElements = document.getElementsByClassName("math");
        var macros = [];
        for (var i = 0; i < mathElements.length; i++) {
            var texText = mathElements[i].firstChild;
            if (mathElements[i].tagName == "SPAN") {
                katex.render(texText.data, mathElements[i], {
                    displayMode: mathElements[i].classList.contains('display'),
                    throwOnError: false,
                    macros: macros,
                    fleqn: false
                });
            }
        }
    });
</script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.css" />   
<body>

## Introduction
When you’re trying to test code, symbolic execution is an effective
way of doing this by automatically generating tests and finding bugs.
However, its cost in terms of performance can be heavy due to path
explosion and this state explosion. To counter this problem, one
approach is to merge states obtained on different paths in order to
reduce the total exploration cost. However, merging two states can
potentially increase costs and therefore decrease performance rather
than the opposite. In order to determine which states to merge or not,
researchers at the Dependable Systems Lab (DSLAB) at the Ecole
Polytechnique Fédéral de Lausanne (EPFL) have published “Efficient State
Merging in Symbolic Execution” in 2012 <span class="citation" data-cites="kuznetsov_state_merging"><a
        href="#ref-kuznetsov_state_merging" role="doc-biblioref">[1]</a></span>,
in which they present two methods for reducing costs of state merging,
especially deployed on KLEE, a symbolic execution engine <span class="citation" data-cites="klee"><a
        href="#ref-klee" role="doc-biblioref">[2]</a></span>. The first one is a query count
estimation which computes the impact of each symbolic variable on solver
queries and indicates which one provides benefits to merge. The second
one is dynamic state merging which “interacts favorably with search
strategies in automated test case generation and bug finding tools”. In
this report, we will be working on an implementation of the query count
estimation based on its description. The goal of the report is first to
explain the implementation of the LLVM pass <span class="citation" data-cites="llvm"><a href="#ref-llvm"
        role="doc-biblioref">[3]</a></span> chosen to be as close as possible to
that described in the paper, and then to test its effectiveness and
robustness on different examples and the <code>GNU COREUTILS</code>.
<span class="citation" data-cites="coreutils"><a href="#ref-coreutils" role="doc-biblioref">[4]</a></span>

## Definition of the problem 
<p>
The central point of this report is the implementation of the
    heuristic query count estimation (QCE). It is essential to explain the
    purpose of this heuristic in order to understand the implementation
    described below: In the context of symbolic execution, one of the
    objectives is to reduce the computational cost that the introduction of
    symbolic variables can generate to increase its efficiency. To do this,
    the paper proposes this QCE heuristic whose aim is to determine whether
    a variable <span class="math inline">v</span> at a location <span class="math inline">\ell</span> in the program
    is a sensitive variable
    (or a “hot variable” as denoted in the paper). As described in section
    3.2, a variable <span class="math inline">v</span> is sensitive (or hot)
    at location <span class="math inline">\ell</span> if the number of
    additional queries <span class="math inline">Q_{add}(\ell,v)</span> is
    greater than a fixed ratio <span class="math inline">\alpha</span> of
    the total number of queries <span class="math inline">Q_t(\ell)</span>
    (see §3.2). Consequently, the paper presents a formula to count
    recursively the number of queries that are selected by a function <span class="math inline">c</span>:</p>
<p><span class="math display">
        q(\ell&#39;,c) =
        \begin{cases}
        \beta q(\mathrm{succ}(\ell&#39;),c) + \beta q(\ell &#39;&#39;,c) +
        c(\ell&#39;, v) &amp; \quad \text{instr($\ell$&#39;) = if (e) go to
        $\ell$&#39;&#39; (1.a)}\\
        0 &amp; \quad \text{instr($\ell$&#39;) = halt (1.b)}\\
        q(\mathrm{succ}(\ell&#39;,c)) &amp; \quad \text{otherwise (1.c)}
        \end{cases}
        \quad(1)
    </span></p>
<p>Where <span class="math inline">e</span> is an expression, <span class="math inline">c</span> is a function given
    in parameter that
    evaluates the expression <span class="math inline">e</span> and <span class="math inline">\beta</span>
    represents the probability that a
    branch is feasible. The paper fixes it to <span class="math inline">0.6</span> for example. Therefore, it
    employs this
    formula to define the two numbers of queries defined above by: <span class="math display">
        Q_{add}(\ell,v) =
        q(l,\lambda(\ell&#39;,e).ite((\ell,v)\triangleleft(\ell&#39;,e), 1,0))
        \qquad(2)
    </span> <span class="math display">
        Q_{t}(\ell,v) = q(l,\lambda(\ell&#39;,e).1) \qquad(3)
    </span></p>
<p>Presented in this way, the formulas do not allow us to understand
    clearly how to implement the LLVM pass and it is necessary to explain
    them. First of all, a program location <span class="math inline">\ell</span> can be understood as an
    instruction,
    <span class="math inline">\ell&#39;</span> can be seen as the next
    instruction (the next line in terms of code) and <span class="math inline">\mathrm{succ}(\ell)</span> as the
    successor of <span class="math inline">\ell</span>. Then, <span class="math inline">(1)</span> separates 3 cases
    which return a value.
    The first one <span class="math inline">(1.a)</span> is a branch
    condition with an expression (an if-else for instance), the second one
    <span class="math inline">(1.b)</span> is a program termination and the
    last one <span class="math inline">(1.c)</span> is all the other type of
    instructions. Hence, <span class="math inline">(1.a)</span> evaluates
    recursively the branch if <span class="math inline">e</span> is true and
    the branch if <span class="math inline">e</span> is false. It also
    evaluates the condition using the function <span class="math inline">c</span>. The two last cases are pretty
    straightforward: respectively return <span class="math inline">0</span>
    if the program ends and return the value of the next instruction. In
    summary, this formula can be represented as if we were walking through a
    tree. When reaching a condition, evaluate the condition using the given
    function and recursively evaluates the 2 branch. When reaching an other
    instruction, skip it, When reaching an end, it’s a leaf.
</p>
<p>As a result, <span class="math inline">Q_{add}</span> becomes the
    evaluation of the formula when the function <span class="math inline">c</span> checks if the expression <span
        class="math inline">e</span> depends on the variable <span class="math inline">v</span> and hence will
    generate a solver query and
    <span class="math inline">Q_{t}</span> becomes the evaluation of the
    formula when <span class="math inline">c</span> returns always 1 as it
    simply counts the number of branch condition.
</p>
<p>However, this explanation is not sufficient in itself, as certain
    instructions are problematic as explained in the paper. Firstly, loops
    and recursion prevent the formula from stopping because of the recursive
    nature of its definition. In the paper, a loop bound <span class="math inline">\kappa</span> is set to limit the
    maximum loop
    iterations and recursive calls. In addition, functions calls are
    problematic because they must be evaluated too and their number of
    queries must be added to the total one. Therefore, the next part below
    will explain the implementation of an LLVM pass with respect to the
    formulas and constraints stated before.</p>

## Implementation
<p>Before describing the code in concrete terms, it is important to
    define the different variables used in the theory in terms of code, and
    more specifically in terms of LLVM assembly or C++. The equivalent of an
    instruction <span class="math inline">\ell</span>, as explained above,
    is the <code>Instruction</code> class, a variable <span class="math inline">v</span> as the paper intended will
    be have for
    equivalent the <code>Value</code> class, the function <span class="math inline">q</span> has for implementation
    the
    <code>compute_query</code> function described below and the function
    <span class="math inline">c</span> has for implementation either the
    <code>returnTrueFunction</code> or the <code>matchValue</code> also
    decribed below.
</p>

### Parameters & Data structure created
<p>The value for <span class="math inline">\beta</span> which represents
    the probability of the feasibility of a branch is fixed to <span class="math inline">0.6</span> and the maximum
    number of iterations
    <span class="math inline">\kappa</span> is separated into two variables
    <code>nloops</code> and <code>dcalls</code> both fixed to
    <code>1</code>. Furthermore, the <span class="math inline">\alpha</span>
    parameter which is used to compute if a variable is “hot” is fixed to
    <span class="math inline">0.5</span>. Theses values were chosen from the
    example at the end of page 5 to facilitate to facilitate comparison with
    the pass defined in the paper and the tests.
</p>
<p>Besides, two data structures were created: <code>BranchCount</code>
    and <code>CallCount</code>. The former associates a branch instruction
    (i.e. <code>BranchInst</code> in LLVM assembly) to the number of time
    the instruction has been encountered, the latter is analog to the former
    one but it counts the number of times a function
    (i.e. <code>Function</code>) is encountered. Basically, they are used to
    keep track of loop iterations and recursions.</p>

### Compute Query function
<p>As stated before, the <code>compute_query</code> function is the
    equivalent of <span class="math inline">q(\ell,c)</span> which computes
    the number of queries. It takes as argument a pointer to an LLVM
    instruction <code>Inst</code> (i.e <span class="math inline">\ell</span>), a pointer to an LLVM value
    <code>researched</code> (i.e. <span class="math inline">v</span>), the
    list of all the <code>BranchCount</code> of the backwards branch, in
    other words all the branches that jump to a label prior to the current
    one, the list of all the <code>CallCount</code> of all the
    <code>CallInst</code> that depends on the value <code>researched</code>
    and finally a function (i.e <span class="math inline">c</span>) that
    evaluates if the condition of the branch instruction depends on the
    value <code>researched</code>. The function applies what has been
    previously defined and handles the constraint such as loops iterations
    or recursion by checking if the maximum count has been reached.
</p>

### Match Value Function
<p>The implementation of the function <span class="math inline">c</span>
    that checks if an expression depends on a particular variable is
    <code>matchValue</code> which takes as argument two LLVM value: the
    first one correspond to the value we are testing and the second one is
    the Value that we are searching (i.e. the variable <span class="math inline">v</span> in the paper). Although
    its goal is pretty
    straightforward, its implementation is more complex. In fact, we have to
    backtrack in the code by going from the operands of an instruction,
    checking if they are the value we search and if not checking their
    operands until the variable searched is found or there is nore more
    instruction and hence the initial condition does not depends on the
    variable. However, this logic might causes <code>matchValue</code> to
    loop if they are loops or recursion. For instance:
</p>
<div class="sourceCode" id="cb1">
    <pre class="sourceCode c"><code class="sourceCode c"><span id="cb1-1"><a href="#cb1-1" aria-hidden="true" tabindex="-1"></a><span class="dt">void</span> loopFunc<span class="op">(</span><span class="dt">int</span> x<span class="op">){</span></span>
<span id="cb1-2"><a href="#cb1-2" aria-hidden="true" tabindex="-1"></a>    <span class="dt">int</span> index <span class="op">=</span> x<span class="op">;</span></span>
<span id="cb1-3"><a href="#cb1-3" aria-hidden="true" tabindex="-1"></a></span>
<span id="cb1-4"><a href="#cb1-4" aria-hidden="true" tabindex="-1"></a>    <span class="cf">while</span> <span class="op">(</span>index <span class="op">&lt;</span> <span class="dv">10</span><span class="op">){</span></span>
<span id="cb1-5"><a href="#cb1-5" aria-hidden="true" tabindex="-1"></a>        <span class="co">//...//</span></span>
<span id="cb1-6"><a href="#cb1-6" aria-hidden="true" tabindex="-1"></a>        index<span class="op">++;</span></span>
<span id="cb1-7"><a href="#cb1-7" aria-hidden="true" tabindex="-1"></a>    <span class="op">}</span></span>
<span id="cb1-8"><a href="#cb1-8" aria-hidden="true" tabindex="-1"></a><span class="op">}</span></span></code></pre>
</div>
<p>When reaching the condition of the while loop, the index has been
    redefined and thus there are two paths leading to it. In LLVM, there
    exist <code>PHINode</code> which are instructions that regroup the
    differents paths of the variable, in other words the differents values
    and the label where they are defined.</p>

### The pass
<p>The pass computes twice the query count of a function for each its
    argument: once using the <code>matchValue</code> function described
    above and once using the <code>returnTrueFunction</code> to count the
    total number of queries. It then outputs the value using both function
    and if the variable is a hot variable.</p>

## Results

### Basic cases

<p>In order to evaluate the correctness on basic code, a set of 21
    cases<a href="#fn1" class="footnote-ref" id="fnref1" role="doc-noteref"><sup>1</sup></a> as been established to
    tests
    implementation and constraint handling, and we are going to demonstrate
    3 of them. As a reminder, the probability <span class="math inline">\beta</span> that a branch is feasible is
    fixed to
    <span class="math inline">0.6</span> and the parameter <span class="math inline">\kappa</span> which correspond
    to the maximum loop
    iterations and recursion depth is fixed to 1.
</p>
<h4 id="basic-condition">Basic condition</h4>
<div class="sourceCode" id="cb2">
    <pre class="sourceCode c"><code class="sourceCode c"><span id="cb2-1"><a href="#cb2-1" aria-hidden="true" tabindex="-1"></a><span class="dt">void</span> basicCondition<span class="op">(</span><span class="dt">int</span> a<span class="op">){</span></span>
<span id="cb2-2"><a href="#cb2-2" aria-hidden="true" tabindex="-1"></a>  <span class="dt">int</span> x <span class="op">=</span> <span class="dv">0</span><span class="op">;</span></span>
<span id="cb2-3"><a href="#cb2-3" aria-hidden="true" tabindex="-1"></a>  <span class="dt">int</span> b <span class="op">=</span> <span class="dv">19</span><span class="op">;</span></span>
<span id="cb2-4"><a href="#cb2-4" aria-hidden="true" tabindex="-1"></a>  <span class="cf">if</span> <span class="op">(</span>a <span class="op">&gt;=</span> <span class="dv">5</span><span class="op">)</span> <span class="op">{</span></span>
<span id="cb2-5"><a href="#cb2-5" aria-hidden="true" tabindex="-1"></a>    x <span class="op">=</span> <span class="dv">5</span><span class="op">;</span></span>
<span id="cb2-6"><a href="#cb2-6" aria-hidden="true" tabindex="-1"></a>  <span class="op">}</span> <span class="cf">else</span> <span class="op">{</span></span>
<span id="cb2-7"><a href="#cb2-7" aria-hidden="true" tabindex="-1"></a>    x <span class="op">=</span> <span class="dv">10</span><span class="op">;</span></span>
<span id="cb2-8"><a href="#cb2-8" aria-hidden="true" tabindex="-1"></a>  <span class="op">}</span></span>
<span id="cb2-9"><a href="#cb2-9" aria-hidden="true" tabindex="-1"></a>  <span class="dt">int</span> y <span class="op">=</span> x <span class="op">+</span> a<span class="op">;</span></span>
<span id="cb2-10"><a href="#cb2-10" aria-hidden="true" tabindex="-1"></a><span class="op">}</span></span></code></pre>
</div>
<p>This case is pretty straightforward when applying the pass as the
    variable tested <span class="math inline">a</span> is in the condition
    of the if and hence <code>matchValue</code> will return 1. Furthermore,
    the evaluation of the two branch will return <span class="math inline">0</span> each as there are not any
    conditions left.
    Thus, the query count will be equals to <span class="math inline">1.0</span> which is the value we theoretically
    expect. Here is the full expectation computation (line 1 correspond to
    the function header):</p>
<p><span class="math display"> Q_{add}(1,a) = q(1,c) = q(4,c) </span>
    <span class="math display"> = \beta q(7,c) + \beta q(5,c) + c(a &gt;=
        5,a)</span> <span class="math display"> = \beta q(10,c) + \beta q(10,c)
        + 1 = 0 + 0 + 1 = 1 </span>
</p>
<h4 id="recursion">Recursion</h4>
<div class="sourceCode" id="cb3">
    <pre class="sourceCode c"><code class="sourceCode c"><span id="cb3-1"><a href="#cb3-1" aria-hidden="true" tabindex="-1"></a><span class="dt">void</span> recursionFunction<span class="op">(</span><span class="dt">int</span> x<span class="op">){</span></span>
<span id="cb3-2"><a href="#cb3-2" aria-hidden="true" tabindex="-1"></a>    <span class="cf">if</span> <span class="op">(</span>x <span class="op">&gt;</span> <span class="dv">0</span><span class="op">){</span></span>
<span id="cb3-3"><a href="#cb3-3" aria-hidden="true" tabindex="-1"></a>        printf<span class="op">(</span><span class="st">&quot;recursion needed&quot;</span><span class="op">);</span></span>
<span id="cb3-4"><a href="#cb3-4" aria-hidden="true" tabindex="-1"></a>        recursionFunction1<span class="op">(</span>x<span class="op">-</span><span class="dv">1</span><span class="op">);</span></span>
<span id="cb3-5"><a href="#cb3-5" aria-hidden="true" tabindex="-1"></a>    <span class="op">}</span> </span>
<span id="cb3-6"><a href="#cb3-6" aria-hidden="true" tabindex="-1"></a><span class="op">}</span></span></code></pre>
</div>
<p>The interesting aspect of this case is recursion. When evaluated the
    first time, <code>matchValue</code> will return <span class="math inline">1</span> on the first condition, the
    <code>printf</code> call will be skipped as it does the x variable and
    it will evaluate a second time the condition:
</p>
<p><span class="math display"> Q_{add}(1,x) = q(1,c) = q(2,c) = \beta
        q(5,c) + \beta q(3,c) + c(x&gt;0,x)</span> <span class="math display"> =
        \beta q(6,c) + \beta q(1,c) + 1 = \beta (\beta q(5,c) + \beta q(3,c) +
        c(x&gt;0,x)) +1 </span> <span class="math display"> = \beta (0 + 0 + 1)
        +1 = 0.6 + 1 = 1.6</span></p>
<p>More generally, the different base cases have all been computed
    manually before being submitted to the pass and comparing the results.
    About twenty base cases have been established, computed and tested to
    make sure the pass correctly work on these cases.</p>
<h4 id="double-while-loop">Double while loop</h4>
<div class="sourceCode" id="cb4">
    <pre class="sourceCode c"><code class="sourceCode c"><span id="cb4-1"><a href="#cb4-1" aria-hidden="true" tabindex="-1"></a><span class="dt">void</span> doublewhileloopfunction<span class="op">(</span><span class="dt">int</span><span class="op">*</span> tab<span class="op">,</span> <span class="dt">int</span> size<span class="op">)</span> <span class="op">{</span></span>
<span id="cb4-2"><a href="#cb4-2" aria-hidden="true" tabindex="-1"></a>    <span class="dt">int</span> i <span class="op">=</span> <span class="dv">0</span><span class="op">;</span></span>
<span id="cb4-3"><a href="#cb4-3" aria-hidden="true" tabindex="-1"></a>    <span class="dt">int</span> j <span class="op">=</span> <span class="dv">0</span><span class="op">;</span></span>
<span id="cb4-4"><a href="#cb4-4" aria-hidden="true" tabindex="-1"></a>    <span class="cf">while</span> <span class="op">(</span>i <span class="op">&lt;</span> size<span class="op">)</span> <span class="op">{</span></span>
<span id="cb4-5"><a href="#cb4-5" aria-hidden="true" tabindex="-1"></a>        <span class="cf">while</span> <span class="op">(</span>j <span class="op">&lt;</span> size<span class="op">)</span> <span class="op">{</span></span>
<span id="cb4-6"><a href="#cb4-6" aria-hidden="true" tabindex="-1"></a>            tab<span class="op">[</span>i<span class="op">]</span> <span class="op">=</span> tab<span class="op">[</span>i<span class="op">]</span> <span class="op">+</span> tab<span class="op">[</span>j<span class="op">];</span></span>
<span id="cb4-7"><a href="#cb4-7" aria-hidden="true" tabindex="-1"></a>            j<span class="op">++;</span></span>
<span id="cb4-8"><a href="#cb4-8" aria-hidden="true" tabindex="-1"></a>        <span class="op">}</span></span>
<span id="cb4-9"><a href="#cb4-9" aria-hidden="true" tabindex="-1"></a>        i<span class="op">++;</span></span>
<span id="cb4-10"><a href="#cb4-10" aria-hidden="true" tabindex="-1"></a>    <span class="op">}</span></span>
<span id="cb4-11"><a href="#cb4-11" aria-hidden="true" tabindex="-1"></a><span class="op">}</span></span></code></pre>
</div>
<p>The case is rather interesting because the functions has 2 arguments
    and there is a double while loop. The result of evaluating the function
    <code>compute_query</code> on <code>tab</code> will be equals to <span class="math inline">0</span> as the
    argument is not part of any
    condition in the program. However, the argument <code>size</code> will
    have <span class="math inline">1.6</span> has result because it is
    inside two conditions.
</p>

### Example from the paper
<p>After creating cases to test the different issues the pass has to
    handle, the next step was testing it on the example from the original
    paper (<span class="citation" data-cites="kuznetsov_state_merging"><a href="#ref-kuznetsov_state_merging"
            role="doc-biblioref">[1]</a></span>
    Figure 1 page 5) . The paper uses this simplified version of the
    <code>echo</code> program. However, the example has been truncated to
    make it work using the pass, mainly by adding the variable ‘arg’ in the
    arguments of the function. Consequently, the pass computes <span class="math inline">1.6</span> which is the
    same value as the one
    computed by the paper and hence slightly strengthens the correctness of
    the pass
</p>

### Applying the pass on COREUTILS

<p>The next step in evaluating the pass is to test its robustness,
    i.e. its ability not to crash when applied to programmes that are much
    more complex than the examples shown above. To this end, the pass was
    applied to 30<code>COREUTILS</code><span class="citation" data-cites="coreutils"><a href="#ref-coreutils"
            role="doc-biblioref">[4]</a></span> of different purposes and sizes.
    They were chosen for 2 reasons: firstly, because they are programmes
    rich in complexity and variety, and secondly, because the paper also
    uses them to carry out its evaluations.</p>
<p>Out of 30 coreutils tested, 21 worked (i.e. the pass worked on all
    the functions of the coreutils tested and did not crash), 7 crashed
    (i.e. the pass did not finish and an error was thrown) and 2 did not
    terminate. The results is rather satisfying as it is 70% of the
    <code>COREUTILS</code> tested that works. Regarding those who does not
    terminate, the main explanation is the increasing complexity of the
    code. In fact, the more complex code becomes (by using function calls,
    recursion, etc), the more time it will need as the number of exploration
    path can increase exponentially. About those who crashed, the error
    thrown is <code>Illegal Instruction: 4</code> which is an issue with
    compile flags and might not happen on other OS.
</p>

### Limitations

<p>Although the pass works on a number of cases including some complex
    ones, it still has flaws, for instance hardly pointers to a function
    that is not constant or switch case that are not taken in account even
    though they are condition. As a side note, the former is not treated by
    KLEE because it is difficult to analyze a function and the pass could do
    the same. Furthermore, the correctness of pass has not been proved and
    hence its application might generate wrong results.</p>
<p>On the other hand, the pass could still be improved on its
    efficiency. Indeed, the pass enters and analyzes every function that are
    being called instead of computing them once as the paper planned to do.
    Thus, the cost of the computation is really heavy and not very useful as
    function that have already being called will be analyzed again.
    Moreover, function from external library such as <code>printf</code>
    initally caused the program to crash and hence have been skipped to
    facilitate the computation. However, this assumption of skipping
    function of external libraries can be discussed: on the one hand, it
    alters the final result but on the other hand the impact on it will
    become less significant as the size of the code and therefore the depth
    of the analysis increases. Losing some millionths may be a good way of
    improving efficiency.</p>
<p>Finally, some existing LLVM pass might exist to make the code easier
    to analyse. For instance, one major issue where the load and store
    because it was costly when encountering a load to trackback until
    finding the corresponding store. Hopefully, we did find the pass
    <code>mem2reg</code> which allows us to get rid of the majority of the
    stores and loads and thus to lower the cost.
</p>

## Conclusion
<p>In this report, we have presented the implementation of the query
    count estimation (QCE) heuristic from the paper “Efficient State Merging
    in Symbolic Execution” in the form of an LLVM pass. We explained the
    purpose of the heuristic and its formulas, and then provided a detailed
    description of the implementation. We also discussed the results of
    testing the pass on basic cases, the example from the paper, and a set
    of coreutils programs.</p>
<p>Overall, the implementation of the pass closely follows the
    description in the paper, and it successfully computes the query count
    for the given examples. The pass demonstrates its effectiveness in
    identifying hot variables and providing insights into the potential
    performance impact of state merging in symbolic execution. However, it
    is important to note that the pass has limitations and may not work on
    all programs. It may crash or fail to terminate on certain complex
    programs. Nevertheless, the pass shows promise and can serve as a
    starting point for future research.</p>
<p>In conclusion, this project contributes to a better understanding of
    the query count estimation heuristic and provides a practical
    implementation that can be used to analyze and optimize symbolic
    execution techniques.</p>

## Personnal discoveries
<p>This project has been a great learning experience on a large range of
    aspects. To start, it was my first big personal project in computer
    science and at EPFL. It required a lot of autonomy and a capacity of
    managing the hours for a result that is closer to reality than exercices
    seen during courses.</p>
<p>Speaking of courses, this project and working with LLVM was a good
    application of concepts seen during courses as Compilers or a good
    introduction to new concept such as Symbolic Executions. Another
    interesting point was the notion of research. Working on a project based
    on a paper from the laboratory in question is very motivating and
    stimulating because it requires you to do research and understand more
    than just what you need to implement.</p>
<p>Finally the main challenge was working with new languages with their
    particularities, more specific libraries with tougher documentation
    requiring more effort to understand how to use them or tools that
    requires a lot attempts to install and make them work. It took me a
    while to understand how does LLVM work and to use in my code to achieve
    the pass. The project required a lot of adaptation, but it developed
    cross-disciplinary skills that I’m sure will come in very useful later
    on.</p>

## References
<div id="refs" class="references csl-bib-body" data-entry-spacing="0" role="list">
    <div id="ref-kuznetsov_state_merging" class="csl-entry" role="listitem">
        <div class="csl-left-margin">[1] </div>
        <div class="csl-right-inline">V.
            Kuznetsov, J. Kinder, S. Bucur, and G. Candea, <span>“Efficient state
                merging in symbolic execution.”</span> </div>
    </div>
    <div id="ref-klee" class="csl-entry" role="listitem">
        <div class="csl-left-margin">[2] </div>
        <div class="csl-right-inline">KLEE Team, <span>“KLEE: Symbolic virtual
                machine.”</span> <a href="http://klee.github.io/" class="uri">http://klee.github.io/</a>.</div>
    </div>
    <div id="ref-llvm" class="csl-entry" role="listitem">
        <div class="csl-left-margin">[3] </div>
        <div class="csl-right-inline">LLVM Project, <span>“The LLVM compiler
                infrastructure project.”</span> <a href="https://llvm.org/" class="uri">https://llvm.org/</a>.</div>
    </div>
    <div id="ref-coreutils" class="csl-entry" role="listitem">
        <div class="csl-left-margin">[4] </div>
        <div class="csl-right-inline">Coreutils contributors, <span>“Coreutils/src at
                master · coreutils/coreutils.”</span> <a href="https://github.com/coreutils/coreutils"
                class="uri">https://github.com/coreutils/coreutils</a>.</div>
    </div>
</div>
<section id="footnotes" class="footnotes footnotes-end-of-document" role="doc-endnotes">
    <hr />
    <ol>
        <li id="fn1">
            <p><a href="https://github.com/abouquet27/BachelorProject/tree/main/project/cases"
                    class="uri">https://github.com/abouquet27/BachelorProject/tree/main/project/cases</a><a
                    href="#fnref1" class="footnote-back" role="doc-backlink">↩︎</a></p>
        </li>
    </ol>
</section>

</body>