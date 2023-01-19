# Chapter 8: Methods
## Item 49: Check parameters for validity
For public and protected methods, use the Javadoc @throws tag to document the exception that will be thrown if a restriction on parameter values is violated.

Example:
```java
/**
* Returns a BigInteger whose value is (this mod m). This method
* differs from the remainder method in that it always returns a
* non-negative BigInteger.
*
* @param m the modulus, which must be positive
* @return this mod m
* @throws ArithmeticException if m is less than or equal to 0
*/
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0)
        throw new ArithmeticException("Modulus <= 0: " + m);

    // ...
}
```
The ```Objects.requireNonNull``` method is flexible and convenient so there's no reason to perform null checks manually anymore.

Example:
```java
// Inline use of Java's null-checking facility
this.strategy = Objects.requireNonNull(strategy, "strategy");
```

Nonpublic methods can check their parameters using ```assertion```.
```java
// Private helper function for a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    // Do the computation
}
```

It is particularly important to check the validity of parameters that are not used by a method, but stored for later use (especially constructors).

## Item 50: Make defensive copies when needed
Attacks on mutable objects (i.e. ```Date```)
```java
package effectivejava.chapter8.item50;
import java.util.*;

// Two attacks on the internals of an "immutable" period (232-3)
public class Attacks {
    public static void main(String[] args) {
        // Attack the internals of a Period instance  (Page 232)
        Date start = new Date();
        Date end = new Date();
        Period p = new Period(start, end);
        end.setYear(78);  // Modifies internals of p!
        System.out.println(p);

        // Second attack on the internals of a Period instance  (Page 233)
        start = new Date();
        end = new Date();
        p = new Period(start, end);
        p.end().setYear(78);  // Modifies internals of p!
        System.out.println(p);
    }
}
```
Defensive copy of ```Period```
```java
package effectivejava.chapter8.item50;
import java.util.*;

// Broken "immutable" time period class (Pages 231-3)
public final class Period {
    private final Date start;
    private final Date end;

    /**
     * @param  start the beginning of the period
     * @param  end the end of the period; must not precede start
     * @throws IllegalArgumentException if start is after end
     * @throws NullPointerException if start or end is null
     */
    public Period(Date start, Date end) {
        if (start.compareTo(end) > 0)
            throw new IllegalArgumentException(
                    start + " after " + end);
        this.start = start;
        this.end   = end;
    }

    public Date start() {
        return start;
    }
    public Date end() {
        return end;
    }

    public String toString() {
        return start + " - " + end;
    }

//    // Repaired constructor - makes defensive copies of parameters (Page 232)
//    public Period(Date start, Date end) {
//        this.start = new Date(start.getTime());
//        this.end   = new Date(end.getTime());
//
//        if (this.start.compareTo(this.end) > 0)
//            throw new IllegalArgumentException(
//                    this.start + " after " + this.end);
//    }
//
//    // Repaired accessors - make defensive copies of internal fields (Page 233)
//    public Date start() {
//        return new Date(start.getTime());
//    }
//
//    public Date end() {
//        return new Date(end.getTime());
//    }

    // Remainder omitted
}
```

We can also use ```Instant``` (or ```LocalDateTime``` or ```ZonedDateTime```) in place of a ```Date``` because ```Instant``` are immutable.

## Item 51: Design method signatures carefully
Three techniques for shortening overly long parameter lists:
1. Break the method into multiple methods, each with a shorter parameter list.
2. Create helper classes to hold groups of parameters.
3. Adapt the Builder pattern from object construction to method invocation. If you have a method with many parameters, especially if some of them are optional, it can be beneficial to define an object that represents all of the parameters and to allow the client to make multiple "setter" calls on this object, each of which sets a single parameter or a small, related group

## Item 52: Use overloading judiciously
The choice of which overloading to invoke is made at compile time based on the static type of the arguments while selection among overridden methods is dynamic and made at runtime.

```java
package effectivejava.chapter8.item52;
import java.util.*;
import java.math.*;

// Broken! - What does this program print?  (Page 238)
public class CollectionClassifier {
    public static String classify(Set<?> s) {
        return "Set";
    }

    public static String classify(List<?> lst) {
        return "List";
    }

    public static String classify(Collection<?> c) {
        return "Unknown Collection";
    }

    public static void main(String[] args) {
        Collection<?>[] collections = {
                new HashSet<String>(),
                new ArrayList<BigInteger>(),
                new HashMap<String, String>().values()
        };

        for (Collection<?> c : collections)
            System.out.println(classify(c)); // Prints "Unknown Collection" three times
    }
}
```
Correct version of ```CollectionClassifier```
```java
package effectivejava.chapter8.item52;

import java.math.BigInteger;
import java.util.*;

// Repaired  static classifier method. (Page 240)
public class FixedCollectionClassifier {
    public static String classify(Collection<?> c) {
        return c instanceof Set  ? "Set" :
                c instanceof List ? "List" : "Unknown Collection";
    }

    public static void main(String[] args) {
        Collection<?>[] collections = {
                new HashSet<String>(),
                new ArrayList<BigInteger>(),
                new HashMap<String, String>().values()
        };

        for (Collection<?> c : collections)
            System.out.println(classify(c));
    }
}
```
Two types are radically different if it is clearly impossible to cast any non-null expression to both types. For example, ```ArrayList``` has one constructor that takes an ```int``` and a second constructor that takes a ```Collection```. It is hard to imagine any confusion over which of these two constructors will be invoked under any circumstances.

