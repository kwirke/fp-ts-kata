# Maybe Container

The Maybe container is a way of representing nullable values ​​so that it is possible to work with them in a safe way.

> In `fp-ts` terminology, Maybe is the `Option` data type available in the `fp-ts/lib/Option` module. However, for didactic purposes, I will use the name Maybe for this exercise.

In fact, Maybe is what we call a "discriminated union" (google it!), and it is defined in a very simple way:

```ts
type Nothing = Readonly<{ tag: 'Nothing' }>;
type Just<A> = Readonly<{ tag: 'Just'; value: A }>;
type Maybe<A> = Nothing | Just<A>;
```

For convenience, a pair of constructors is also defined:

```ts
const nothing: Maybe<never> = { tag: 'Nothing' };
const just = <A>(value: A): Maybe<A> => ({ tag: 'Just', value });
```

> **A note about type classes**
>
> From now on I will use the term "type class". All the constructions we will see further in this kata — functors, applicatives, monads — are type classes. You can think of them as interfaces that define a certain behavior. If the type belongs to a type class, then this means that it implements and supports the behavior that describes the type class.
> 
> In TypeScript, there are no "pure" type classes, so I will describe them using interfaces. I will use the term "instance of (class) X for (container) Y" to emphasize that the interface X should not be firmly bound to container Y, but can be defined independently. This, for example, gives you the opportunity to import into your program only the type classes you need for your container Y at a given time.

## Functor

You can think of the functor for Maybe as an interface that implements only one method — `fmap`:

```ts
interface Functor<A> {
  fmap: <B>(f: (a: A) => B) => (ma: Maybe<A>) => Maybe<B>;
}
```

The `fmap` method can be viewed from two sides:
1. As a way to “apply” a pure function to a “containerized” value;
2. As a way to “raise to container context” a pure function.

Indeed, if we place brackets in the interface a little differently, we can get such a signature of the  `fmap` function:

```ts
const fmap: <A, B>(f: (a: A) => B) => ((ma: Maybe<A>) => Maybe<B>);
```

You can use the `Function1` interface from `fp-ts`, and get the definition of `fmap`:

```ts
const fmap: <A, B>(f: (a: A) => B) => Function1<Maybe<A>, Maybe<B>>;
```

The functor must obey two algebraic laws:

1. The law of conservation of function composition: `fmap f ∘ fmap g ≅ fmap (f ∘ g)`
2. Identity preservation law: `fmap id ≅ id`

Your task is to define an instance of the functor for the Maybe container by writing the `fmap` function.

## Applicative Functor (Applicative)

An applicative functor allows you to apply a function raised in a context to a value in that context. We can think of the applicative functor for Maybe as an interface defining a pair of operations:

```ts
interface Applicative<A> {
  of: (a: A) => Maybe<A>;
  ap: <B>(fab: Maybe<(a: A) => B>) => (ma: Maybe<A>) => Maybe<B>;
}
```

The applicative functor obeys 4 laws:

1. Identity: `ap (of id) ma ≅ ma`
2. Homomorphism: `ap (of ab) (of a) ≅ of (ab a)`
3. Interchange: `ap fab (of a) ≅ ap (\ab -> ab a) fab`
4. Composition: `ap (ap (ap (of compose) u) v) w ≅ ap u (ap v w)`

Your task is to define an instance of the applicative functor for Maybe by writing the functions `of` and `ap`.

## Alternative

The alternative type allows you to define a pair of operations: An associative alt operation, which “chooses” between its two arguments, and a zero operation, which serves as a neutral (zero) element for the alt operation both on the left and on the right. An alternative can be implemented for an applicative functor - due to the requirement of alt associativity and the presence of a neutral element.

You can think of an alternative as the binary operation `||` in JS, but only for containers:

```js
const iamnull = null;
const iamnotnull = 'hello';
const fallback = 42;

const result1 = iamnull || fallback; // => 42;
const result2 = iamnotnull || fallback; // => 'hello';
```

An alternative must obey these laws:

1. Left identity: `alt zero ma ≅ ma`
2. Right identity: `alt ma zero ≅ ma`
3. Associativity: `alt (alt ma mb) mc ≅ alt ma (alt mb mc)`
4. Distributivity: `ap (alt fab gab) ma ≅ alt (ap fab ma) (ap gab ma)`
5. Annihilation: `fmap zero f ≅ zero`, `ap fa zero ≅ zero`

Your task is to define an alternative instance for Maybe by writing the functions `zero` and `alt`.

## Monad

The monad combines the applicative functor interface and the `Chain` interface, a sequential chain of calculations that uses the result of a previous function to determine the sequence of the following functions.

You can think of the monad for Maybe as an interface that implements the functions `of` and `chain`:

```ts
interface Monad<A> {
  of: (a: A) => Maybe<A>;
  chain: <B>(f: (a: A) => Maybe<B>) => (ma: Maybe(A)) => Maybe<B>;
}
```

A monad must obey three laws:

1. Left identity: `chain f (of a) ≅ f a`
2. Right identity: `chain of fa ≅ fa`
3. Associativity: `chain (chain afb fa) bfc ≅ chain (\a -> chain bfc (afb a)) fa`

plus all laws for an applicative functor.

> The important bit: A monad can be defined not only by a pair of `of` and `chain` , but also by a pair `of` and `flatten`!

The `flatten` function “collapses” one level of the container:

```ts
const flatten: <A>(mma: Maybe<Maybe<A>>) => Maybe<A>;
```

The functions `chain` and `flatten` can be expressed in terms of each other:

```ts
const chain = f => ma => flatten(fmap(f)(ma));
const flatten = mma => chain(fmap(identity))(mma);
```
Therefore, in some functional operations, the `chain` operation can still be called `flatMap`, which reflects its essence more accurately: It “raises” (map) computations of the form `A => F<B>` to the context `F`, receives an intermediate result of the form `F<F<B>>`, and then “flattens” one level, returning just `F<B>`.

> By the way, in the ECMAScript 2015 standard, the arrays have a `flatMap` operation, which is exactly the monadic `chain` operation.

Your task is to define an instance of the monad for Maybe by writing the functions `chain` and `flatten`.

# Conclusion

Hopefully, after finishing this kata you will be able to draw the following conclusions:

1. The concept of “container” is not equivalent to the concepts of “monad” / “functor” / “applicative” / etc., since we define instances of the type classes functor / monad / etc. for the container, *not modifying* the container itself.
2. In your programs, you can use just a part of the interface of a container that has instances for several type classes. This allows you to use the principle of least knowledge and limit the types of function arguments in the code to only those features that you need at a particular point in time. For example, when using fp-ts, you can import only the instances of the type classes you need:

    ```ts
    // Suppose you only use `of` in `foo`.
    // Then, instead of:
    const foo: (M: Monad<F>) => <A, B>(fa: HKT<F, A>) => HKT<F, B>;
    // you can write:
    const foo: (M: Applicative<F>) => <A, B>(fa: HKT<F, A>) => HKT<F, B>;
    ```

    About the notation of `HKT<F, A>` we will talk later when we get to the definition of higher kinded types.
3. Due to the fact that the type classes of a functor, monad, applicative, alternative, etc. obey algebraic laws, they can be expressed in terms of each other and reuse parts of their interfaces.
