---
layout: post
title: Implementing a Java library in Clojure -- Constructing parameterized objects with reify
categories: [Programming, Clojure, Interop]
---
In my [previous post]({% post_url 2015-05-26-clj-interop-require.md %}), I looked at the problem of how to implement a Java API in Clojure when the API requires construction of parameterized objects.  In that post, I used a Clojure record to implement the API interface and wrote a static factory method in Java to encapsulate the magic incantations required to instantiate the object and load the code it depends on.  In this post, I will show a simpler pattern using `reify` instead of `defrecord` to implement an interface.

## Reifying an interface

In our previous example, we had to implement this Java interface

```java
package interop.example;

public interface ICalculator {
    public int multiply(int x);
}
```

which we did by defining a record


```clojure
(ns interop-blog.core
  (:import [interop.example ICalculator]))

(defn mult [x y] (* x y))

(defrecord Calculator [operand]
  ICalculator
  (multiply [this x] (mult x operand)))
```

another way to accomplish this is using `reify` in a factory function.

```clojure
(defn new-reified-calc [operand]
  (reify
    ICalculator
    (multiply [this x] (mult x operand))))
```

`reify` creates an anonymous class implementing one or more interfaces or protocols.  The method bodies in `reify` form lexical closures over their local environment.  That means, in this case, that we can capture the argument of our `new-reified-calc` function and use it in the methods of our reified class, just like a constructor argument (except of course that it is immutable).

## Instantiation from Java

To call our new factory function from Java, we still need the usual boilerplate to load the namespace in which it is defined and to invoke the factory function.  As in the previous post, a static factory method in Java is a pretty good way to encapsulate this boilerplate.  This looks almost exactly like the factory function for instantiating the record in our previous post.

```java
package interop.example;

import clojure.java.api.Clojure;
import clojure.lang.IFn;

public class CalculatorFactory2 {
    public static ICalculator newCalculator(int operand) {
    	IFn require = Clojure.var("clojure.core","require");
    	require.invoke(Clojure.read("interop_blog.core"));
    	IFn newCalcFn = Clojure.var("interop-blog.core", "new-reified-calc");
        return (ICalculator) newCalcFn.invoke(Integer.valueOf(operand));
    }
}
```


## defrecord vs reify

`defrecord` allows us to define a class in Clojure and to construct parameterized instances of this class.  `reify` can also be used for this purpose.  So what's the difference?   The major differences are that records are named classes, and records are Clojure maps.  Being able to refer to the class by name directly in Java is not that useful when the class is being provided by an API, because we want our consumers to program to interfaces anyway.  The fact that a record is a Clojure map isn't particularly useful either when we are using it idiomatically in java.

Referring again to Chas Emerick's [Clojure type selection flowchart](https://github.com/cemerick/clojure-type-selection-flowchart), a record is useful if you are modeling a domain value.  If some of the methods of your type T will produce new instances of type T, this is probably most naturally expressed with `defrecord`. To use the classic `Point` example, we might have a operation on a Euclidean point that translates it to another point in the same space.

```clojure
(ns interop-blog.point)

(defprotocol TranslatePoint
  (moveX [this delta] "Translate a point along the X axis."))

(defrecord Point [x y]
  TranslatePoint
  (moveX [this delta] (Point. (+ x delta) y)))
```

In this case the `moveX` method instantiates a new `Point` based on its own immutable state.

## Conclusion

When designing a Java API, it is idiomatic to provide callable entry points through parameterized objects instantiated by a factory.  Using `reify` inside a factory function provides us a concise way to bind a Java interface to a set of Clojure functions through the construction of an anonymous, parameterized object.
