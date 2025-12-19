---
layout: "post"
title:  "Relational Operators: The Building Blocks of Queries"
date:   "2025-12-19 11:23:35 +0530"
categories: database
--- 

# Why Do Databases Need a Query Planner?

You could be forgiven for thinking that a "query optimizer" is a component of a database that takes a query plan and makes it better, hance, the typical programmer definition of "optimize." This is not really how the term is used in practice, and "query optimizer" is really pretty synonymous with "query planner." I think "query planner" is a better term so I'm going to use that.

SQL is fundamentally designed such that users express what data they want rather than an algorithm for retrieving it, and this property gives us I think, the most basic definition of a query planner's job:

> A database's query planner is responsible for translating a query from a declarative language such as SQL into an executable query plan.

This means things like:

* choosing which indexes to scan for a give query.
* choosing the order in which to perform joins, and
* choosing the join algorithm to use for any given join,

plus a myriad of other decisions, with the goal of making the query execute as fast as possible.

That leads us to the paradigm with which we interact with databases:

1. Write a query to extract data,
2. database generates a query plan,
3. database executes query plan.
4. (repeat 0 or more times) decide query plan is too slow, change query and go back to (2).

It's worth noting that step 2 is actually fairly complicated in modern SQL databases and so the feedback loop from step 4 back into step 1 is nontrivial.

A rational performance sensitive programmer can have an adverse reaction to this, but this workflow is fundamental to SQL's declarative nature. So what's the reason for the longevity of declarative query languages?


# Why the Declarative Model Works?

One of the early goals of implementing a relational database was to bring data analysis to people who didn't have the expertise to write a bespoke FORTRAN program to analyze their data.

Both SQL's syntax and semantics descend from this:

* In order to not scare off people without a computer background in the 70s, your language should resemble English as closely as possible.
* Your query language should, as much as is reasonable, and to a larger degree than might be appropriate in other systems, obscure the underlying implementation of the system.

In modern times, there has been a clear bifurcation in the types of workloads people use relational databases for. "OLTP" (On-Line Transactional Processing) workloads emphasize lots of writes and point reads, and possibly very simple and small aggregations or joins, while "OLAP" (On-line Analytics Processing) workloads have few or zero writes and more complex reads, featuring many joins and aggregations.

That these two use-cases are often addressed by the same tools is sort of a fact of convenience. It seems kind of obvious that if we were optimizing our paradigm for these two different styles of workload in isolation, we'd probably end up with at least slightly different languages. As if is though, well, I need a database, I've already got one and I don't want another. If it doesn't do the job well enough I'll complain to the vendor and give them money until it does.

We specifically are interested in thinking about the OLTP use-case, where a query is written once and executed many times, and given that it is no longer the 1970s, it's valid to question if having such a high-level, declarative language in one of the more performance sensitive parts of a system still makes sense.

Immediately the reasonable objections to any high-level language spring to mind:

1. We don't trust the system to effectively translate my high-level query into acceptably fast low-level plans.
2. The amount of abstraction between what the user writes and what actually happens on the hardware makes it harder to understand the consequences of various decisions.
3. If we ever need to take fine-grained control of how my queries execute, it's going to be very difficult because we would be going to have to trick this piece of software that thinks it knows better than us into doing what we want.

Despite this, smart query planner have been a mainstay in relational databases since System R in 1970s, so they must be providing some kind of value. I think there's a handful of mitigating factors for the above.

1. The majority of query planning decisions (say, 97%) are pretty obvious. Modern query planners are still pretty good even at the non-obvious ones. Cases where they mess up catastrophically are overwhelmingly the exception, not the rule.
2. The solution to most problems non-experts encounter is "add an index to the field in your WHERE clause", which is pretty easily learned.
3. Mature query planners have, with some exceptions, gotten pretty good about providing escape hatches for when users need more precise control over how their queries execute.

Whatever the factors that caused query planners to stick around, We are going to make the obnoxious philosophical appeal that an intelligent query planner is actually prerequisite for realizing the vision that Ted Codd had for what the relational model would do.

The very first blurb from `A Relational Model of Data for Large Shared Data Banks` outlines what he wanted hist model to achieve:

> Future users of large data banks must be protected from having to know how the data is organized in the machine (the internal representation). A prompting service which supplies such information is not a satisfactory solution. Activities of users at terminals and most application programs should remain unaffected when the internal representation of data is changed
and even when some aspects of the external representation are changed. Changes in data representation will often be needed as a result of changes in query, update and report traffic and natural growth in the types of stored information.

Codd was interested in achieving what he called `access path independence`, which in less grandiose terms just means he wanted to decouple "what data do I want to retrieve" from "how do I retrieve it."

A database schema is composed of an interface called the `logical schema` and an implementation called the `physical schema`, and it was Codd's belief that queries should only be written to the interface, and never to the implementation.

In a modern SQL system, the logical schema is the set of tables and logical constraints (foreign keys, CHECK constraints, etc) on those tables, while the physical schema is the set of indexes provided for those tables. So, more concretely, if we have access path independence, if indexes are added to or removed from the database's schema, existing queries should:

* automatically become faster if those indexes are useful for that query and
* not cease to be valid because an index they used no longer exists.

This is really more of a language design argument - our query language should work such that the semantic meaning of a query shouldn't depend on the physical layout of the data on disk.

