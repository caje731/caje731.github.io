---
last_modified_at: 2025-06-04T09:57:38+02:00
categories:
  - Blog
tags:
  - performance engineering
  - python
  - C
  - programming
  - technology
  - natural language processing
  - semantic analysis
---

*This is material from a lightning-talk I gave at a [PyPizza Berlin 2019](https://2019.berlin.python.pizza/?ref=python.pizza)*

---

At [TrustYou](https://www.trustyou.com/), we deal with heaps of review texts from hotel guests, terabytes of data from all over the Internet. Our goal is to help hoteliers and chains understand what their guests think about their services and how to improve the guest experience so that people like you can have an even more pleasant vacation next time.

In order to analyze the huge pile of unstructured, plain text reviews that we sit on, we have a project called **Sema** that provides a semantic analysis of the review text content, and is therefore at the core of our internal processing.

Sema can tell you that “*I hate this hotel*” conveys a fairly negative perception of the overall place by a guest, whereas “*they have a lovely pool with a gorgeous view on the Alps*” is quite the opposite and improves the performance rank of the hotel by increasing the score of its pool category.

For the text processing inside of Sema, we use a library called [*NLTK*, the Natural Language ToolKit](https://www.nltk.org/), which is one of the great libraries that make Python such a versatile ecosystem. While there are faster text processing tools for Python available (especially [*spaCy*](https://spacy.io/), which we started using in recent projects), there is none that supports so many languages out of the box. At TrustYou, we process reviews in more than 20 languages by now, including Croatian 🇭🇷, Thai 🇹🇭, Japanese 🇯🇵, and Mandarin 🇨🇳.

Now, processing the text of millions of reviews that we crawl into our system every week takes a lot of time, and, sadly, NLTK is better known for its feature set than for its speed. It was originally written for research and teaching purposes and is entirely implemented in Python, and all three properties add up to its processing overhead.

---

Finding the problem
===================

We recently spent some time profiling specifically the Japanese text processing, because we saw from our monitoring that it is much slower than other languages, especially the English text processing. We knew that the speed would eventually become problematic, so we decided to look into it.

For profiling Python code, the usual tool is cProfile. It comes with a nice command line interface (in addition to its API), which allows you to just take a Python script and run it.

`$ python -m cProfile -s time myscript.py`
![profiling output](https://miro.medium.com/max/700/1*IojhpWij3sR6w4AysKYOyw.png "Profiling a Sema script that analyses a corpus of Japanese reviews")

As it turns out, most of the time is spent inside of the NLTK library, and specifically in just two modules: `nltk.parse.chart` and `nltk.grammar`. Both are parts of the language parser in NLTK, which uses our custom set of grammars to make sense of review texts.

Nicer profiles
==============

Numbers are a great start. To make this profile more accessible, we use SnakeViz, an in-browser application that visualises Python run profiles. To use it, simply run these commands:

`$ pip install snakeviz`  
`$ python -m cProfile -o profiledump.prof myscript.py`  
`$ snakeviz profiledump.prof`

It will automatically open a new tab in your browser that looks similar to this:
![snakeviz profile](https://miro.medium.com/max/700/1*qSLO0_xEtQDYKlIVw9EN_A.png)

It allows you not only to sort and search the profile as you please, but it also gives you a nice interactive visualization of the call stack - very useful for drilling down into specific calls and identifying problems.

Faster, Faster!
===============

Now, NLTK is an old library, and a lot of work has probably gone into improving its speed already. So looking for low hanging fruit for speeding up the parser is not very likely to lead anywhere. What we did instead was to use the [Cython](https://cython.org/) compiler for translating the bottleneck modules from (interpreted) Python into native code.

`$ pip install Cython`  
`$ cythonize -i -3 nltk/parse/chart.py nltk/grammar.py`

This translates the Python 3 code into C modules (right next to the `.py` files), and then runs `distutils` and `gcc` to compile them into extension modules - shared native libraries that can be imported from Python.

In order to validate that this improves the performance, we enabled profiling support in the modules by passing the compiler directive `-X profile=True` to Cython:

`$ cythonize -f -i -3 -X profile=True nltk/parse/chart.py nltk/grammar.py`

and then ran the profiler over it again, to see how the profile changed. Profiling support is disabled in Cython-compiled modules by default to further improve the performance. Thus the need for a directive.

![improved profile](https://miro.medium.com/max/700/1*m4Xow1I2gClBD_kK5GDQwA.png "The new profile using Cython-compiled modules")

As you can see, while the two modules are still at the top of the list, they account for much less time now, which leads to an overall speedup of more than 30%. And we changed no lines in our code!

This encourages us to build and use our own binary package of NLTK in which we will include the two compiled modules, by simply running the `cythonize` command over it. This should speed up our Sema processing by a couple of hours per day – and we didn't have to change a single line of code in NLTK either!

We obviously also filed [an issue](https://github.com/nltk/nltk/issues/2093)
<img src="https://github.githubassets.com/images/icons/emoji/octocat.png" width="20" height="20"/> against NLTK to integrate the compilation into their build process.

Good to Remember
================

... if you're going down this path:

* Build separate binary wheel packages for each Python version you support.
* Make packages available for each platform/os you need
* Target an old platform for backwards-compatibility

As mentioned, NLTK is a pure Python package that runs in Python 2.x and 3.x. That makes it very easy to ship it as a single package that installs and works on all current Python versions.

Once you start compiling modules from it, however, the resulting binaries will no longer work across different Python versions, and separate binary wheel packages will need to be prepared and shipped for each supported Python version, e.g. 2.7, 3.5, 3.6, 3.7. Obviously, the binaries are also platform specific, which means that this set of packages is needed for each platform now. And, being native code, care has to be taken that the C compiler generates code that runs on all CPUs that should be supported. You *may* also want to avoid using recent CPU instructions only available in the latest processors. The latter can be done by targeting an old CPU, e.g. by setting the C compiler flags to something like `CFLAGS="-O3 -march=core2"`.

In our case at TrustYou, the number of different Python versions that we support is low (currently only 2.7 and 3.6/3.7), and the only relevant architecture is Linux, for which distribution independent Linux binaries can be built using the [manylinux](https://github.com/pypa/manylinux) Docker images.

But, since we did not modify the NLTK source code at all, using it uncompiled as it is normally distributed is just fine and leads to the exact same results. Compiling the modules simply makes them faster for us in production.

That's it, thanks for reading!