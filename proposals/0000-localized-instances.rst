Localized instances
==============

.. author:: Evgeny Osipenko
.. date-accepted::
.. ticket-url::
.. implemented::
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

We propose adding a new attribute to an instance, which would control its export/import behavior: global or localized.

"Global instance" refers to the Haskell2010 behavior: an instance is automatically exported and imported, with no way to hide or suppress it.

"Localized instance", on the other hand, behaves like most other top-level declarations: it must be added to the export list to become visible externally, and during import, it can be suppressed by an explicit import list. To facilitate it, we add new syntax to give a name to an instance, and to refer to it in an export/import list.

For example: ::

    module Foo
        ( Foo (..)
        , showText
        , defaultFoo
        ) where
    
    import Data.Text (Text, pack)
    
    data Foo = Foo Int Int
    
    showText :: (Show a) => a -> Text
    showText = pack . show
    
    defaultFoo :: Foo
    defaultFoo = Foo 10 20
    
    --------------------
    
    module ShowingFoo
        ( instance showFooAsTuple
        , instance showFooWithX
        ) where
    
    import Foo
    
    instance showFooAsTuple :: Show Foo where
        show (Foo a b) = show (a, b)
    
    instance showFooWithX :: Show Foo where
        show (Foo a b) = show a ++ "x" ++ show b
    
    -- textBad :: Text
    -- textBad = showText defaultFoo
    -- error: overlapping instances for `Show Foo` (both are in scope)
    
    --------------------
    
    module TextA
        ( textA
        ) where
    
    import Foo
    import ShowingFoo (instance showFooAsTuple)
    
    -- only instance showFooAsTuple gets imported from ShowingFoo
    -- since it's not mentioned in the export list, it is not re-exported
    
    textA :: Text
    textA = showText defaultFoo
    -- textA = "(10, 20)"
    
    --------------------
    
    module TextB
        ( textB
        ) where
    
    import Foo
    import ShowingFoo (instance showFooWithX)
    
    -- only instance showFooWithX gets imported from ShowingFoo
    -- since it's not mentioned in the export list, it is not re-exported
    
    textB :: Text
    textB = showText defaultFoo
    -- textB = "10x20"
    
    --------------------
    
    module ManyTexts
        ( textA
        , textB
        , textC
        ) where
    
    import Foo
    import TextA
    import TextB
    
    -- TextA and TextB don't export any instances
    
    -- if a localized instance isn't intended for export, it can be nameless:
    deriving stock instance _ :: Show Foo
    
    textC :: Text
    textC = showText defaultFoo
    -- textC = "Foo 10 20"


Motivation
----------
The main goal of this proposal is to control the spread of orphan instances.

Currently, once an orphan instance is declared, it becomes visible in *all* transitive dependencies of the module, which may lead to some problems:

