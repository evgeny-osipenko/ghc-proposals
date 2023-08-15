Associativity over another operator
==============

.. author:: Evgeny Osipenko
.. date-accepted:: Leave blank. This will be filled in when the proposal is accepted.
.. ticket-url:: Leave blank. This will eventually be filled with the
                ticket URL which will track the progress of the
                implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::


This proposal introduces a new syntactical feature to binary operators in Haskell, ``infix over``. This is a new associativity mode, where consecutive occurences of the operator are resolved by repeating the operation in pairs and combining the results with another operator.

As an example::

    -- given
    infix 4 <= over &&
    within a b c =
        a <= b <= c   -- is rewritten into   (a <= b) && (b <= c)


Motivation
----------
A somewhat frequent problem in programming is to test whether a number falls within the given bounds, or whether a finite sequence of values adheres to a specific ordering.

In some programming languages, this test can be written compactly, for example, Python allows ``(a <= b) && (b <= c)`` to be written as ``a <= b <= c`` instead.

In current Haskell, however, there is no such mechanism. This proposal aims to remedy this, while trying to keep the scope of changes to a minimum, but still with a reasonably general mechanism.


Proposed Change Specification
-----------------------------
Syntax changes
~~~~~~~~~~~~~~
In addition to the three existing associativities (left-, right- and non-associative), define a fourth one, "associativity over".

In code, this associativity is defined with a fixity declaration of this form::

    infix n <1>, <2>, <3> over <.>

Where ``n`` stands for the associativity rank, ``<1>, <2>, <3>`` refer to the operators that the declaration applies to, literal word ``over`` distinguishes the new syntax, and ``<.>`` is a reference (possibly qualified) to the combining operator.

This declaration defines operators ``<1>``, ``<2>`` and ``<3>`` as "associative over ``<.>``".

Given a sequence of binary applications at the same rank, where each operator in the sequence is "associative over" the same operator ``^`` (compared using its fully-qualified symbol name), this sequence is rewritten as follows:

1. For each instance of the associated operators, construct an expression consisting of only a single application of this operator to its immediately neighbouring arguments.

   Each argument in the sequence, except for the very first and the last ones, will appear in the resulting list twice: once on the right side of the receding operator, and once on the left side on the succeeding one.

2. Combine the resulting pairwise applications into a new sequence, using repeated applications of ``^``. If there are more than two expressions in the new sequence, use associativity for ``^``, potentially executing this algorithm recursively.

For example, given the fixity declaration above, the following expression::

    a <1> b <2> c <3> d

Will be rewritten into::

    (a <1> b) <.> (b <2> c) <.> (c <3> d)

After which the associativity rules for ``<.>`` will be triggered.

Note that, even though the given algorithm is recursive, it is guaranteed to terminate — as the number of the repeated applications decreases by one with each iteration.

``base`` changes
~~~~~~~~~~~~~~~
Once the feature is supported by the compiler, some of the operators in ``Prelude`` will have their definitions changed to take advantage of it::

    infix 4 <, <=, >, >=, ==, /= over &&


Examples
--------
Once the specified changes are implemented, the consecutive applications of the comparison operators will become legal in Haskell, and behave as they usually do in mathematics::

    within a b c   =   a <= b <= c
    eq3 a b c   =   a == b == c

At the same time, the proposed implementation doesn't tie the mechanism to only these oparators on the level of the compiler, and therefore can be also used by library authors for similar or completely different purposes.

As an example, the same mechanism can be used with `beam-core <https://hackage.haskell.org/package/beam-core-0.10.1.0/docs/Database-Beam-Query.html#t:SqlEq>`_\'s expression builders::

    infix 4 ==., /=., <., <=., >., >=. over &&.
    infix 4 ==?., /=?. over &&?.
    -- allows writing
    select $ do
        t1 <- all_ myTable
        t2 <- all_ myTable
        guard_ (val_ lowerBound <=. value t1 <=. value t2 <=. val_ upperBound)
        pure (t1, t2)


Effect and Interactions
-----------------------
As this proposal adds a new associativity mode, it will induce changes to all places where the associativity information is given to the user:

