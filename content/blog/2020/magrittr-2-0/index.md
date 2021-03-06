---
output: hugodown::hugo_document

slug: magrittr-2-0
title: magrittr 2.0 is coming soon
date: 2020-08-26
author: Lionel Henry
description: >
    A new version of the magrittr package brings laziness, better performance, and leaner backtraces for debugging errors.

photo:
  url: https://unsplash.com/photos/X-NAMq6uP3Q
  author: Mike Benna

categories: [package]
tags: []
rmd_hash: 7e0f4eae0236841e

---

<div class="highlight">

</div>

It is with unclouded composure that we announce the upcoming release of [magrittr](https://magrittr.tidyverse.org/) 2.0. magrittr is the package home to the [`%>%`](https://magrittr.tidyverse.org/reference/pipe.html) pipe operator written by Stefan Milton Bache and used throughout the tidyverse.

This last and likely final version of magrittr resolves the longstanding issues of overhead and backtrace footprint. It also makes the magrittr pipe more compatible with a native pipe that will probably be included in the next version of R.

This version of magrittr has been completely rewritten in C to give better backtraces and much improved performance. It also uses a different approach in order to support laziness. This enables new uses of the pipe, and ensures magrittr is as similar as possible to the future base pipe. Our analysis and testing suggests that the new version should be a drop-in replacement, but we'd really like you to try it out and give us some feedback before we submit to CRAN. You can install the development version from GitHub with:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='c'># install.packages("remotes")</span>
<span class='nf'>remotes</span><span class='nf'>::</span><span class='nf'><a href='https://remotes.r-lib.org/reference/install_github.html'>install_github</a></span><span class='o'>(</span><span class='s'>"tidyverse/magrittr"</span><span class='o'>)</span>
</code></pre>

</div>

If you discover any issues, please let us know by posting issues on the [Github repository](https://github.com/tidyverse/magrittr) of magrittr.

This blog post covers the three main changes in this new version of the magrittr pipe.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'><a href='http://magrittr.tidyverse.org'>magrittr</a></span><span class='o'>)</span>
</code></pre>

</div>

**Update 2020-09-22:** Some minor issues have arised from the first round of beta testing (see note on visibility and laziness below). We postponed the release to late October to give more time for potential further issues to surface before publication of magrittr 2.0 to CRAN.

Backtraces
----------

The R implementation of the magrittr pipe was rather costly in terms of backtrace clutter. This made it difficult to debug errors with functions using the pipe:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>foo</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='o'>)</span> <span class='nf'><a href='https://rdrr.io/r/grDevices/plotmath.html'>bar</a></span><span class='o'>(</span><span class='o'>)</span>
<span class='nv'>bar</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='o'>)</span> <span class='m'>1</span> <span class='o'>%&gt;%</span> <span class='nf'><a href='https://rdrr.io/r/base/identity.html'>identity</a></span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>baz</span><span class='o'>(</span><span class='o'>)</span>
<span class='nv'>baz</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nf'>rlang</span><span class='nf'>::</span><span class='nf'><a href='https://rlang.r-lib.org/reference/abort.html'>abort</a></span><span class='o'>(</span><span class='s'>"oh no"</span><span class='o'>)</span>

<span class='nf'>foo</span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; Error: oh no</span>

