# Codecs

**WARNING**: This tutorial expects a strong understanding of Java generics.

The `Codec` class from [DataFixerUpper](https://github.com/Mojang/DataFixerUpper) is the backbone of content serialization and deserialization in Minecraft.
It provides an abstraction layer between Java Objects and serialization types, such as [`json`](https://minecraft.wiki/w/JSON), [`nbt`](https://minecraft.wiki/w/NBT_format), and more.
Internally, each `Codec` uses an `Encoder` and a `Decoder` to write and read the data respectively.
Mojang provides us with utilities for creating codecs, which means we won't have to worry about making our own encoders and decoders.

## Primitive Codecs

There are default `Codec` implementations for most of the [primitive types](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html), which greatly simplifies creating `Codec`s.
Combining these types will allow us to serialize any object, no matter how complex. Before we do so, we need to get familiar with how to use one of these primitive codecs

The most important codecs are those for the primitive types:
- `Codec.BOOL`
- `Codec.BYTE`
- `Codec.SHORT`
- `Codec.INT`
- `Codec.LONG`
- `Codec.FLOAT`
- `Codec.DOUBLE`
- `Codec.STRING`

The keen eyed may have noticed that `String` is standing in for `char`. Since a string can be used to represent a single character, there's no need to have a separate codec for `char`.
Now that we know what tools are available to us, let's find out how to use them!

## Basic Codec Example
To start, here's the simplest possible usage of a `Codec` to decode data:

```java
boolean bool = 
        Codec.BOOL.decode(
            JsonOps.INSTANCE,
            new JsonPrimitive(true)
        )
        .result()
        .get()
        .getFirst();
```

For a simple example, that doesn't look simple *at all*!
Using `Codec`s is fairly verbose, but that means you get a lot of useful information to help with errors and such, which is important for Mojang to provide since we want to know why Minecraft failed to load something, not just that it failed.
Now, let's break down how this works.

First off:
```java
[...]
Codec.BOOL.decode(
    JsonOps.INSTANCE, // DynamicOps<T> ops
    new JsonPrimitive(true) // T input
) // returns a DataResult<Pair<Boolean, JsonElement>>
[...]
```
The `decode` method on a codec takes two values, a `DynamicOps<?>` instance and an input.
As shown in the comments above, the type of the input and a generic parameter on `ops` must match.
In this example, we use `com.mojang.serialization.JsonOps.INSTANCE`, which operates on JSON elements using [GSON](https://github.com/google/gson).
We then pass in a `JsonPrimitive` with a value of `true` for this example.

Finally, the `com.mojang.serialization.DataResult<Pair<A, T>>` type allows us to encode more information than just the result.
First off, the `A` type is the output of the `Codec`, which is the parsed `Boolean` with a value of `true`, and the `T` is the same as the input: our JSON data with a value of `true`.

Let's look more into the `DataResult`:
```java
...
.result() // Optional<Pair<Boolean, JsonElement>>
...
```

`DataResult` has a lot of associated methods, but the two most important are `result` and `error`.
`error` returns a `PartialResult`, which allows you to both recover a decode attempt, and to get the error message for why decoding failed.
`result` returns an `Optional<A, T>`, which provides the parsed data along with the input if decoding was successful.

Now that we have the result, we get to the last two lines:
```java
...
.get() // Pair<Boolean, JsonElement>
.getFirst(); // Boolean
```

We use `get` to unbox the `Optional`. When using a `Codec` on data you're not sure will successfully parse, this is unsafe to do.
In this case, we don't have to verify that the `Optional` contains data, since we directly sent in a boolean value.
If we were sourcing this boolean from a user-facing config file, for example, we would need to verify the value's presence before getting it.
Then finally, we call `getFirst` on `com.mojang.datafixers.util.Pair` to get the first half of the pair: our parsed boolean.

When working with Minecraft, most of the time you only need to provide the `Codec`, and Minecraft will do the (de)serialization for you.
Now that we know how to use a `Codec` and understand the classes associated with it, let's move on to building complex `Codec`s.

## Collection Codecs // TODO

While the primitive `Codec`s are the most basic building blocks for `Codec`s, we need to we able to put them together to be able to fully represent serializable objects.
These collection `Codec`s are fairly straight forward, and each has a constructor which takes a `Codec` parameter for each associated type with the collection.

<!-- TODO: Use the static methods instead of the classes -->

### `ListCodec<T>`
A codec for a `List<T>`. 
You can make a `ListCodec<T>` by calling
- `listOf()` on an instance of `Codec<T>`.
- `Codec.list(Codec<T>)` with the codec for the element type.

There are also methods that allow you to set a minimum and maximum size for the list.

### `SimpleMapCodec<K, V>`
A codec for a `Map<K, V>` with a known set of keys of type `K`. This known set is an additional parameter. Because of this, we usually recommend using `UnboundedMapCodec<K, V>`

You create a `SimpleMapCodec<K, V>` by calling `Codec.simpleMap(Codec<K>, Codec<V>, Keyable)`.

### `UnboundedMapCodec<K, V>`
A codec for a `Map<K, V>`.

You create a `UnboundedMapCodec<K, V>` by calling `Codec.unboundedMap(Codec<K>, Codec<V>)`.

### `PairCodec<F, S>`
A codec of a `Pair<F, S>`. This is fairly rare as it isn't too often you have just two values you want to serialize together without names for the fields. 

You create a `PairCodec<F, S>` by calling `Codec.pair(Codec<F>, Codec<S>)`.

### `EitherCodec<L, R>`
A codec of an `Either<L, R>`. This is part of the strength of `Codec`s. This allows you to represent one value with multiple different serializers, and it will choose the correct one based on the type. For example, if you want something to serialize to either a `String` or and `Integer`, you would use an `EitherCodec<String, Integer>`. We will cover this more in depth later on, as there are still a few more concepts to go over before the full use of `EitherCodec<L, R>` becomes apparant.

You create a `EitherCodec<L, R>` by calling `Codec.either(Codec<L>, Codec<R>)`.


## The `RecordCodecBuilder`
Oh no. There a full type name from DFU in a header. This is going to get crazy.

Let's start it off simple: a `RecordCodecBuilder` creates a `Codec` that can directly serialize and deserialize a Java object.
While it has the name `Record` in it, this isnt specific to the `record`s in Java, but it's often a good idea to use `record`s.
Going over a basic example will probably be the clearest here.

```java
record Foo(int bar, List<Boolean> baz, String qux) {
    public static final Codec<Foo> CODEC =
        RecordCodecBuilder.create(
            instance ->
                instance.group(
                    Codec.INT.fieldOf("bar").forGetter(Foo::bar),
                    Codec.BOOL.listOf().fieldOf("baz").forGetter(Foo::baz),
                    Codec.STRING.optionalFieldOf("qux", "default").forGetter(Foo::qux)
                ).apply(instance, Foo::new)
        ).codec();
}
```

Ok, thats not too bad. 
One nice thing about this is that Mojang did a lot of magic behind the scene to make this feel nice.
Trust us, there have been a few Quilt developers who have tried making a similar library (OroArmor) and gave up on doing the right thing.

Now, `RecordBuilder.create` takes a lambda, providing an `instance` parameter.
The main bulk of this lambda is the `group` method.
You can pass up to 16 different `Codec`s turned into fields through this method.

Turning a `Codec` into a field follows a fairly simple pattern.

First, you start with the `Codec` (`Codec.INT`, `Codec.BOOL.listOf()`, and `Codec.STRING`).

Then, you can call one of two methods:
 - `fieldOf`, which takes a string parameter for the serialized field name.
 - `optionalFieldOf`, also takes the same name parameter. 
    By default this represents an `Optional<T>`, with `T` being the `Codec` type. 
    There is a method overload, like used in the example, which allows you to provide a default value and not have to store an `Optional<T>`.

Next, you call `forGetter`, which takes a `Function<O, T>`, with `O` being the object you are trying to serialize, and `T` being the type of the field on the object.

Once you have finished calling group, the next thing to do is chain an `apply` call. The first parameter is always `instance`, and the second parameter is usually the method handle to constructor for the object you are making the `Codec` for. One thing that is important is to make sure that the order of the constructor parameters and the order of the fields in the group match, as this could cause either runtime or compile time errors.

Finally, you call `.codec()` on the returned value, since the returned value would be a `MapCodec` otherwise. Even though it sounds like it, a `MapCodec` isn't like a Java `Map`. While we won't cover `MapCodec`s much, they do have their uses which will be explained later.

### Serialization
Now, let's see a serialized `new Foo(8, List.of(true, false, true), "string")` in json:
```json
{
    "bar": 8,
    "baz": [true, false, true],
    "qux": "string"
}
```

Now, since we had an optional field, let's see what this json looks like:
```json
{
    "bar": -2,
    "baz": []
}
```
Once deserialized, we get an object equal to `new Foo(-2, List.of(), "default")`

## `Codec.dispatch`

Dispatched `Codec`s are probably both the most complex feature of `Codec`, but also the most elegant and powerful. What if I told you that something you already deserialized could change the rest of the deserialization? An example will probably be the best way to start off:

```java
interface Dispatched {
    String type();

    Map<String, MapCodec<? extends Dispatched>> TYPES = Map.of(
        "a", A.CODEC,
        "b", B.CODEC
    );

    Codec<Dispatched> CODEC = Codec.STRING.dispatch(
        Dispatched::type,
        TYPES::get
    );
}

record A(int a) implements Dispatched {
    public static MapCodec<A> CODEC = 
        RecordCodecBuilder.create(
            instance ->
                instance.group(
                    Codec.INT.fieldOf("a").forGetter(A::a)
                ).apply(instance, A::new)
        );

    public String type() { return "a"; }
}

record B(String b) implements Dispatched {
    public static MapCodec<B> CODEC = 
        RecordCodecBuilder.create(
            instance ->
                instance.group(
                    Codec.STRING.fieldOf("b").forGetter(B::b)
                ).apply(instance, B::new)
        );

    public String type() { return "b"; }
}
```

Alright, thats a *lot* of code, but most of it is fairly straight forward. First we have an interface that defines one method, `String type()`. This is so that we can know the types of any implementing classes. We then have a `TYPES` map which is a map of the type of object to its `Codec`. Next, we have the dispatch `Codec`. Since our type's type is `String`, we start off with `Codec.STRING`, since this is how to serialize/deserialize the type. Then we call `dispatch`. The first parameter is a `Function` that takes in the object (a `Dispatched`), and returns the type (a `String`), and here we use the method reference. The second parameter is another `Function`, but takes in a `String` and returns a `MapCodec` (explained shortly). Here we use a method reference to `TYPES.get(s)` to keep the code cleaner.

Finally, we have two different records implementing `Dispatched` with their own `MapCodec`s. Now, `MapCodec`s can be thought of like `Map<String, Codec>`, where the keys are field names. While it's certainly more flexible than that, this covers 90+% of use cases. The reason we use a `MapCodec` is because a `MapCodec` can be inlined into a larger object. I can't put a `Codec.BOOL` into an object without some form of name for it. 

Now, lets look at some serializations:
| Java | JSON |
|:---:|:---:|
| `new A(10)` | `{"type": "a", "a": 10}` |
| `new B("str")` | `{"type": "b", "b": "str"}` |
