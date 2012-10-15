---
layout: overview-large
title: TypeTags and Manifests

partof: reflection
num: 5
---


A type tag encapsulates a representation of type T.
Type tags replace the pre-2.10 concept of a [[scala.reflect.Manifest]] and are integrated with reflection.

=== Overview and examples ===

Type tags are organized in a hierarchy of three classes:
[[scala.reflect.ClassTag]], [[scala.reflect.api.Universe#TypeTag]] and [[scala.reflect.api.Universe#WeakTypeTag]].

Examples:
  {{{
  scala> class Person
  scala> class Container[T]
  scala> import scala.reflect.ClassTag
  scala> import scala.reflect.runtime.universe.TypeTag
  scala> import scala.reflect.runtime.universe.WeakTypeTag
  scala> def firstTypeArg( tag: WeakTypeTag[_] ) = (tag.tpe match {case TypeRef(_,_,typeArgs) => typeArgs})(0)
  }}}
  TypeTag contains concrete type arguments:
  {{{
  scala> firstTypeArg( implicitly[TypeTag[Container[Person]]] )
  res0: reflect.runtime.universe.Type = Person
  }}}
  TypeTag guarantees concrete type arguments (fails for references to unbound type arguments):
  {{{
  scala> def foo1[T] = implicitly[TypeTag[Container[T]]]
  <console>:11: error: No TypeTag available for Container[T]
         def foo1[T] = implicitly[TypeTag[Container[T]]]
  }}}
  WeakTypeTag allows references to unbound type arguments:
  {{{
  scala> def foo2[T] = firstTypeArg( implicitly[WeakTypeTag[Container[T]]] )
  foo2: [T]=> reflect.runtime.universe.Type
  scala> foo2[Person]
  res1: reflect.runtime.universe.Type = T
  }}}
  TypeTag allows unbound type arguments for which type tags are available:
  {{{
  scala> def foo3[T:TypeTag] = firstTypeArg( implicitly[TypeTag[Container[T]]] )
  foo3: [T](implicit evidence$1: reflect.runtime.universe.TypeTag[T])reflect.runtime.universe.Type
  scala> foo3[Person]
  res1: reflect.runtime.universe.Type = Person
  }}}
  WeakTypeTag contains concrete type arguments if available via existing tags:
  {{{
  scala> def foo4[T:WeakTypeTag] = firstTypeArg( implicitly[WeakTypeTag[Container[T]]] )
  foo4: [T](implicit evidence$1: reflect.runtime.universe.WeakTypeTag[T])reflect.runtime.universe.Type
  scala> foo4[Person]
  res1: reflect.runtime.universe.Type = Person
  }}}


[[scala.reflect.api.Universe#TypeTag]] and [[scala.reflect.api.Universe#WeakTypeTag]] are path dependent on their universe.

The default universe is [[scala.reflect.runtime.universe]]

Type tags can be migrated to another universe given the corresponding mirror using

 {{{
 tag.in( other_mirror )
 }}}

 See [[scala.reflect.api.TypeTags#WeakTypeTag.in]]

=== WeakTypeTag vs TypeTag ===

Be careful with WeakTypeTag, because it will reify types even if these types are abstract.
This makes it easy to forget to tag one of the methods in the call chain and discover it much later in the runtime
by getting cryptic errors far away from their source. For example, consider the following snippet:

{{{
  def bind[T: WeakTypeTag](name: String, value: T): IR.Result = bind((name, value))
  def bind(p: NamedParam): IR.Result                          = bind(p.name, p.tpe, p.value)
  object NamedParam {
    implicit def namedValue[T: WeakTypeTag](name: String, x: T): NamedParam = apply(name, x)
    def apply[T: WeakTypeTag](name: String, x: T): NamedParam = new Typed[T](name, x)
  }
}}}

This fragment of the Scala REPL implementation defines a `bind` function that carries a named value along with its type
into the heart of the REPL. Using a [[scala.reflect.api.Universe#WeakTypeTag]] here is reasonable, because it is desirable
to work with all types, even if they are type parameters or abstract type members.

However if any of the three `WeakTypeTag` context bounds is omitted, the resulting code will be incorrect,
because the missing `WeakTypeTag` will be transparently generated by the compiler, carrying meaningless information.
Most likely, this problem will manifest itself elsewhere, making debugging complicated.
If `WeakTypeTag` context bounds were replaced with `TypeTag`, then such errors would be reported statically.
But in that case we wouldn't be able to use `bind` in arbitrary contexts.

=== Backward compatibility with Manifests ===

Type tags correspond loosely to manifests.

More precisely:
The previous notion of a [[scala.reflect.ClassManifest]] corresponds to a scala.reflect.ClassTag,
The previous notion of a [[scala.reflect.Manifest]] corresponds to scala.reflect.runtime.universe.TypeTag,

In Scala 2.10 class manifests are deprecated, and manifests are planned to be deprecated in one of the
subsequent point releases. Therefore it's advisable to migrate manifests to tags.

In most cases it is enough to replace `ClassManifest` with `ClassTag` and `Manifest` with `TypeTag`.
There are however a few caveats:

1) Tags don't support the notion of `OptManifest`. Tags can reify arbitrary types, so they are always available.

2) There's no equivalent for `AnyValManifest`. Consider comparing your tag with one of the base tags
   (defined in the corresponding companion objects) to find out whether it represents a primitive value class.
   You can also use `<tag>.tpe.typeSymbol.isPrimitiveValueClass`.

3) There's no replacement for factory methods defined in `ClassManifest` and `Manifest` companion objects.
   Consider assembling corresponding types using the reflection APIs provided by Java (for classes) and Scala (for types).

4) Certain manifest functions (such as `<:<`, `>:>` and `typeArguments`) weren't included in the tag API.
   Consider using the reflection APIs provided by Java (for classes) and Scala (for types) instead.

 === Known issues ===

 Type tags are marked as serializable, but this functionality is not yet implemented.
 An issue tracker entry: [[https://issues.scala-lang.org/browse/SI-5919 https://issues.scala-lang.org/browse/SI-5919]]
 has been created to track the implementation of this feature.

