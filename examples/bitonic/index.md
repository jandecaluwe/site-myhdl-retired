---
title:   Bitonic Sort 
layout:  wide_article
content: []
summary.md: |
    * writing structural MyHDL code using recursion and lists of signals
    * using a reference software implementation
---

Introduction
============

This page focusses on the possibilities to describe hardware structure in
MyHDL. MyHDL uses classic functions for this purpose. 

Before simulation or conversion to Verilog, a design is *elaborated* by running
the Python interpreter. This implies that all structure is flattened out. As a
consequence, the complexity of the structural description has no impact on the
possibility to convert to Verilog.

As an example, this page shows how to describe recursive structures in MyHDL.
Python is very well suited to do this elegantly. But after conversion to
Verilog, all recursion is flattened out, so it doesn't matter whether the
back-end language or tools support it or not.

Specification
=============

We will develop a circuit that sorts an array of integer values. More
specifically, the array to be sorted is the input, and the sorted array the
output. We are looking for a data-flow oriented circuit, where the output
follows the input "combinatorially".

We will use the bitonic sort algorithm. We will not explain it here: for more
info, read this [web page that explains the bitonic sort
algorithm](http://www.inf.fh-flensburg.de/lang/algorithmen/sortieren/bitonic/bitonicen.htm).

For our purposes, it is sufficient to know that the bitonic algorithm is well
suited for a hardware implementation, and that it is recursive in nature. To
develop the MyHDL code, we will start from a reference software implementation.

Reference implementation in Java
================================

The [web page](http://www.inf.fh-flensburg.de/lang/algorithmen/sortieren/bitonic/bitonicen.htm)
mentioned earlier also contains an annotated reference implementation in Java.
We will use this as a starting point to develop the MyHDL code. Therefore, the
original Java code and annotations are quoted literally in the remainder of
this section.

*(start quote)*

In the following, an implementation of bitonic sort in Java is given. The number of elements to be sorted must be a power of 2.

The program is assumed to be encapsulated in a class with the following elements:

```
private static int[] a;         // the array to be sorted
private final static boolean ASCENDING=true, DESCENDING=false; // sorting direction
```

A comparator is modelled by the procedure `compare`, where the parameter `dir`
indicates the sorting direction. If `dir` is `ASCENDING` and `a[i] > a[j]` is
true or `dir` is DESCENDING and `a[i] > a[j]` is false then `a[i]` and `a[j]`
are interchanged.

```
private static void compare(int i, int j, boolean dir)
{
    if (dir==(a[i]>a[j]))
    {
        int h=a[i];
        a[i]=a[j];
        a[j]=h;
    }
}
```

The procedure `bitonicMerge` recursively sorts a bitonic sequence in ascending
order, if `dir = ASCENDING`, and in descending order otherwise. The sequence to
be sorted starts at index position `lo`, the number of elements is `n`.

```
private static void bitonicMerge(int lo, int n, boolean dir)
{
    if (n>1)
    {
        int k=n/2;
        for (int i=lo; i<lo+k; i++)
            compare(i, i+k, dir);
        bitonicMerge(lo, k, dir);
        bitonicMerge(lo+k, k, dir);
    }
}
```

Procedure `bitonicSort` first produces a bitonic sequence by recursively
sorting its two halves in opposite directions, and then calls `bitonicMerge`.

```
private static void bitonicSort(int lo, int n, boolean dir)
{
    if (n>1)
    {
        int k=n/2;
        bitonicSort(lo, k, ASCENDING);
        bitonicSort(lo+k, k, DESCENDING);
        bitonicMerge(lo, n, dir);
    }
}
```

When called with parameters `lo = 0`, `n = a.length()` and `dir = ASCENDING`,
procedure `bitonicSort` sorts the whole array `a`.

```
public static void sort()
{
    bitonicSort(0, a.length(), ASCENDING);
}
```

*(end quote)*

MyHDL implementation
====================

Let's start with some general considerations.

First, note that the Jave code consists of a number of procedures and procedure
calls. In MyHDL, we can map each procedure to a function that models a hardware
module and each procedure call to an instantiation. The Java procedures return
nothing; they are run solely for their side effects. In MyHDL, we will build
the hardware structure by returning the instances that compose each module.

Secondly, note that the Java code sorts the array in-place, and the procedure
parameters consist of the indices of the array elements to be manipulated. In
our data-flow oriented MyHDL solution, we will have to use a separate input and
output array instead. The array will be represented by a list of signals. If a
function performs several transformations, we have to introduce additional
lists of signals internally.

Finally, in the Java code the recursion stops by "doing nothing". In the MyHDL
code, the equivalent is to assign the input to the output signal. We will need
a "feedthrough" circuit for this purpose.

Now we are ready to fill in the details, in the same order as in the original
Java version.

We start by importing the `myhdl` library, and by defining some boolean
constants:

```python
from myhdl import *

DESCENDING, ASCENDING = False, True
```

The `compare` leaf module is a combinatorial circuit with two input and two
output signals. By default, the inputs are "fed through" to the outputs, but
under the same condition as in the Java code, they are interchanged:

```python
def compare(a1, a2, z1, z2, dir):
    
    @always_comb
    def logic():
        z1.next = a1
        z2.next = a2
        if dir == (a1 > a2):
            z1.next = a2
            z2.next = a1
            
    return logic
```

As mentioned, we will also need a feedthrough circuit, the equivalent of "doing
nothing" in the Java code:

```python
def feedthru(a, z):
    
    @always_comb
    def logic():
        z.next = a
        
    return logic
```

Function `bitonicMerge` describes a higher-level module that takes a list of
signals as its input and its output. Instead of manipulating array elements
based on indices as in Java, it generates the output from the input list of
signals. Note how an additional list of signals is introduced in the case of 
`n > 1`, and how the recursion stops with a `feedthru` instance.

```python
def bitonicMerge(a, z, dir):
    
    n = len(a)
    k = n//2
    w = len(a[0])
    
    if n > 1:
        t = [Signal(intbv(0)[w:]) for i in range(n)]
        comp = [compare(a[i], a[i+k], t[i], t[i+k], dir) for i in range(k)]
        loMerge = bitonicMerge(t[:k], z[:k], dir)
        hiMerge = bitonicMerge(t[k:], z[k:], dir)
        return comp, loMerge, hiMerge
    else:
        feed = feedthru(a[0], z[0])
        return feed
```

Function `bitonicSort` is written in a similar fashion:

```python
def bitonicSort(a, z, dir):
    
    n = len(a)
    k = n//2
    w = len(a[0])
    
    if n > 1:
        t = [Signal(intbv(0)[w:]) for i in range(n)]
        loSort = bitonicSort(a[:k], t[:k], ASCENDING)
        hiSort = bitonicSort(a[k:], t[k:], DESCENDING)
        merge = bitonicMerge(t, z, dir)
        return loSort, hiSort, merge
    else:
        feed = feedthru(a[0], z[0])
        return feed
```

Now we can define a top level module, for example to sort an array of 8 values.
There is a slight restriction however. The Verilog convertor cannot handle
lists of signals in a top level interface. (This is related to the fact that
Verilog memories cannot be used as ports.) To make the design suited for
conversion, we have to splice the lists into individual signals:

```python
def Array8Sorter(a0, a1, a2, a3, a4, a5, a6, a7,
                 z0, z1, z2, z3, z4, z5, z6, z7):

    a = [a0, a1, a2, a3, a4, a5, a6, a7]
    z = [z0, z1, z2, z3, z4, z5, z6, z7]
    sort = bitonicSort(a, z, ASCENDING)
    return sort
```

Verification
============

Verifying the sorter design is easy enough. We set up a list of random data
values, assign it to the inputs of the circuit, then sort the data list and
compare it with the outputs.

```python
from random import randrange

from myhdl import *

from bitonic import Array8Sorter

def bench():
    
    n = 8
    w = 4
    
    a0, a1, a2, a3, a4, a5, a6, a7 = inputs = \
        [Signal(intbv(0)[w:]) for i in range(n)]
    z0, z1, z2, z3, z4, z5, z6, z7 = outputs = \
        [Signal(intbv(0)[w:]) for i in range(n)]

    
    inst = Array8Sorter(a0, a1, a2, a3, a4, a5, a6, a7,
                        z0, z1, z2, z3, z4, z5, z6, z7)

    @instance
    def check():
        for i in range(100):
            data = [randrange(2**w) for i in range(n)]
            for i in range(n):
                inputs[i].next = data[i]
            yield delay(10)
            data.sort()
            assert data == outputs

    return inst, check


def test_bench():
    sim = Simulation(bench())
    sim.run()
```

This test bench can be used with a unit test framework such as `py.test`.

Verilog generation and synthesis
================================

The sorter design can be converted to Verilog as usual, with the `toVerilog`
function. See the [Verilog code](verilog.txt) for the result. Note that it is a
"net list of always blocks", with all structure and recursion flattened out.

Of course, you will want to verify that the generated Verilog code is correct.
This can be done by using Verilog co-simulation with the same test bench as
used for the MyHDL code. You can find the details about the procedure
[here](cookbook:sinecomp#verilog_co-simulation).

To get an idea of the hardware implementation characteristics, check out the
[synthesis results](synthesis.txt).