* In `template-haskell <https://hackage.haskell.org/package/template-haskell-2.20.0.0/docs/Language-Haskell-TH-Syntax.html#t:FixityDirection>`_, ``FixityDirection`` datatype will gain a new constructor::

    data FixityDirection
        = InfixL
        | InfixR
        | InfixN
        {-| The `Name` field is the fully-qualified name of the combining operator -}
        | InfixOver Name

* In `base <https://hackage.haskell.org/package/base-4.18.0.0/docs/GHC-Generics.html#t:Associativity>`_, in ``GHC.Generics.Associativity`` datatype will have to be parametrized or split, and will gain a new constructor::

    data Associativity' str
        = LeftAssociative
        | RightAssociative
        | NotAssociative
        {-| The three `Symbol`s comprise the fully-qualified name of the combining operator.

            @\'AssociativeOver "op" "pkg-0.1" "Mod.Submod"@ refers to @pkg-0.1:Mod.Submod.op@.

            The order of the fields is the same as in `MetaData`. -}
        | AssociativeOver str str str

    type Associativity = Associativity' String
    type AssociativityI = Associativity' Symbol


Costs and Drawbacks
-------------------

Give an estimate on development and maintenance costs. List how this affects
learnability of the language for novice users. Define and list any remaining
drawbacks that cannot be resolved.


Backward Compatibility
----------------------

* The new fixity declaration gives meaning to syntax that is currently invalid.

  As a more specific example, to give a fixity to a function ``over``, one has to backquote-wrap it to turn an identifier into an operator::

      infix 7 `over`

* Changing an operator's associativity from "none" to "over smth" also only gives meaning to presently invalid expressions.

The main breakage hazard lies in generic programs that read off and make decisions based on fixity information:

* In the most benevolent scenario, they will continue to work as usual with declarations (functions and constructors) that don't involve the new associativity mode, and fail gracefully upon encountering one that does.

* In code that analyzes data constructors' fixity with ``GHC.Generics``, changing ``GHC.Generics.Associativity`` to have two versions (value-level and type-level) may turn it invalid.

* In a more dangerous case, a program might handle some associativity modes explicitly and others with a fallback case, where it makes assumptions that are no longer true::

      -- in TH code:
      case thFixity of
          InfixL -> smth
          InfixR -> smth
          _ -> assuming (thFixity == InfixN) smth -- assumption no longer true

      -- in Generics code:
      instance GSmth ('InfixI 'NotAssociative n) a
      instance {-# OVERLAPPABLE #-} GSmth ('InfixI astv n) a
          -- assuming astv is either 'LeftAssociative or 'RightAssotiative, which is no longer true

* A somewhat esoteric case may be a Template Haskell program that produces `unresolved infix expressions <https://hackage.haskell.org/package/template-haskell-2.20.0.0/docs/Language-Haskell-TH-Syntax.html#g:3>`_ and specifically relies on some of them failing to resolve, with this program then breaking due to an unexpected success. Such programs, if there are any, are most likely to be found in test suites as error-triggering examples.


Alternatives
------------
The main main objective of this proposal is to make writing certain expressions more convenient, for example, a range check as ``a <= b <= c``. Without this feature, the same expression can be written out explicitly, albeit more verbously, as ``(a <= b) && (b <= c)``.


Unresolved Questions
--------------------
* What should be the behavior if, when rewriting an associated-over operator sequence, a complex expression happens to be in the middle? Should it be duplicated literally, or pulled out into a let-expression outside the sequence?
  ::
      a < someExprOver b < someExprOver c < d
      -- can be rewritten into
      (a < someExprOver b) && (someExprOver b < someExprOver c) && (someExprOver c < d)
      -- or into
      let {b' = someExprOver b; c' = someExprOver c} in {(a < b') && (b' < c') && (c' < d)}

  Since Haskell by design assumes referential transparency, this decision would mostly impact performance.

  This question might not even matter in the end, if the Core optimizer later decides to overrule the decision anyway.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other resources
and prerequisites are required for implementation?

Endorsements
-------------
(Optional) This section provides an opportunity for any third parties to express their
support for the proposal, and to say why they would like to see it adopted.
It is not mandatory for have any endorsements at all, but the more substantial
the proposal is, the more desirable it is to offer evidence that there is
significant demand from the community.  This section is one way to provide
such evidence.
