---
layout: post
title: 'Problem solving: Validating ECS graph'
date: 2022-07-06 15:15:00 GMT+3
categories: [Tutorials, Algorithms]
tags: [Tutorials, C++, Algorithms, ECS]
math: true
---

Designing algorithms is quite rare task, so it is always feels great when you can apply this skill to solve some
practical problem. In this post I'll talk about ECS graph validation algorithm that I've written for
[Emergence project](https://github.com/KonstantinTomashevich/Emergence).

### Problem definition

Let's start from defining properties of ECS system:

- List of resources to which system has read-only access.
- List of resources to which system has write access.
- List of systems that are dependencies of this system.

ECS graph is a set of systems where all references are internal. That means that if system A depends on system B and
system A is a part of the graph than system B should be part of the graph too. It is called graph because it is usually
visualized as directed graph of systems where dependencies are edges: if system A depends on system B than there
is an edge from B system node to A system node. To make sure that graph is valid and can be executed we need to 
check that:

- There are no deadlocks caused by dependencies.
- There are no race conditions: when one system modifies resource other systems should not be able to access it.
- All the references in system lists (resources and dependencies) are valid.

Checking that all references in system properties are valid is trivial, so we will ignore this validation step here.

### Applying theory

To design an algorithm we firstly need to define what we're doing mathematically. Let's start from our ECS graph:
it is directed graph $$ G (V, E) $$, where $$ V $$ is equal to the set of all systems in a graph. Then set of edges
$$ E $$ is defined like that:

$$ \forall A, B \in V \hspace{1em} \exists! (A, B) \in E \Leftrightarrow A \in dependencies(B) $$

Now lets translate deadlock check into more math-friendly variant. If deadlock happens in ECS graph context it means 
that system is waiting for dependency that cannot be finished. In this context it can happen only if system depends
on itself, because every system must be finishable by definition. Such dependency may arise only if $$ G $$ has
any cycle, so to pass this verification check $$ G $$ must be an **acyclic** graph.

Race condition happens when one system reads or modifies resource, for example component storage, while other system
also modifies this resource. System dependencies must be specified in a way that prevents such race conditions from
happening. So, how do we check that there is no system accesses resource $$ R $$ while system $$ A $$ is modifying 
this resource? Concurrent access to one resource $$ R $$ can be prevented by dependencies in two ways:

$$ \forall A, B \in V \hspace{1em} \exists path (A \to B) \rightarrow \nexists RaceCondition $$

$$ \forall A, B \in V \hspace{1em} \exists path (B \to A) \rightarrow \nexists RaceCondition $$

On top of that we can build race condition criteria:

$$ \forall A, B, R \hspace{1em} modifies (A, R) \land accesses (B, R) \land \nexists path (A \to B) 
\land \nexists path (B \to A) \Leftrightarrow \exists RaceCondition $$

I won't bother readers with formal proof because it's kind of boring. The main idea here is that absence of 
dependencies between any task that modifies $$ R $$ and any task that accesses $$ R $$ leads to a race condition.
You can draw several graphs yourself to get a grip of this idea.

### Solution

We've found out what we need to do mathematically, now it is time to implement it!

Let's start from cycle detection: I've decided to use 
[depth-first search](https://en.wikipedia.org/wiki/Depth-first_search) based graph traversal. Each node will be marked
with one of 3 markers: `Unvisited`, `InStack` or `Verified`, and all nodes will be marked as `Unvisited` from the start.
It's better to explain how this works through pseudocode:

```
// This is our recursive visitor for DFS traversal.
function VisitSystem (system)
{
    // If node is alrady verified, we can be sure that 
    // there is no cycles that involve this node.
    if (marks[system] == Verified)
    {
        return CycleNotFound;
    }

    // We've reached the node which children we're currently 
    // iterating. It means that node can be reached from itself
    // and therefore is a part of the cycle.
    if (marks[system] == InStack)
    {
        return CycleFound;
    }

    // Mark that we're currently checking dependant systems
    // from this node.
    marks[system] = InStack;

    // Recursively visit all the dependant systems 
    // (dependency -> dependant edges).
    for (dependantSystem : GetDependantSystems (system))
    {
        if (VisitSystem (dependantSystem) == CycleFound)
        {
            return CycleFound;
        }
    }

    // We've recursively visited all graph nodes that can be 
    // reached from this node and found no cycles. Therefore,
    // we can say that this node is successfully verified.
    marks[system] = Verified;
    return CycleNotFound;
}

// This is our entrypoint to recursive VisitSystem-based traversal.
function SearchForCycles ()
{
    // To traverse the whole graph we need to visit every 
    // system without dependencies.
    for (starterSystem : GetSystemsWithoutDependencies ())
    {
        if (VisitSystem (starterSystem) == CycleFound)
        {
            return CycleFound;
        }
    }

    return CycleNotFound;
}
```

That's all for our cycle detection, but what about race conditions? At first glance their detection looks much less
straighforward. Of course, we could just bruteforce this and search path from every system, but it would be too
inefficient. Thankfully, we could modify our visitation algorithm from cycle detection check to collect all the 
reachable nodes for every node!

```
function VisitSystem (system)
{
    if (marks[system] == Verified)
    {
        return CycleNotFound;
    }

    if (marks[system] == InStack)
    {
        return CycleFound;
    }

    marks[system] = InStack;
    for (dependantSystem : GetDependantSystems (system))
    {
        if (VisitSystem (dependantSystem) == CycleFound)
        {
            return CycleFound;
        }

        // Surprisingly, that's the whole change! 
        // I won't strictly prove it here, because it 
        // is obvious after drawing DFS of any directed
        // acyclic graph.
        reachable[system] += dependantSystem;
        reachable[system] += reachable[dependantSystem];
    }

    marks[system] = Verified;
    return CycleNotFound;
}
```

Now $$ path (A \to B) $$ check can be replaced with `reachable[A].contains(B)` check. After that everything is quite
straighforward: using our `reachable` map we can detect all possible race conditions as pairs of mutually unreachable 
nodes and check whether these nodes access or modify the same resource.

### Optimizations

Pseudocode above omits some implementation details that can be crucial for algorithm performance. It is logical,
because it is pseudocode after all. Therefore, I've decided to list these details below:

- Use numeric indices for systems and resources: it allows to make `marks` and `reachable` arrays instead
  of maps, which will significantly speed up operations on them.
- Use bitsets for reachability recording: using bitsets instead of arrays to record reachable nodes would improve
  both performance and memory usage.
- Cache dependant systems for every system: in initial format every system stores its dependencies instead of
  dependant systems which makes `GetDependantSystems` function quite slow. You can reverse dependencies to get
  all dependant systems for all nodes in one iteration before doing graph traversal, which would significantly
  improve performance by making `GetDependantSystems` complexity `O(1)`.

### Implementation in Emergence

Above I've described the algorithm that I've used to verify ECS graph in 
[Flow library](https://github.com/KonstantinTomashevich/Emergence/tree/daf48ca/Library/Public/Flow) for 
[Emergence project](https://github.com/KonstantinTomashevich/Emergence). Of course, this implementation is a bit more
complex and specific, because 
[Flow](https://github.com/KonstantinTomashevich/Emergence/tree/daf48ca/Library/Public/Flow) task registration routine
is more advanced than system registration in our example problem, but it still remains technically the same
algorithm.

Hope you've enjoyed reading! :)