<span class='nf'>rlang</span><span class='nf'>::</span><span class='nf'><a href='https://rlang.r-lib.org/reference/last_error.html'>last_trace</a></span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; &lt;error/rlang_error&gt;</span>
<span class='c'>#&gt; oh no</span>
<span class='c'>#&gt; Backtrace:</span>
<span class='c'>#&gt;      █</span>
<span class='c'>#&gt;   1. └─global::foo()</span>
<span class='c'>#&gt;   2.   └─global::bar()</span>
<span class='c'>#&gt;   3.     └─1 %&gt;% identity() %&gt;% baz()</span>
<span class='c'>#&gt;   4.       ├─base::withVisible(eval(quote(`_fseq`(`_lhs`)), env, env))</span>
<span class='c'>#&gt;   5.       └─base::eval(quote(`_fseq`(`_lhs`)), env, env)</span>
<span class='c'>#&gt;   6.         └─base::eval(quote(`_fseq`(`_lhs`)), env, env)</span>
<span class='c'>#&gt;   7.           └─`_fseq`(`_lhs`)</span>
<span class='c'>#&gt;   8.             └─magrittr::freduce(value, `_function_list`)</span>
<span class='c'>#&gt;   9.               ├─base::withVisible(function_list[[k]](value))</span>
<span class='c'>#&gt;  10.               └─function_list[[k]](value)</span>
<span class='c'>#&gt;  11.                 └─global::baz(.)</span>
</code></pre>

</div>

This clutter is now completely resolved:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'>foo</span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; Error: oh no</span>

<span class='nf'>rlang</span><span class='nf'>::</span><span class='nf'><a href='https://rlang.r-lib.org/reference/last_error.html'>last_trace</a></span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; &lt;error/rlang_error&gt;</span>
<span class='c'>#&gt; oh no</span>
<span class='c'>#&gt; Backtrace:</span>
<span class='c'>#&gt;     █</span>
<span class='c'>#&gt;  1. ├─global::foo()</span>
<span class='c'>#&gt;  2. │ └─global::bar()</span>
<span class='c'>#&gt;  3. │   └─1 %&gt;% identity() %&gt;% baz()</span>
<span class='c'>#&gt;  4. └─global::baz(.)</span>
</code></pre>

</div>

Speed
-----

The pipe is now written in C to improve the performance. Here is a benchmark for the old R implementation:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>f1</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f2</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f3</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f4</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>

<span class='nf'>bench</span><span class='nf'>::</span><span class='nf'><a href='http://bench.r-lib.org/reference/mark.html'>mark</a></span><span class='o'>(</span>
  `1` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span>,
  `2` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span>,
  `3` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f3</span><span class='o'>(</span><span class='o'>)</span>,
  `4` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f3</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f4</span><span class='o'>(</span><span class='o'>)</span>,
<span class='o'>)</span>
<span class='c'>#&gt;   expression     min  median `itr/sec` mem_alloc `gc/sec` n_itr  n_gc</span>
<span class='c'>#&gt;   &lt;bch:expr&gt; &lt;bch:t&gt; &lt;bch:t&gt;     &lt;dbl&gt; &lt;bch:byt&gt;    &lt;dbl&gt; &lt;int&gt; &lt;dbl&gt;</span>
<span class='c'>#&gt; 1 1           59.4µs  68.9µs    13648.      280B     59.1  6004    26</span>
<span class='c'>#&gt; 2 2           82.6µs 101.6µs     9252.      280B     42.8  3894    18</span>
<span class='c'>#&gt; 3 3          106.4µs 124.7µs     7693.      280B     18.8  3690     9</span>
<span class='c'>#&gt; 4 4          130.9µs 156.1µs     6173.      280B     18.8  2956     9</span>
</code></pre>

</div>

The new implementation is less costly, especially with many pipe expressions:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'>bench</span><span class='nf'>::</span><span class='nf'><a href='http://bench.r-lib.org/reference/mark.html'>mark</a></span><span class='o'>(</span>
  `1` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span>,
  `2` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span>,
  `3` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f3</span><span class='o'>(</span><span class='o'>)</span>,
  `4` <span class='o'>=</span> <span class='kc'>NULL</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f3</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f4</span><span class='o'>(</span><span class='o'>)</span>,
