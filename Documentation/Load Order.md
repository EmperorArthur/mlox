
# How Sorting plugins works:

## Summary
Every time mlox reads through mlox_base.txt it creates an acyclic directed graph of all the mod dependencies.  It then uses a topological sort to convert this graph into a sorted load order for every single mod.  Finally, it filters that giant load order by what mods are currently active/installed and spits that out as the "Sorted Load Order."

## Details

1) mlox sorts your load order, based on the observation that:
* orderings for plugins make up a partially ordered set.
* A partially ordered set can be sorted by a topological sort.
* A topological sort requires a DAG (Directed Acyclic Graph).
** So we need to avoid putting cycles in our graph.
* A topological sort can have many valid solutions.
** So we try to pick the solution we'd most like to see.
*** by forcing following (the [NearEnd] rule)
*** maintaining previous order (pseudo-rules)
*** by root picking (the [NearStart] rule)

### orderings for plugins make up a partially ordered set.

Let's say we have plugins A, B and C. We may know that A needs to precede B in
the load order. But A and B have nothing to do with C, so it doesn't matter if
A and B come before or after C. This is the essence of a partial ordering. And
we can represent the ordering relationships in a graph, where "->" is called
an "edge" and A -> B means: "A comes before B".

### A partially ordered set can be sorted by a topological sort.

The topological sort is a well known solution to sorting a partially
ordered set.
(See: http://en.wikipedia.org/wiki/Topological_sorting)

#### A topological sort requires a DAG (Directed Acyclic Graph).
 So we need to avoid putting cycles in our graph.

In our example, we have this (called a graph):
A -> B
C

If we were to add this new edge to our graph: B -> A, it would produce a
cycle, meaning that if you follow the edges from A -> B -> A, you've come back
to your starting point.

To avoid cycles, all we have to do while building our graph from the input
rules is to check before we add an edge to see if it will produce a cycle, and
if so, we discard it. When mlox does its add_edge(X, Y) function (which adds
the edge: X -> Y), it first checks via a depth first search to see if we can
already reach Y from X (i.e. that some path of edges exists, such that Y ->
... -> X). If so, adding X -> Y would produce a cycle, and so that edge is
discarded.

### A topological sort can have many valid solutions.

If you have a set of A, B and C, and you know that A comes before B which
comes before C, then you have a "total ordering", because every item knows
it's ordering with respect to every other item in the set. If you have the
same set, but only know that A -> B (A comes before B, a partial ordering),
then the following results of a sort are all valid:

Solution 1: A, B, C
Solution 2: A, C, B
Solution 3: C, A, B

because in all three solutions, A comes before B, in contrast with a total
ordering, where there is only one solution.

### So we try to pick the solution we'd most like to see.

Well, it turns out that human beings dislike change :) so since the user of
mlox is a human being, we want to ensure that the sorted load order differs no
more from the original order than is dictated by the ordering rules in our
rule-base.


### by forcing following (the [NearEnd] rule)

For the NearEnd rules, we use a method of forcing plugins marked with NearEnd
to follow all other active plugins.

This is accomplished by adding an edge from every plugin in the current load
order to each NearEnd plugin. That is, we force each NearEnd plugin to follow
all other plugins in the user's load order. Of course, when we add these
edges, we omit any that might cause a cycle just like in the process of
creating pseudo-rules from the current load order discussed in the next
section.

### maintaining previous order (pseudo-rules)

We also try to pick the most desirable solution by maintaining as best we can,
the initial sorted order.

Returning to our example, if the original order was: B, A, C, and our ordering
rule is: A -> B, then a nice result would be: A, B, C, even though the other
solutions we enumerate above are just as valid.

mlox tries to achieve this by using the original order as a set of
"pseudo-rules", which, for our example would be: B -> A, and A -> C.

So, putting it all together, what mlox does is:

First, reads rules from the user rule-base (mlox_user.txt), then the main
rule-base (mlox_base.txt), and finally applies pseudo-rules from the original
load order. And of course, it does the cycle detection and discarding as we
proceed along.

So we take the defined rule first:

A -> B

then add the next rule from the pseudo-rules:

A -> B
B -> A

But that is a cycle, so we chuck the B -> A, then add the next
pseudo-rule:

A -> B
B -> C

And that's okay, no cycle detected.
And if we apply those to the set B, A, C, we end up with:

A, B, C

*** by root picking (NearStart)

A root is a node in the graph that has no incoming edges. In our example
graph, A and C are roots. B is not a root because we know the ordering:
A -> B. (B is a child of A), and so it has an incoming edge.

We allow some control over the topological sort by allowing the user to
specify nodes (plugins) that they would prefer to see near the start of the
sorted order. I call this "root-picking", because the first step of the
topological sort is to collect all the root nodes of the graph in a list. In a
standard topological sort there is no priority of one root over another. But
by introducing the [NearStart] rules, we can let the user choose roots that
should be near the start of the sorted order or near the end.

In our example, the user could have said that B was a [NearStart]. So what
mlox does is it finds root of the subgraph that B belongs to (in this case,
that would be root A). and it would choose that root as the first root.


# Pitfalls to be aware of
We prefer:

    [Order]
    a
    b

    [Order]
    a
    c


instead of:

    [Order]
    a
    b
    c

Because the graph looks like:

    a->b
    a->c

instead of:

    a->b
    b->c

If using the wrong method, then any rule in the future that wants 'c' to be ahead of 'b' will cuase problems.
Also, while the load order might work, the fact that 'c' depends on 'b' instead of 'a' makes debugging issues that much harder.

When mlox is creating the graph, it throws out duplicates whenever it sees a rule twice.

Example:

    [Order]
    x
    a
    b
    c

    [Order]
    y
    a
    b       <- ignored
    c       <- ignored

is the same as:

    [Order]
    x
    a

    [Order]
    y
    a

    [Order]
    a
    b
    c

Both create the graph:

    x->a
    y->a
    a->b
    b->c

However, the second method is both more clear, and is faster.
