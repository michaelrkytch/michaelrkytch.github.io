---
layout: post
title: Implementing a Java library in Clojure
subtitle: Entry point pitfalls
categories: [Programming, Clojure, Interop]
---

One of the arguments for using Clojure on the server side is its deep interoperability with Java.  Using Java libraries from a Clojure program generally works seamlessly, although there are pitfalls due to the "impedance mismatch" between OO and functional languages.  But what if you want to write a library in Clojure for use in a Java server?  There are some useful, though confusing, features for fitting your code into existing interface and class hierarchies.  See Chas Emerick's excellent [flowchart](http://cemerick.com/2011/07/05/flowchart-for-choosing-the-right-clojure-type-definition-form/) for choosing the right Clojure type definition form.

In particular, `defrecord` and `deftype` give you very convenient ways to define classes with Clojure implementations.  Unfortunately, using these classes from a Java program can be a bit awkward.  A Java programmer expects a library to expose classes that can either be directly instantiated or which are provided by factory methods in the API.

## An example
Suppose you want to implement some existing Java interface in your library.

```java
package interop.example;

public interface ICalculator {
    public int multiply(int x);
}
```

And suppose your implementation needs some configuration before it is used.  In our contrived example, our `Calculator` takes a base `operand` as a constructor argument.  The `mult` method then multiplies its argument by the base `operand`.

An intuitive way to create an entry point for your Clojure library would be to use `defrecord` to define a class implementing that interface.

```clojure
(ns interop-blog.core
  (:import [interop.example ICalculator]))

(defn mult [x y] (* x y))

(defrecord Calculator [operand]
  ICalculator
  (multiply [this x] (mult x operand)))
```

If we ahead-of-time compile this namespace, a `Calculator` class will be generated which we can use in our Java program.  Here, for example we use Leiningen's `:aot` tag to accomplish this.

```clojure
(defproject interop-blog "0.1.0-SNAPSHOT"
  :description "Example code for blog on Java interop pitfalls"
  :dependencies [[org.clojure/clojure "1.6.0"]]
  :java-source-paths ["src-java"]
  :aot [interop-blog.core])
```

And we can see the resulting .class file:

```
$ lein compile
$ find . -name Calculator.class
./target/classes/interop_blog/core/Calculator.class
```

## Invoking a record's constructor

So we can now instantiate our new class and use it from a Java program:

```java
package interop.example;
import interop_blog.core.Calculator;

public class TestMain {
    public static void main(String[] args) {
        int input = Integer.parseInt(args[0]);
        ICalculator calc = new Calculator(100);
        System.out.println(calc.multiply(input));
    }
}
```

Let's try it out.

```
$ javac -classpath target/classes -sourcepath src-main -d target/classes src-main/interop/example/TestMain.java
$ java -classpath /Users/michael/.m2/repository/org/clojure/clojure/1.6.0/clojure-1.6.0.jar:target/classes interop.example.TestMain 100
Exception in thread "main" java.lang.IllegalStateException: Attempting to call unbound fn: #'interop-blog.calc/mult
	at clojure.lang.Var$Unbound.throwArity(Var.java:43)
	at clojure.lang.AFn.invoke(AFn.java:36)
	at interop_blog.core.Calculator.multiply(core.clj:7)
	at interop.example.TestMain.main(TestMain.java:8)
```

What went wrong?  It turns out that the generated `Calculator` class does not automatically import and compile the Clojure code that it depends on.  In this case, we are calling a Clojure method `mult` from `Calculator.multiply()`, and this code is not loaded in the runtime.  It sure would be nice if this were taken care of automatically by the Clojure AOT compiler, but no such luck.

So to load the required Clojure code, we need to invoke some magic incantations before attempting to use our new class.  Here we use the [clojure.java.api](http://clojure.github.io/clojure/javadoc/clojure/java/api/package-summary.html) package to `require` our `interop-blog.core` namespace.

```java
package interop.example;
import interop_blog.core.Calculator;
import clojure.java.api.Clojure;
import clojure.lang.IFn;

public class TestMain2 {
    public static void main(String[] args) {
    	IFn require = Clojure.var("clojure.core","require");

        // NB: our namespace interop-blog contains a hyphen.  This is translated to 
        // an underscore in the file path.  It can be tricky to remember when to use a 
        // hyphen and when to use an underscore, so maybe a better practice is to 
        // avoid hypens in namespaces
    	require.invoke(Clojure.read("interop_blog.core"));

        int input = Integer.parseInt(args[0]);
        ICalculator calc = new Calculator(100);
        System.out.println(calc.multiply(input));
    }
}
```

And now it works...

```
$ javac -classpath /Users/michael/.m2/repository/org/clojure/clojure/1.6.0/clojure-1.6.0.jar:target/classes -sourcepath src-main -d target/classes src-main/interop/example/TestMain2.java
$ java -classpath /Users/michael/.m2/repository/org/clojure/clojure/1.6.0/clojure-1.6.0.jar:target/classes interop.example.TestMain2 100
10000
```

## A cleaner design

This is not much code, but it's not nice for a library to leak its implementation in a way which requires the consumer to use it in this awkward and non-idiomatic way.  The best pattern I have found to hide these implementation details is to encapsulate the magic incantations in a static factory method.

```java
package interop.example;

import clojure.java.api.Clojure;
import clojure.lang.IFn;

public class CalculatorFactory {
    public static ICalculator newCalculator(int operand) {
    	IFn require = Clojure.var("clojure.core","require");
    	require.invoke(Clojure.read("interop_blog.core"));
    	IFn newCalcFn = Clojure.var("interop-blog.core", "->Calculator");
        return (ICalculator) newCalcFn.invoke(Integer.valueOf(operand));
    }
}
```

Note that in the code above, we are using the automatically generated factory function `->Calculator` rather than calling the record's constructor directly.

Now our Java client code is back to being nice and simple:

```java
package interop.example;

public class TestMain3 {
    public static void main(String[] args) {
        int input = Integer.parseInt(args[0]);
        ICalculator calc = CalculatorFactory.newCalculator(100);
        System.out.println(calc.multiply(input));
    }
}
```

...and it works...

```
$ lein compile
$ javac -classpath /Users/michael/.m2/repository/org/clojure/clojure/1.6.0/clojure-1.6.0.jar:target/classes -sourcepath src-main -d target/classes src-main/interop/example/TestMain3.java
$ java -classpath /Users/michael/.m2/repository/org/clojure/clojure/1.6.0/clojure-1.6.0.jar:target/classes interop.example.TestMain3 100
10000
```


## Conculusion

In my experience, many Java developers are skeptical of Clojure and other JVM languages.  An important part of developing interoperable libraries in Clojure is making the experience of your Java consumers as smooth as possible.  In addition to the usual documentation, testing, clean error messages, etc, you also need to pay attention to packaging, dependencies, and idiomatic APIs.  Ideally your consumers won't know that they are using a library implemented in Clojure -- although they are likely to figure it out if they get an exception stack trace or have to step over library code in a debugger.  Dynamic compilation and dynamic typing make the fit awkward enough.  Don't give your Java clients anything else to complain about!
