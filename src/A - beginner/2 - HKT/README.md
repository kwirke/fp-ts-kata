# Higher-Kinded Types (HKT)

Higher-kinded types (kinds) are genera, metatypes. A kind is a type of types, or metatype: In the same way that a set of values forms a type, a set of types forms a kind. For more details on kinds, you can check [Wikipedia](https://en.wikipedia.org/wiki/Kind_(type_theory)).

In order to define a kind in TypeScript, it is necessary to specify in the language that the generic interface or class parameter is itself a generic type. Unfortunately, in TS there is no such possibility. We cannot express something like this:

```ts
interface Functor<F> {
  // Error TS2315: Type 'F' is not generic.
  map: <A, B>(f: (a: A) => B) => (fa: F<A>) => F<B>;
}
```

`fp-ts` uses findings from the article [Lightweight higher-kinded polymorphism](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf) in order to define higher-kinded types, as well as the [module augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) mechanism from TypeScript. To do so, you can use the type `Kind<F, A>`.

## Notation

When you see the `Kind<F, A>` expression, think of it as a way to write down `F<A>` when `F` is not polymorphic.
An excerpt from the `fp-ts` documentation:

> The `Kind<F, A>` internally uses the `URItoKind` dictionary, so that it is able to define an abstract data type for a specific data type. So if `URI = 'Identity'`, then `Kind<URI, number>` is `Identity<number>`.

A detailed description of the steps to be taken is available in [the fp-ts documentation](https://gcanti.github.io/fp-ts/recipes/HKT.html).

# Result Container

In this kata we will get acquainted with another type of container: Result. Result expresses the idea of ​​calculations, which can fail with an error or return the result:

```ts
// Type definition:
type Failure<E> = Readonly<{ tag: 'Failure', error: E }>;
type Success<A> = Readonly<{ tag: 'Success'; value: A }>;
export type Result<E, A> = Failure<E> | Success<A>;
// Constructors:
export const failure = <E, A>(error: E): Result<E, A> => ({ tag: 'Failure', error });
export const success = <E, A>(value: A): Result<E, A> => ({ tag: 'Success', value });
```

## Bifunctor

When we have a type with two parameters, like `Result<E, A>`, then we can implement for it an instance of the bifunctor type. This type class defines two operations: `mapLeft`, which allows you to perform a functor's `map` for the left (erroneous) part of the container, and a `bimap`, which performs simultaneous mapping of one of the two functions `f` and `g`, depending on the state of the container:

```ts
interface Bifunctor<E, A> {
  mapLeft: <R>(f: (e: E) => R) => (fea: Result<E, A>) => Result<R, A>;
  bimap: <R, B>(f: (e: E) => R, g: (a: A) => B) => (fea: Result<E, A>) => Result<R, B>;
}
```

The laws of a bifunctor are similar to the laws of a functor, only with the amendment that the `bimap` must comply with the composition law for both its arguments.

A good description of a bifunctor with picture diagrams of `bimap` behavior is given in [Bifunctors by Mark Seemann](https://blog.ploeh.dk/2018/12/24/bifunctors/).

# The Task

In this kata, you need to implement a functor, monad, applicative, alt, and bifunctor in the form of HKT for the Result container.

> Think about why we cannot write an instance of the alternative class for Result, but we can write an instance of the Alt class, which defines only the `alt` method, but does not define the `zero` method.
>> An important lesson is: a type class implementation means an *unambiguous* and *unique* implementation. If we can implement a method from a type class in several ways, it means either we are thinking of a container specialization (for example, Validation is a specialized implementation of Either that is left-associative), or it is impossible to implement this type class in principle.

# Conclusion

In this kata you got acquainted with the concept of the type of types, as well as the way they are represented in TypeScript. You can draw the following conclusions:
1. `Kind<F, A>` expressions are equivalent to `F<A>`, but do not require `F` to be a generic type in the current context.
2. With the `Kind` type, you can also represent types with an arity greater than 1. What is `Kind2<F, L, A>` equivalent to? And the expression `Kind3<F, U, L, A>`?
Bifunctor allows you to simultaneously convert containers of type "A or B", saving resources on consecutive calls .map().mapLeft .
3. Bifunctor allows you to simultaneously convert containers of type «A or B», and avoids having to do two consecutive `.map().mapLeft()` calls every time.
4. Not all type classes can be implemented for each container.