<span class='o'>)</span>
<span class='c'>#&gt;   expression      min   median `itr/sec` mem_alloc `gc/sec` n_itr  n_gc</span>
<span class='c'>#&gt;   &lt;bch:expr&gt; &lt;bch:tm&gt; &lt;bch:tm&gt;     &lt;dbl&gt; &lt;bch:byt&gt;    &lt;dbl&gt; &lt;int&gt; &lt;dbl&gt;</span>
<span class='c'>#&gt; 1 1            2.16µs   3.11µs   306145.        0B     61.2  9998     2</span>
<span class='c'>#&gt; 2 2            2.68µs   3.85µs   246869.        0B     74.1  9997     3</span>
<span class='c'>#&gt; 3 3            3.22µs   4.55µs   207548.        0B     83.1  9996     4</span>
<span class='c'>#&gt; 4 4            3.88µs   5.25µs   180807.        0B     72.4  9996     4</span>
</code></pre>

</div>

We don't generally except this to have much impact on typical data analysis code, but it might yield meaningful speed ups if you are using the pipe inside very tight loops.

Laziness
--------

R core has expressed their interest in adding a native pipe in the next version of R and are working on an implementation[^1]. The main user-visible change in this release makes magrittr more compatible with the behaviour of the base pipe by evaluating the expressions lazily, only when needed.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>ignore_arguments</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>...</span><span class='o'>)</span> <span class='s'>"value"</span>

