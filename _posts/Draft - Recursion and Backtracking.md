<!-- ---
layout: post
title: "Recursion and Backtracking"
author: "Krithik Vaidya"
tags: [dsa, algorithms, recursion, backtracking]
image: A-Comprehensive-Guide-to-LiFi/1.jpeg
--- -->

## Introduction

One of my motivations to write this article is to primarily get a grip on the topics of Recursion, Backtracking and consequently, Dynamic Programming. These are topics that have always been hard for me to get confidence in. There will be two articles, this one focusing on Recursion and Backtracking, and the next one on Dynamic Programming.

## What is Recursion

Recursion is a problem solving paradigm in which the problem to be solved is broken down in such a way that it's solution can be formed from a smaller subproblem, and the solution of that subproblem can be formed from an even smaller subproblem, and so on. This goes on until the recursion converges to a base case, which is solved directly (i.e. without involving subproblems). A properly designed recursive algorithm ensures progress by making sure that the recursive steps always converge to the base case(s), for all inputs.

## What is Divide-and-Conquer

Divide-and-Conquer algorithms are kind of a subset of Recursive problems (but not exactly), wherein the problem is decomposed into two or more smaller independent problems of the same kind, until the instances are small enough to solve directly. Then, the solutions of these instances are combined together to eventually give the final solution. Examples of this category of algorithms include Merge Sort and Quick Sort.

## How is Divide-and-Conquer Different from Recursion

As stated before, Recursion is kind of a superset of Divide-and-Conquer. There may be only one sub-problem at each step (as in Binary Search, where we consider only on section of the array at each step, discarding the other), the subproblems may not be independent (like in Dynamic Programming) and the subproblems may be of different kinds (like in Regular Expression matching). Divide and Conquer problems may also be implemented with iteration rather than recursion, and hence we can't say that divide and conquer algorithms are exactly subsets of recursive algorithms.


## An Example of a Recursive Algorithm

- Calculating `GCD` of `x, y`

If `x > y`, then `GCD (x, y) = GCD (x - y, y)`. This can be proved simply by assuming the GCD as 'd', and writing `x = ad` and `y = bd`. Then we can see that `x - y = d (a - b)` => 'd' is also a divisor of `x - y`.

Using this, we can say that
`GCD (x, y) where x > y = GCD (x % y, y)`, since on repeated application of the above, we end up with `x % y`. Then, we can swap the two numbers obtained above and continue till one of them becomes 0. Then, the required GCD = the other number. This is called Euclid's GCD Algorithm. The function for this would look like:

```cpp

int calc_gcd (int x, int y)
{
    if (y == 0)
        return x;
    return calc_gcd (y, x % y); // notice that x % y and y positions have been swapped here.
}

```

- Complexity analysis

We can get an approximate worst case complexity with the following:
The maximum value of `x mod a`, where `a <= x` is `(x / 2 - 1)` (why?). Using this, we can see that at each recursive step, one of the arguments is at least halved, it implies that the time complexity is  
`O (log x + log y)`, or more simply `O (log max (x,y))`. In other words, the time complexity is `O (n)`, where `n = the number of bits required to represent x or y, whichever is larger`. The space complexity is the same, although we can achieve `O (1)` space complexity by using an iterative implementation.

- Tower of Hanoi Problem.



## References

[Elements of Programming Interviews](https://elementsofprogramminginterviews.com/)