1. An excessively general orphan instance may overlap with non-orphans declared later.

   As an example, `Crypto.JOSE.Error <https://hackage.haskell.org/package/jose-0.9/docs/Crypto-JOSE-Error.html#section.orphans>`_ defines an orphan instance like this: ::
   
       instance (
           MonadRandom m
         , MonadTrans t
         , Functor (t m)
         , Monad (t m)
         ) => MonadRandom (t m) where
           getRandomBytes = lift . getRandomBytes

   Naturally, if we then wanted to make a monad transformer that provides its own implementation, the instances would overlap: ::
   
       newtype RandomT m a = RandomT (IORef ChaChaDRG -> m a)
       
       instance {-# OVERLAPPING #-} (MonadIO m) => MonadRandom (RandomT m) where
           getRandomBytes n = RandomT $ \drgRef ->
             liftIO $ atomicModifyIORef drgRef $ \drg1 ->
               let (result, drg2) = randomBytesGenerate drg1
                in (drg2, result)

   We are required to add ``{-# OVERLAPPING #-}`` to override the orphan.

2. If module A defines an orphan, module B imports A and uses the orphan for its code, and then module C imports B - it may become reliant on the orphan by accident without importing A explicitly.

   As an example, in the same package, `Crypto.JOSE.Types <https://hackage.haskell.org/package/jose-0.9/docs/src/Crypto.JOSE.Types.html>`_ imports `Test.QuickCheck.Instances <https://hackage.haskell.org/package/quickcheck-instances>`_. If we use JOSE to build something in our application, and then write property tests for our code - we're very likely to become dependent on the QuickCheck's orphan instances, such as ``Arbitrary Text``, without even realizing it.

With "localized instances", it would become possible for libraries to define and use such instances internally, without polluting their transitive reverse dependencies.

As an additional use case, named localized instances could be used to configure the behavior of ``DeriveAnyClass``-able instances, as an alternative to ``DerivingVia``, for example: ::

    module Data.Aeson where
    
    class ToJSON a where
        toJSON :: a -> Value
        default toJSON :: JSONPolicy => a -> Value
        toJSON = genericToJSON jsonPolicyOptions
    
    class JSONPolicy where
        jsonPolicyOptions :: Options
    
    --------------------

    module Data.Aeson.Policy
      ( instance asTaggedObject
      , instance asObjectWithSingleField
      ) where
    
    instance asTaggedObject :: JSONPolicy where
        jsonPolicyOptions = defaultOptions
    
    instance asObjectWithSingleField :: JSONPolicy where
        jsonPolicyOptions = defaultOptions {sumEncoding = ObjectWithSingleField}
    
    --------------------
    
    module DomainA where
    
    import Data.Aeson
    import Data.Aeson.Policy (instance asTaggedObject)
    
    data Optional r = None | Some r
        deriving anyclass (ToJSON, FromJSON)
    
    x = encode (Some 10)
    -- x = "{\"tag\":\"Some\",\"contents\":10}"
    
    --------------------
    
    module DomainB where
    
    import Data.Aeson
    import Data.Aeson.Policy (instance asObjectWithSingleField)
    
    data Result e r = Failure e | Success r
        deriving anyclass (ToJSON, FromJSON)
    
    x = encode (Success 10 :: Result () Int)
    -- x = "{\"Success\":10}"
    
    --------------------
    
    module DomainC where
    
    import Data.Aeson
    
    instance _ :: JSONPolicy where
        jsonPolicyOptions = UntaggedValue
    
    instance _ :: {-# OVERLAPPING #-} (ToJSON a, ToJSON b) => ToJSON (Either a b)
    
    data SomeData = SomeData Double (Either String Int)
        deriving anyclass (ToJSON)
    
    x = encode (SomeData 2.3 (Right 10))
    -- x = "[2.3, 10]"
    
    


Proposed Change Specification
-----------------------------
For each typeclass instance, we add a new attribute: locality, which could be *global* or *localized*.

We change the way instances are exported and imported. *Global* instances are always exported and imported whenever available. *Localized* instances are controlled similar to other top-level declarations, with a more detailed description below.

The syntax of instance declaration is extended with a *binder* of the form: ::

    inst_binder
      ::= var '::'
        | '_' '::'
        | {- empty -}

Thus, the instance syntax becomes: ::

    inst_decl
      ::= 'instance' inst_binder overlap_pragma inst_type where_inst

Stand-alone derived instances can also be given a binder: ::

    stand_alone_deriving
      ::= 'deriving' deriv_standalone_strategy 'instance' inst_binder overlap_pragma inst_type

Instances without a binder are global. Instances with the binder are localized (including the binder ``_ ::``).

Examples: ::

    -- global instance
    instance (Show a) => Show (Wrapped a) where
        {decl; decl; decl}

    -- localized instance, bound to name "showFoo"
    instance showFoo :: {-# OVERLAPPABLE #-} (Show a) => Show (Wrapped a) where
        {decl; decl; decl}

    -- localized instance, not bound to a name
    instance _ :: {-# INCOHERENT #-} Show (Wrapped a) where
        {decl; decl; decl}

    -- localized instance, bound to name "showFooDefault"
    deriving instance showFooStock :: (Show a) => Show (Wrapped a)
    
    -- localized instance, bound to name "showFooId"
    deriving newtype instance showFooId :: (Show a) => Show (Wrapped a)

The syntax of export/import lists is extended to include localized instances: ::

    export
      ::= qcname_ext export_subspec
        | 'module' modid
        | 'pattern' qcon
        | 'instance' qvar
        
    qcname_ext_w_wildcard
      ::= qcname_ext
        | 'instance' qvar
        | '..'

There are two ways to export a localized instance: as a standalone definition, and attached to a data or class declaration (similar to pattern exports).

Example: ::

    module MyModule
      ( instance myInst
      , MyData (.., instance myInst)
      , MyClass (.., instance myInst)
      ) where
    
    data MyData = MyData
    
    class MyClass a where
      someFun :: a -> a
    
    instance myInst :: MyClass MyData where
      someFun = id

To insert the instance into a downstream module's resolution context:

1. The exporting module must be imported unqualified.

2. The associated import list must satisfy at least one of these:

   a. be absent (that is, if we import everything from the module - we import all its localized instances too);
   
   b. contain the ``instance`` clause on the top level;
   
   c. contain a reference to a data or a class, that has the desired instance attached, and include it in the subspec:
   
      - as an explicit ``instance`` clause,
      
      - or with the ``..`` wildcard.

If the import is given a ``hiding`` list, the condition is inverted: the instance is imported if, after removing the word ``hiding``, the resulting declaration would not import it in any way.

The attachment behavior should be consistent with pattern synonyms.

Continuing the ``MyModule`` example: ::

    module User where
    
    -- imported
    import MyModule
    import MyModule (instance myInst)
    import MyModule (MyData (instance myInst))
    import MyModule (MyData (..))
    import MyModule (MyClass (instance myInst))
    import MyModule (MyClass (..))
    -- not imported
    import MyModule hiding (instance myInst)
    import MyModule (MyData (MyData))
    import MyModule (MyData, MyClass (someFun))
    import MyModule ()
    -- doesn't import regardless of an import or hiding list
    import qualified MyModule

The new syntax is enabled by a new extension ``LocalizedInstances``.

Examples
--------

Effect and Interactions
-----------------------

Costs and Drawbacks
-------------------

Alternatives
------------

Unresolved Questions
--------------------

Implementation Plan
-------------------
