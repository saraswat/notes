# A simple account of (constraint) logic programming

This is a reprise of the essence of the [CLP paper](http://dl.acm.org/citation.cfm?id=41635).

### Programs

Let the category of `Terms`, `Constraints`, `Atoms`, `Rules` and `Queries` be defined thus:

```
(Terms) s, t ::= X | f(t1,..., tn)
(Constraint) c ::= s=t | c,c
(Atoms) a := p(t1,..., tn)
(Rules) r ::= a :- a1, ..., an.
(Query) q ::= :- a1, ..., an.
```

Here, `X` ranges over variables -- identifiers that start with an upper case letter. `p` and `f` range over constants -- 
identifiers that start with a lower case letter. `n >= 0`, and we will omit parentheses if `n=0`, and write a rule `a :-` . as `a.`.
A query is said to be _empty_ if `n=0`, and is written as `:-.` For terms and atoms `n `is the _arity_.

A _Prolog program_ is given by a set of rules. A _Datalog program_ is a Prolog program in which no term has 
arity `> 0` (atoms are allowed to have positive arities).

Here is an example of a program.
```prolog
/*1*/ borders(nj,ny).
/*2*/ borders(ny, ct).
/*3*/ borders(ny,ma).
/*4*/ borders(ct,ma).
/*5*/ borders(me,ma).
/*6*/ state(S):- borders(S, T). 
/*7*/ connected(S1,S2) :- borders(S1,S2).
/*8*/ connected(S1,S2) :- borders(S2,S1).
/*9*/ path(S1,S2):- connected(S1,S2).
/*10*/ path(S1,S2) :- path(S1, S3), path(S3, S2).
```
(Prolog conventions: everything between “/*” and “*/” is a comment.) It contains a database of facts about which 
state borders which state (rules 1 through 5). Rule 6 asserts that `S` is a state if there is some `T` such that `S` borders `T`.  
Rules 7 and 8 specify that `S1` is connected to `S2` if they border each other (in any order). 
Rules 9 and 10 state that there is a path from `S1` to `S2` if they are connected through one or more border relations.

This is a Datalog program.

## Executions

A _substitution_ `S` is a mapping of variables to terms. We are only concerned with mappings whose domain is finite, and 
that are _idempotent_ (that is for any variable `X`, `S(X) = S(S(X))`, e.g. `S` maps `X` to `f(a,Y)`, and `Z` to `Y`). 
This disallows substitutions which map `X` to `Y` and `Y` to `Z`. Instead the substitution should map `X` to `Z` and `Y` to `Z`.
A substitution whose range contains only variables is said to be a _renaming_. (e.g. `[X-> Y, U->Y]` is a renaming, but `[X->a, U->Y]` 
is not, and here I have used an evident shorthand for finite mappings). A term `s` is said to be _an instance of_ a term `t` if there is some substition `S` s.t. `s=S(t)`.

A constraint `s=t` is _consistent_ if there is some substitution `S` s.t. `S(s)=S(t)` (`S` is also called a _unifier_, 
since it “unifies” (= makes the same) `s` and `t`. A set `g` of constraints is consistent if there is a substitution 
`S` s.t. for every `s=t` in `g`, `S(s)=S(t)`. The set of all such substitutions is called the _solution set_ of `g`, `[g]`. 
The _restriction_ of the solution set of `g` to a set of variables `V`, `[g] | V`, is obtained by restricting the 
domain of each substitution in `[g]` to `V`. For `V` a set of variables, we say that two sets of constraints 
`g` and `h` are `V`-equivalent if `[g] | V = [h] | V`.

We can describe the execution of a Prolog program `P` as follows. The _current state_ of the system is given by a pair 
`(query | store)`, where the `store` is a set of constraints. The initial state is the query `q` given by the user and 
the empty store, i.e. `(q | {})`. The set of variables of the current state `u`, `var(u)`, is the union of the variables 
in the query and store.

Execution progresses through application of the _execution rule_:
>Given the current state `u=(:- a1, ..., an | c)`, execution moves to the state `u'=(:- a1, ..., ai-1, b1, ..., bk, ai+1, ..., an | ai=h,c)` 
>provided that `h :- b1, ..., bk` is a “fresh copy” of a rule `r` in `P`, and the constraint set `{ai=h,c}` is consistent. 

We say that `ai` is the _chosen goal_, and `r` is the _chosen rule_.  (A fresh copy `r'` of a rule `r` is obtained by applying a 
renaming `S` to `r`; `S` is chosen so that its range is distinct from the set of variables of the current state.)

Note that if execution moves from state `u` to state `u'` then `var(u)` is a subset of `var(u')`; the newly introduced variables 
come from the rule and are called _local variables_. Computation _terminates_ when no more move is possible in the current state `(q, s)`. 
When this happens, and `q` is empty, execution is said to _succeed_, with answer `[s] | V`, where `V` is the set of variables of the 
original query. Otherwise execution is said to _fail_. It is possible that a query `q` can have many executions, some of which fail 
and some of which succeed with different answers. We say that `q` _fails_ if all its executions fail.

We can succinctly represent an execution as a trace, a sequence `z=(a1,b1), (a2,b2), ...., (ak,bk)` of pairs of numbers where 
`ai` is the index of the chosen goal (in the state `si`), and `bi` is the index of the chosen rule (assume all rules are ordered from 
`1` to `n`, where there are `n` rules), or `0` if no rule can be chosen. Here we do not record the renaming used to get a fresh copy, 
since any renaming will do. Stated differently, given `z` one can construct an underlying execution sequence; further two such 
successful sequences will have the same answer.

Here is an example of a (partial) execution sequence for the above program. The initial query is `?- path(nj, ct).` 
We wish to know if there is a path from `nj` (New Jersey)  to `ct` (Connecticut). 
```prolog
(path(nj,ct) | )
(path(S1,S3), path(S3,S2) | path(nj,ct)=path(S1,S2) )
(connected(S11,S31),path(S3,S2) | path(S1,S3)=path(S11,S31),path(nj,ct)=path(S1,S2))
(borders(S111,S311),path(S3,S2) | connected(S11,S31)=connected(S111,S311),path(S1,S3)=path(S11,S31),path(nj,ct)=path(S1,S2))
(path(S3,S2) | borders(S111,S311)=borders(nj, ny), connected(S11,S31)=connected(S111,S311),path(S1,S3)=path(S11,S31),path(nj,ct)=path(S1,S2))
```
It has the corresponding trace `(1,10), (1,9), (1,7), (1,1)`.

When writing execution sequences it is much more convenient to represent the store in a simplified form. 
Let `V` be the variables in the original query. We can replace any state `u = (q, s)`, with `u' = (q, s')`, where `s'` is such that 
`([s] | W) = ([s'] | W)` for `W = V u var(q)`, without losing any information. (Basically we can ignore constraints on 
local variables if they do not get reflected in a constraint on a variable in an atom in the current query or a variable 
in the original query.) 

Using this, we can simply the above execution sequence as follows. Note that the initial query `(path(nj,ct))` has no variables 
(`V= {}`).
```prolog
(path(nj,ct) | )
(path(S1,S3), path(S3,S2) | S1=nj, S2=ct)
(connected(S11,S31),path(S3,S2) | S11=nj, S31=S3, S2=ct)
(borders(S111,S311),path(S3,S2) | S111=nj, S311=S3, S2=ct) 
(path(S3,S2) | S3=ny, S2=ct)
And now you should be able to see how this execution sequence can be continued:
(connected(S31,S21) | S31=ny, S21=ct)
(borders(S311,S211) | S311=ny, S211=ct)
( | )
```

Where in the last step we were able to simplify to the empty set of constraints because `V={}` and there are no goals left, 
hence `var(q)={}`. (Note that this will always be the case if the original query did not have any variables, and the 
execution sequence is successful.) Thus the trace for the proof that there is a path from `nj` to `ct` is:
```
(1,10), (1,9), (1,7), (1,1), (1, 9), (1,7), (1,2)
```

### Higher-level representation of traces.

Depending on what is desired it is possible, of course, to represent this information in a more abstract way. 
Each step simply involves pairing an atom in the query of the current state with a rule (if there is such a rule).
Thus the trace above could equivalently have been represented by:
```prolog
(path(nj,ct), (path(nj,ct):- path(nj,S3),path(S3,ct)))
(path(nj,S3), (path(nj,S3):- connected(nj,S3)))
(connected(nj,S3), (connected(nj,S3):- borders(nj,S3)))
(borders(nj,S3), (borders(nj,ny)))
(path(ny,ct), (path(ny,ct):- connected(ny,ct)))
(connected(ny,ct), (connected(ny,ct):- borders(ny,ct)))
(borders(ny,ct), (borders(ny,ct)))
```
Note that in the above the second component of a pair is not just a fresh renaming of a rule from the program, 
it is actually an instance of that rule (obtained by applying a substitution) such that the head of the rule is 
an instance of the first component of the pair. I have written it like this just for convenience -- once the 
first component is selected and the rule is selected the instance of the rule is uniquely determined (upto renaming of the variables
in the rule).

Here is an example of a trace in which the original query contains some variables:
```prolog
(path(X,ct), (path(X,ct):- path(X,S3),path(S3,ct)))
(path(X,S3), (path(X,S3):- connected(X,S3)))
(connected(X,S3), (connected(X,S3):- borders(X,S3)))
(borders(X,S3), (borders(nj,ny)))
(path(ny,ct), (path(ny,ct):- connected(ny,ct)))
(connected(ny,ct), (connected(ny,ct):- borders(ny,ct)))
(borders(ny,ct), (borders(ny,ct)))
```
Note that at step 4, `X` was unified with `nj` and `S3` (a local variable) with `ny`. This is the substitution needed to unify
`borders(X,S3)` and `borders(nj,ny)`. At no other step does the selected goal have variables 
that occur in the original query. Hence the “answer” associated with this query is `[X=nj]`. 

Here is an example of a failed execution. Recalled that in a failed execution for no goal in the last state can a rule be chosen
(while preserving consistency). We indicate this using the symbol `:-` (stands for inconsistency, `false :- true.`) in the second 
place of the last configuration.
```prolog
(path(nj,ct), (path(nj,ct):- connected(nj,ct)))
(connected(nj,ct), (connected(nj,ct):- borders(nj,ct)))
(borders(nj,ct), :-)
```

## Additional Notes.

Constraints were introduced above to make it easier to describe the operational semantics, without having to resort to 
excessively specialized notions such as “most general unifiers”. It should be clear that the above presentation can 
easily be generalized to rules with constraints in the body, and the notion of consistency of constraints can be 
defined more abstractly without reference to substitutions.

Briefly, we let rules be of the form:
```
(Rules) r ::= a :- c1, ... ck : a1, ..., an.
```
and keep the notion of configurations the same as before and change the transition rule to:

>Given the current state u=(:- a1, ..., an | c), execution moves to the state 
>`u'=(:- a1, ..., ai-1, b1, ..., bk, ai+1, ..., an | ai=h, c1,..., cm, c)` provided that 
>`h :- c1,...,cm : b1, ..., bk` is a “fresh copy” of a rule `r` in `P`, and the constraint set 
>`{ai=h, c1,..,cm, c}` is consistent.