Anyone who has ever carefully turned a database schema will tell you that claiming queries "don't break" if indexes are removed is tenuous, since in practice a query running sufficiently slowly is indistinguishable from it not running at all. There's always going to be a limit to the degree we can decouple logical and physical schema, which is why modern systems tend to conflate them via tools like query plan hints. Despite this, aiming for access path independence as a default is a laudable goal.

# How Query Planners Differ from Compilers?

You can partition the parts of a query planner into two main groups:

1. the parts that are like a compiler, and
2. the parts that are not like a compiler.

This is obviously tautological but it's a useful separation. Ignoring some details, most of the major parts of a compiler are present in a database's query engine. A query will be lexed, parsed, typechecked, translated into an intermediate representation, "optimized", and finally directly translated into a form more suitable for execution.

Since that's more broadly understood we gonna focus on the parts of query planner that differ from a compiler. SO we should ask ourselves, why should a query planner be different from a compiler in the first place? What's sufficiently different about the setting that it's unique in some way?

One thing is that SQL as a language is substantially more limited in scope than a general-purpose programming language, and is intended for use by non-experts. This gives it a higher expectation of "just doing the right thing" than an imperative language whose translation to machine code is fairly obvious.

The much more important difference is that a database is a `closed system`. The life of a query begins and ends in the database itself. Compare this to a compiler, which gets to look only at the code it's compiling and `nothing else`; it has no purvue on the code's actual execution.

Consider this: we are tasked to write a program to perform matrix multiplication on two `n x n` matrices.

There's broadly two approaches we could take for representing our matrices:

1. Flat in memory, or
2. as lists of their nonzero entries.

In the general case, multiplying these is `O(n^3)` regardless, but option (1) is preferable when our matrices are dense, due to less overhead and better use of locality, while option (2) is better when our matrices are sparse, since there's less work overall to be done.

Given these two approaches, which path should a clever programmer choose? it's unclear:

* If we're typically going to have matrices with very few zero entries, the first approach is better since there's less overhead and better use of locality.
* If we're going to have sparse but potentially very large matrices, the second approach is better.

This kind of problem demands the programmer to draw on their domain knowledge and understanding of the problem space to make the right decision of what algorithm to use. There's no mechanical process for knowing that the adjacency matrix of a social graph is typically going to be pretty sparse. This is why human judgement is still necessary.

Early implementors of query planners referred to them as "automatic programmers", and that framing is not so bad.

Let's look more concretely at the kind of decisions faced by planners: we want to plan a query that looks like this:

```
SELECT * FROM t WHERE a = 1 AND b = 2
```

We have an index on `a` and an index on `b`, but not both. So, we have two options:

1. Do a lookup into the `a` index to directly get all the records satisfying `a = 1`, and for each of those results, check if they satisfy `b = 2`.
2. Do a lookup into the `b` index to directly get all the records satisfying `b = 2`, and for each of those results, check if they satisfy `a = 1`.

Which is the better choice? It's unclear: if there are lots of records with `a = 1` and not many with `b = 2`, we'd save ourselves a lot of effort by scanning the `b` index and filtering, but if the reverse is true, we should scan the `a` index.

What if we reframe the problem?

```
SELECT * FROM customers
WHERE address  = "Streeto street" AND first_name = "Clien"
```
with same index setup, where we have an index on each `address` and `first_name`.

Here, a thoughtful human can have a useful insight: it's much more likely that there are multiple people named "Clien" than there are having the address "Streeto street", so for this query we almost certainly want to do a lookup into the `address` index and filter on `first_name`, rather than the other way around (when this is the case, we say "`address` is more selective than `first_name`").

Of course, a piece of software can't draw on its life experience the way a human can to make this kind of observation, and so it must find other ways to accomplish the same goal. Typically this is done by periodically collecting statistics about the shape of the data stored in the database (how big is each table, how many distinct values does each column contain, etc).

This brings use to a slightly more abstract, but more interesting definition of what a planner is supposed to do:

> A planner's job is to approximate domain knowledge that it can exploit to make queries run as fast as possible.

# What Planners Enable in Practice?

Automatically things is good. Pretty much any mechanical process that can produce an outcome that's 80% as good as human is a massive win. Not just because the work the human would do is saved, but because having a process be automated allows it to be used in completely different ways.

Let's look back to our example from before, where we have a query with two conjunctive predicates.

```
SELECT * FROM
customers
WHERE
favorite_food = $1 AND favorite_animal = $2
```

Automating and re-planning a query for each execution gives us the ability to pick different plans for the same query at different points in time, For instance, for an instance of the query where favorite_food = 'spaghetti', that filter is likely to be very nonselective, but favorite_food = 'broccoli' is likely to be very selective. But since we're not locked into a plan when we write the query, we can evaluate each of these options each time the query is run and choose our best guess of the best one.

Automating planning like this, we actually get a `family` of plans from one query, rather than just a single plan. Even if a human is capable of finding a better plan than an automated planners in one case, constructing the family of plans and the rules for choosing between them is a much different task.

Since we're not tying what we want to any particular implementation of how to get it, the planner has the freedom to pick whatever plan makes sense at the time it's being run.

A reasonable summary of all this might be: thinking of automating query planning as automating the job of a one-and-done human planner is the wrong mental model, but thinking about it as a human programmer you have access to on-demand is not so bad.