But autoboxing can make it hard among primitive types.

```java
package effectivejava.chapter8.item52;
import java.util.*;

// What does this program print? (Page 241)
public class SetList {
    public static void main(String[] args) {
        Set<Integer> set = new TreeSet<>();
        List<Integer> list = new ArrayList<>();

        for (int i = -3; i < 3; i++) {
            set.add(i);
            list.add(i);
        }
        for (int i = 0; i < 3; i++) {
            set.remove(i);
            list.remove(i); // should change to list.remove((Integer)i) or list.remove(Integer.valueOf(i))
        }
        System.out.println(set + " " + list);
        // Prints [-3, -2, -1] [-2, 0, 2]
    }
}
```
The ```List``` interface has two overloadings of tyhe ```remove``` method: ```remove(Object)``` and ```remove(int)```. ```remove(int)``` removes the element at the specified position or index in the list.

It is generally best to refrain from overloading methods with multiple signatures that have the same number of parameters. In some cases, especially where constructors are involved, it may be impossible to follow this advice. In these cases, you should at least avoid situations where the same set of parameters can be passed to different overloadings by the addition of casts. If this cannot be avoided, for example, because you are retrofitting an existing class to implement a new interface, you should ensure that all overloadings behave identically when passed the same parameters.

## Item 53: Use varargs judiciously
The varargs facility works by first creating an array whose size is the number of arguments passed at the call site, then putting the argument values into the array, and finally passing the array to the method.

To write a method that requiress one or more arguments of some type, rather than zero or more:

```java
// The right way to use varargs to pass one or more arguments (Page 246)
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
            min = arg;
    return min;
}
```

When using varargs in performance-critical situations: Suppose you’ve determined that 95 percent of the calls to a method have three or fewer parameters. Then declare five overloadings of the method, one each with zero through three ordinary parameters, and a single varargs method for use when the number of arguments exceeds three:

```java
public void foo() { ... }
public void foo(int a1) { ... }
public void foo(int a1, int a2) { ... }
public void foo(int a1, int a2, int a3) { ... }
public void foo(int a1, int a2, int a3, int... rest) { ... }
```
Remember to precede the varargs parameter with any required parameters.

## Item 54: Return empty arrays or collections, not nulls
```java

private final List<Cheese> cheesesInStock = ...;
/**
 * @return an array containing all of the cheeses in the shop,
 * or an empty array if no cheeses are available for purchase.
 */
 // The right way to return a possibly empty array (Page 247)
public List<Cheese> getCheeses() {
    return new ArrayList<>(cheesesInStock);
}
```

To improve performance, you can consider to return the same immutable empty collection repeatedly as immutable objects may be shared freely.

```java
// Optimization - avoids allocating empty collections (Page 248)
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? Collections.emptyList()
            : new ArrayList<>(cheesesInStock);
}
```

```java
// The right way to return a possibly empty array
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[0]);
}
```

```java
// Optimization - avoids allocating empty arrays
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];

public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

```java
// Don't do this - preallocating the array harms performance
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[cheesesInStock.size()]);
}
```

## Item 55: Return optionals judiciously
The ```Optional<T>``` class represents an immutable container that can hold either a single non-null ```T``` reference or nothing at all.

```java
// Returns maximum value in collection as an Optional<E> (Page 250)
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();

    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);

    return Optional.of(result);
}
```
Never return a null value from an ```Optional```-returning method. 

If a method returns an optional, the client gets to choose what action to take if the method can't return a value.

```java
// Using an Optional to provide chosen behavior for a method (Page 251)
String lastWordInLexicon = max(words).orElse("No words...");
```

```java
// Using an optional to throw a chosen exception
Toy myToy = max(toys).orElseThrow(Toy::new);
```

Container types, including collections, maps, streams, arrays, and optionals should not be wrapped in optionals. Rather than returning an empty ```Optional<List<T>>```, return an empty ```List<T>```.

You should never return an optional of a boxed primitive type (you have ```OptionalInt```, ```OptionalLong```, and ```OptionalDouble``` for that purpose).

For performance-critical methods, it may be better to return a ```null``` or throw an exception.

## Item 56: Write doc comments for all exposed API elements
Read *How to Write Doc Comments* web page [Javadoc-guide](https://www.oracle.com/technical-resources/articles/java/javadoc-tool.html).