<span class='kr'><a href='https://rdrr.io/r/base/stop.html'>stop</a></span><span class='o'>(</span><span class='s'>"foo"</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>ignore_arguments</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [1] "value"</span>
</code></pre>

</div>

This has subtle implications but should be backward compatible with existing pipelines that run without error. The main source of behaviour change is that some code that previously failed may stop failing if the latter part of the pipeline specifically handled the error.

Similarly, warnings that were previously issued might now be suppressed by a function you're piping into. That's because the following expressions are now almost completely equivalent:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='c'># Piped</span>
<span class='kr'><a href='https://rdrr.io/r/base/warning.html'>warning</a></span><span class='o'>(</span><span class='s'>"foo"</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'><a href='https://rdrr.io/r/base/warning.html'>suppressWarnings</a></span><span class='o'>(</span><span class='o'>)</span>

<span class='c'># Nested</span>
<span class='nf'><a href='https://rdrr.io/r/base/warning.html'>suppressWarnings</a></span><span class='o'>(</span><span class='kr'><a href='https://rdrr.io/r/base/warning.html'>warning</a></span><span class='o'>(</span><span class='s'>"foo"</span><span class='o'>)</span><span class='o'>)</span>
</code></pre>

</div>

Thanks to this change, you will now be able to pipe into testthat error expectations, for instance:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'><a href='http://testthat.r-lib.org'>testthat</a></span><span class='o'>)</span> <span class='o'>%&gt;%</span>
  <span class='nf'><a href='https://rdrr.io/r/base/message.html'>suppressMessages</a></span><span class='o'>(</span><span class='o'>)</span>

<span class='o'>{</span> <span class='m'>1</span> <span class='o'>+</span> <span class='s'>"a"</span> <span class='o'>}</span> <span class='o'>%&gt;%</span>
  <span class='nf'><a href='https://testthat.r-lib.org/reference/expect_error.html'>expect_error</a></span><span class='o'>(</span><span class='s'>"non-numeric argument"</span><span class='o'>)</span>
</code></pre>

</div>

Note that one consequence of having a lazy pipe is that the whole pipeline will be shown on the call stack before any errors are thrown:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>f1</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f2</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f3</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>
<span class='nv'>f4</span> <span class='o'>&lt;-</span> <span class='kr'>function</span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span> <span class='nv'>x</span>

<span class='kr'><a href='https://rdrr.io/r/base/stop.html'>stop</a></span><span class='o'>(</span><span class='s'>"oh no"</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f1</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f2</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f3</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>f4</span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; Error in f1(.) : oh no</span>

<span class='nf'>rlang</span><span class='nf'>::</span><span class='nf'><a href='https://rlang.r-lib.org/reference/last_error.html'>last_trace</a></span><span class='o'>(</span><span class='o'>)</span>
<span class='c'>#&gt; &lt;error/rlang_error&gt;</span>
<span class='c'>#&gt; oh no</span>
<span class='c'>#&gt; Backtrace:</span>
<span class='c'>#&gt;     █</span>
<span class='c'>#&gt;  1. ├─stop("oh no") %&gt;% f1() %&gt;% f2() %&gt;% f3() %&gt;% f4()</span>
<span class='c'>#&gt;  2. ├─global::f4(.)</span>
<span class='c'>#&gt;  3. ├─global::f3(.)</span>
<span class='c'>#&gt;  4. ├─global::f2(.)</span>
<span class='c'>#&gt;  5. └─global::f1(.)</span>
</code></pre>

</div>

The last function of the pipeline is `f4()`, so that's the first one to be run. It evaluates its argument which is provided by `f3()`, so that's the second function pushed on the stack. And so on until `f1()` needs the result of [`stop("oh no")`](https://rdrr.io/r/base/stop.html) which causes an error.

**Update 2020-09-22:** Another issue caused by laziness is that if any function in a pipeline returns invisibly, than the whole pipeline returns invisibly as well.

``` r
1 %>% identity() %>% invisible()
1 %>% invisible() %>% identity()
1 %>% identity() %>% invisible() %>% identity()
```

This is consistent with the equivalent nested code. This behaviour can be worked around in two ways. You can force visibility by wrapping the pipeline in parentheses:

``` r
my_function <- function(x) {
  (x %>% invisible() %>% identity())
}
```

Or by assigning the result to a variable and return it:

``` r
my_function <- function(x) {
  out <- x %>% invisible() %>% identity()
  out
}
```

Towards a release
-----------------

Though we have changed the behaviour of the pipe, there should be no impact on your user code. The laziness makes it possible to use the pipe in more situations but is not any stricter. It should only cause problems in very rare corner cases and these should be minor. To confirm our analysis, we ran reverse dependency checks for magrittr, purrr, tidyr, dplyr, and tidymodels. Only a dozen out of the 2800 packages were broken by the new implementation, and fixing them is generally easy (see the breaking changes section of the [NEWS file](https://github.com/tidyverse/magrittr/blob/master/NEWS.md)).

**Update 2020-09-22:** The issue you're most likely to encounter is that using [`return()`](https://rdrr.io/r/base/function.html) inside `{` inside [`%>%`](https://magrittr.tidyverse.org/reference/pipe.html) is no longer supported. If you do this, you will see this error:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='m'>1</span> <span class='o'>%&gt;%</span> <span class='o'>{</span>
  <span class='kr'>if</span> <span class='o'>(</span><span class='nv'>.</span> <span class='o'>&gt;</span> <span class='m'>0</span><span class='o'>)</span> <span class='o'>{</span>
    <span class='kr'><a href='https://rdrr.io/r/base/function.html'>return</a></span><span class='o'>(</span><span class='nv'>.</span><span class='o'>)</span>
  <span class='o'>}</span>
  <span class='nv'>.</span> <span class='o'>+</span> <span class='m'>1</span>
<span class='o'>}</span>

<span class='c'>#&gt; Error in 1 %&gt;% {: no function to return from, jumping to top level</span>
</code></pre>

</div>

Despite these few corner cases, we are confident that this release should be seamless for the vast majority of users. But, to be extra sure, we'd be grateful for any additional testing on real-life scripts with this development version. Please let us know of any issues you find with this new version of the pipe, if any.

Finally, if you're interested in the design tradeoffs involved in the creation of a pipe operator in R, see the [tradeoffs](https://magrittr.tidyverse.org/articles/tradeoffs.html) vignette. Any comments about the choices we have made are welcome.

[^1]: See Luke Tierney's [keynote](https://youtu.be/X_eDHNVceCU?t=3099) at the useR! 2020 conference

