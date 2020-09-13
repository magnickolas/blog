---
title: "Logic Programming with Prolog"
subtitle: ""
date: 2020-09-13T03:42:00+03:00
draft: false
description: ""

tags: [logic-programming, prolog]
categories: [exploration]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: true
lightgallery: false
license: ""
---

## Introduction

I'm taking a course in mathematical logic & logic programming in my university. And I wanted to play around with first-order logic in practice. As a bonus, I would be more familiar with a logic programming paradigm.

In order to not complicate things, I chose Prolog language and a compiler [GNU Prolog][gprolog].

Let's try to represent some examples involving basic concepts and structures with it.

[Click to see a meme][prolog-meme].

## Applications

### Lists

#### Get the last element

One of the simplest yet not trivial operations on a singly linked list is to get its last element.

In a functional language like Haskell we would implement it like that (omit type signature and exceptions handling):

```haskell
my_last [x] = x
my_last (_:xs) = my_last xs
```

In Prolog we get:

```prolog
% my_last/2 for (X, Y) is true iff
% X is the last element of Y

my_last(X, [X]).
my_last(X, [_|XS]) :- my_last(X, XS).
```

To me, this is visually similar to the functional way. So, submitting a query `?- my_last(X, [3, 1, 7]).` gives us `X=7` as an answer.

Also, we can substitute "the last element" variable by hand and ask Prolog to prove/disprove our goal by submitting it. Example: `?- my_last(2, [5, 2]).` returns `yes`.

#### Reverse a list

Let's move on to another operation: reversing the list (build the list that contains the elements of the initial list in the reverse order).

Again, let's write a functional solution first:

```haskell
my_reverse [] = []
my_reverse (x:xs) = my_reverse xs ++ [x]
```

Now move to Prolog.
First, I want to define an auxiliary predicate that drops the last elements of a list:

```prolog
% all_but_last/2 for (X, Y) is true iff
% X is Y, but with its last element dropped off

all_but_last([], [_]).
all_but_last(X, [H|T]) :-
    all_but_last(Y, T),
    X = [H|Y].
```

No we can express reverse predicate in terms of the predefined predicates `my_last/2` and `all_but_last/2`:

```prolog
% my_reverse/2 for (X, Y) is true iff
% X is reversed Y

my_reverse([], []).
my_reverse(X, [H|T]) :-
    all_but_last(U, X), my_last(V, X),
    V = H, my_reverse(U, T).
```

Let's check: `?- my_reverse(X, [1, 5, 3]).` yields `X = [3,5,1]`.

{{<admonition tip "Observation">}}
For this example, we can notice one thing that the functional program has, but the logic one lacks. That difference has already appeared, but now it is more clear. It's a constructive algorithm.
{{</admonition>}}

##### Summary

**Functional way**: 

Accepts a list as an input, returns the reversed one
- For an empty list the answer is an empty list
- Otherwise, reverse the tail and append the currently first element to its end

**Logic way**:

Accepts two lists, tries to prove statement that the first list is the reversed second one
- If one of the arguments is not a constant, Prolog (not us) searches for its substitution that would satisfy out predicate.

{{<admonition note>}}
The key point in the logic approach is that we haven't told a computer how to reverse a list (built some constructive algorithm by ourselves). Instead, we treated predicate input arguments pair as a potential candidate to satisfy the property of one list in the pair being the reversed version of the other one.
{{</admonition>}}

This observation allows us to do some fancy things, one example is shown in the next subsection.

#### Solution from constraints

Suppose that you want to check if some list satisfying several conditions exists, and if it is, construct it:
- Its first element is 4
- Its last element is 7
- Its fourth element is 9
- It is a palindrome list if we drop the first element
- Contains value 10
- The length equals 12

We submit the following query that reflects these requirements (using predicates from standard library):

```prolog
?- X=[4|_T], last(X, 7), nth(4, X, 9), reverse(_T, _T), member(10, X), length(X, 12).
```

And get what X can be to satisfy the conditions:

```prolog
X = [4,7,10,9,A,B,_,B,A,9,10,7]
X = [4,7,A,9,10,B,_,B,10,9,A,7]
X = [4,7,A,9,B,10,_,10,B,9,A,7]
X = [4,7,A,9,B,C,10,C,B,9,A,7]
```

where A, B, C — variables that defines some elements equality constraints.

### Graphs

#### Find a path in oriented graph

Assume we have a directed graph $G = (V, E)$ and want to find out for some $u, v \in V$ if there exists a path between $u$ and $v$ and find one in the form of 

$$[w_0, w_1, \cdots, w_n],$$ where $w_0=u,\\, w_n=v,\\, (i \ne j \implies w_i \ne w_j)$.

Let's introduce a predicate $\text{edge}$ that expresses the set of edges:
$$\text{edge}(x, y) = x,y \in V \\, \wedge \\, (x,y) \in E$$

We'll hide the fact that the arguments of the predicate are vertices in its meaning.

```prolog
% Our graph structure
% +-->b--+
% |      ↓
% a      d     e-->f
% |      ↑
% +-->c--+

edge(a, b).
edge(a, c).
edge(b, d).
edge(c, d).
edge(e, f).

% path/3 for (X, Y, P) is true iff
% P is a simple path from X to Y in the form of [X, ..., Y]

path(X, Y, P) :- path(X, Y, P, [X]).

% path/4 for (X, Y, P, Visited) is true iff
% P is a simple path from X to Y not visiting
% vertices from Visited except the first one

path(X, X, P, Visited) :- reverse(Visited, P).
path(X, Y, P, Visited) :-
    p(X, Z),
    \+ member(Z, Visited),
    path(Z, Y, P, [Z|Visited]).
```

{{<admonition warning>}}
Here we use an accumulator that stores appeared vertices in a path, then utilize the fact of it being a reversed path. However, even if one only wanted to check a path existance, they wouldn't be able to write just something like `path(X, Y) :- X = Y; p(X, Y); p(X, Z), path(Z, Y).`, because if no path exists (e.g. substite $\text{X}$ with $\text{a}$, $\text{Y}$ with $\text{e}$), nothing can disprove the predicate and we'll get "local stack overflow" error.
{{</admonition>}} 

In particular we're able to find a path from a vertex to another vertex, but additionally we can instantly do some other things now:

{{<admonition example "Examples">}}
- Find a simple path from one vertex to another (in this example from $\text{a}$ to $\text{d}$):
  - Query:
   `?- path(a, d, P), !.`
  - Answer:
   `P = [a,b,d]`
- Find all the simple paths from one vertex to another (in this example from $\text{a}$ to $\text{e}$):
  - Query:
   `?- path(a, e, P).`
  - Answer:
   `P = [a,b,d,e];
    P = [a,c,d,e]`
- Find all the reachable vertices from the given one (in this example from $\text{b}$):
  - Query:
   `?- findall(X, path(b, X, _), _V), sort(_V, V).`
  - Answer:
   `V = [b,d,e]`
{{</admonition>}}

## Conclusion

In this post we've seen how we can apply a logic approach to some prolems and how it's different from imperative/functional ones.

[gprolog]: http://www.gprolog.org/
[prolog-meme]: https://preview.redd.it/sabq5qemg8321.png?width=640&crop=smart&auto=webp&s=cb11053ece09b3c342dfccc4c26bd259ddd387bc
