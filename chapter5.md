# Chapter 5: Generics

## Item 26: Don't use raw types
With generics, you tell the compiler what types of objects are permitted in each collection. The compiler inserts casts for you automatically and tells you at compile time if you try to insert an object of the wrong type.

If you use raw types, you lose all the safety and expressiveness benefits of generics.

```java
// Fails at runtime - unsafeAdd method uses a raw type (List)!
public static void main (String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0); // Throws ClassCastException
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
```

The program compiles, but because it uses the raw type ```List```, you get a warning:
```bash
Test.java:10 warning: [unchecked] unchecked call to add(E) as a member of the raw type List
    list.add(o);
```

```List<Object>``` allow insertion of arbitrary objects, but ```List<String>``` is not a subtype of ```List<Object>```. Thus, if we change raw type ```List``` to ```List<Object>```, the program will not compile.

For unbounded wildcard types, you can't put any element (other than null) into a ```Collection<?>```. Attempting to do so will generate a compile-time error. It is better than raw types beacause raw type may easily corrupt the collection's type invariant.

```java
// Use unbounded wildcard type - typesafe and flexible
static int numElementsInCommon(Set<?> s1, Set<?> s2) {
    int result = 0;
    for (Object o1 : s1)
        if (s2.contains(o1))
            result++;
    return result;
}
```

There are two exceptions to use raw types:
- Class literals (List.class, String[].class, int.class)
- ```instanceof``` operator: Because generic type information is erased at runtime, it is illegal to use the ```instanceof``` operator on parameterized types other than unbouned wildcard types.


## Item 27: Eliminate unchecked warnings
Eliminate every unchecked warning that you can. If you can't, but you can prove that the code that provoke the warning is typesafe, then suppress the warning with a ```@SuppressWarnings("unchecked")``` annotation.

Always use the ```@SuppressWarnings("unchecked")``` annotation on the smallest scope possible. 

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
If you compile this code, you get a warning:
```bash
Test.java:3 warning: [unchecked] unchecked cast
    return (T[]) Arrays.copyOf(elements, size, a.getClass());
```

It is illegal to put a ```SuppressWarnings``` annotation on the return statement because it isn't a declaration. But you can declare a local variable to hold the return value and annotate its declaration:
```java
// Adding local variable to reduce scope of @SuppressWarnings
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // The cast is correct because the array we're creating
        // is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```


## Item 28: Prefer lists to arrays
Arrays differ from generic types in two important ways. First, arrays are covariant. It means that if ```Sub``` is a subtype of ```Super```, then ```Sub[]``` is a subtype of ```Super[]```. Generics, by contrast, are invariant: for any two distinct types ```T1``` and ```T2```, ```List<T1>``` is neither a subtype nor a supertype of ```List<T2>```.

The code fragment is legal:
```java
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
```
but this one is not:

```java
// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in");
```

The second difference is that arrays are refied, which means arrays know and enforce their element type at runtime. Generics are implemented by erasure. This means that they enforce their type constaints only at compile time and discard their element type information at runtime.

Arrays and generics do not mix well. For example, it is illegal to create an array of a generic type, a parameterized type or a type parameter. (e.g. ```new List<E>[]```, ```new E[]``` are illegal)

```java
package effectivejava.chapter5.item28;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Random;
import java.util.concurrent.ThreadLocalRandom;

// List-based Chooser - typesafe (Page 129)
public class Chooser<T> {
    private final List<T> choiceList;

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }

    public static void main(String[] args) {
        List<Integer> intList = List.of(1, 2, 3, 4, 5, 6);

        Chooser<Integer> chooser = new Chooser<>(intList);

        for (int i = 0; i < 10; i++) {
            Number choice = chooser.choose();
            System.out.println(choice);
        }
    }
}
```

## Item 29: Favor generic types
First method
```java
package effectivejava.chapter5.item29.technqiue1;
import effectivejava.chapter5.item29.EmptyStackException;

import java.util.Arrays;

// Generic stack using E[] (Pages 130-3)
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    // The elements array will contain only E instances from push(E).
    // This is sufficient to ensure type safety, but the runtime
    // type of the array won't be E[]; it will always be Object[]! -> heap pollution
    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

    // Little program to exercise our generic Stack
    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        for (String arg : args)
            stack.push(arg);
        while (!stack.isEmpty())
            System.out.println(stack.pop().toUpperCase());
    }
}
```
Second method
```java
package effectivejava.chapter5.item29.technqiue2;

import java.util.Arrays;
import effectivejava.chapter5.item29.EmptyStackException;

// Generic stack using Object[] (Pages 130-3)
public class Stack<E> {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    // Appropriate suppression of unchecked warning
    public E pop() {
        if (size == 0)
            throw new EmptyStackException();

        // push requires elements to be of type E, so cast is correct
        @SuppressWarnings("unchecked") E result =
                (E) elements[--size];

        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

    // Little program to exercise our generic Stack
    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        for (String arg : args)
            stack.push(arg);
        while (!stack.isEmpty())
            System.out.println(stack.pop().toUpperCase());
    }
}
```
First method is preferred since it is more readable while the second requires a separate cast each time an array element is read.

## Item 30: Favor generic methods
The type parameter list, which declares the type parameters, goes between a method's modifiers and its return type.
```java
package effectivejava.chapter5.item30;
import java.util.*;

// Generic union method and program to exercise it  (Pages 135-6)
public class Union {

    // Generic method
    public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
        Set<E> result = new HashSet<>(s1);
        result.addAll(s2);
        return result;
    }

    // Simple program to exercise generic method
    public static void main(String[] args) {
        Set<String> guys = Set.of("Tom", "Dick", "Harry");
        Set<String> stooges = Set.of("Larry", "Moe", "Curly");
        Set<String> aflCio = union(guys, stooges);
        System.out.println(aflCio);
    }
}
```
generic singleton factory: Because generics are implemented by erasure, you can use a single object for all required type parameterizations, but you need to write a static factory method to repeatedly dole out the object for each requested type parameterization.
```java
package effectivejava.chapter5.item30;

import java.util.function.UnaryOperator;

// Generic singleton factory pattern (Page 136-7)
public class GenericSingletonFactory {
    // Generic singleton factory pattern
    private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FN;
    }

    // Sample program to exercise generic singleton
    public static void main(String[] args) {
        String[] strings = { "jute", "hemp", "nylon" };
        UnaryOperator<String> sameString = identityFunction();
        for (String s : strings)
            System.out.println(sameString.apply(s));

        Number[] numbers = { 1, 2.0, 3L };
        UnaryOperator<Number> sameNumber = identityFunction();
        for (Number n : numbers)
            System.out.println(sameNumber.apply(n));
    }
}
```
Recursive type bound (rarely used)
```java
package effectivejava.chapter5.item30;
import java.util.*;

// Using a recursive type bound to express mutual comparability (Pages 137-8)
public class RecursiveTypeBound {
    // Returns max value in a collection - uses recursive type bound
    public static <E extends Comparable<E>> E max(Collection<E> c) {
        if (c.isEmpty())
            throw new IllegalArgumentException("Empty collection");

        E result = null;
        for (E e : c)
            if (result == null || e.compareTo(result) > 0)
                result = Objects.requireNonNull(e);

        return result;
    }

    public static void main(String[] args) {
        List<String> argList = Arrays.asList(args);
        System.out.println(max(argList));
    }
}
```
## Item 31: Use bounded wildcards to increase API flexibility
For maximum flexibility, use wildcard types on input parameters that represent producers or consumers. If an input parameter represents a producer, use ```<? extends T>```; if it represents a consumer, use ```<? super T>```. If a parameter should be both producer and consumer, then wildcard types will do you no good.

```java
package effectivejava.chapter5.item31;
import java.util.*;

// Generic stack with bulk methods using wildcard types (Pages 139-41)
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    // The elements array will contain only E instances from push(E).
    // This is sufficient to ensure type safety, but the runtime
    // type of the array won't be E[]; it will always be Object[]!
    @SuppressWarnings("unchecked") 
        public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size==0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

//    // pushAll staticfactory without wildcard type - deficient!
//    public void pushAll(Iterable<E> src) {
//        for (E e : src)
//            push(e);
//    }

     // Wildcard type for parameter that serves as an E producer
    public void pushAll(Iterable<? extends E> src) {
        for (E e : src)
            push(e);
    }

//    // popAll staticfactory without wildcard type - deficient!
//    public void popAll(Collection<E> dst) {
//        while (!isEmpty())
//            dst.add(pop());
//    }

    // Wildcard type for parameter that serves as an E consumer
    public void popAll(Collection<? super E> dst) {
        while (!isEmpty())
            dst.add(pop());
    }

    // Little program to exercise our generic Stack
    public static void main(String[] args) {
        Stack<Number> numberStack = new Stack<>();
        Iterable<Integer> integers = Arrays.asList(3, 1, 4, 1, 5, 9);
        numberStack.pushAll(integers);

        Collection<Object> objects = new ArrayList<>();
        numberStack.popAll(objects);

        System.out.println(objects);
    }
}
```

Comparables are always consumers, so you should generally use ```Comparable<? super T>``` in preference to ```Comparable<T>```. The same is true of comparators.

```java
public static <T extends Comparable<? super T>> T max(List<? extends T> list)
```

SSwap with private helper functyion since the ```List<?>``` can't be put into any value except ```null```.

```java
package effectivejava.chapter5.item31;
import java.util.*;

// Private helper method for wildcard capture (Page 145)
public class Swap {
    public static void swap(List<?> list, int i, int j) {
        swapHelper(list, i, j);
    }

    // Private helper method for wildcard capture
    private static <E> void swapHelper(List<E> list, int i, int j) {
        list.set(i, list.set(j, list.get(i)));
    }

    public static void main(String[] args) {
        // Swap the first and last argument and print the resulting list
        List<String> argList = Arrays.asList(args);
        swap(argList, 0, argList.size() - 1);
        System.out.println(argList);
    }
}
```

## Item 32: Combine generics and varargs judiciously
Heap pollution occurs when a variable of a parameterized type refers to an object that is not of that type.

```java
// Mixing generics and varargs can violate type safety! (Page 146)
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList; // Heap pollution
    String s = stringLists[0].get(0); // ClassCastException
}
```
It is unsafe to store a value in a generic varargs array parameter. Also, don't allow a reference to the array to escape.

```java
// UNSAFE - Exposes a reference to its generic varargs parameter array! (Page 147)
static <T> T[] toArray(T... args) {
    return args;
}
```

Two exceptions: It is safe to pass the generic varargs parameter array to another varargs method that is correctly annotated with ```@SafeVarargs```, and it is safe to pass the array to a non-varargs method that merely computes some function of the contents of the array.

```java
// Safe method with a generic varargs parameter
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

```java
// List as a typesafe alternative to a generic varargs parameter (Page 149)
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```
```java
audience = flatten(List.of(friends, romans, countrymen));
```

## Item 33: Consider typesafe heterogeneous containers
When a class literal is passed among methods to communicate both compile-time and runtime type information, it is known as a *type token*.

```java
package effectivejava.chapter5.item33;
import java.util.*;

// Typesafe heterogeneous container pattern (Pages 151-4)
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }

//    // Achieving runtime type safety with a dynamic cast
//    public <T> void putFavorite(Class<T> type, T instance) {
//        favorites.put(Objects.requireNonNull(type), type.cast(instance));
//    }

    public static void main(String[] args) {
        Favorites f = new Favorites();
        f.putFavorite(String.class, "Java");
        f.putFavorite(Integer.class, 0xcafebabe);
        f.putFavorite(Class.class, Favorites.class);
        String favoriteString = f.getFavorite(String.class);
        int favoriteInteger = f.getFavorite(Integer.class);
        Class<?> favoriteClass = f.getFavorite(Class.class);
        System.out.printf("%s %x %s%n", favoriteString,
                favoriteInteger, favoriteClass.getName());
    }
}
```

There are two limitations to the ```Favorites``` class.
- A malicious client could easily corrupt the type safety of the ```Favorites``` instance by using a ```Class``` object in its raw form, but the resulting client code would generate an unchecked warning when it wass compiled.
- ```Favorite``` class cannot be used on a non-reifiable type such as ```List<String>``` since ```List<String>.class``` is a syntax error.

Bounded type token
```java
package effectivejava.chapter5.item33;
import java.lang.annotation.*;
import java.lang.reflect.*;

// Use of asSubclass to safely cast to a bounded type token (Page 155)
public class PrintAnnotation {
    static Annotation getAnnotation(AnnotatedElement element,
                                    String annotationTypeName) {
        Class<?> annotationType = null; // Unbounded type token
        try {
            annotationType = Class.forName(annotationTypeName);
        } catch (Exception ex) {
            throw new IllegalArgumentException(ex);
        }
        return element.getAnnotation(
                annotationType.asSubclass(Annotation.class));
    }

    // Test program to print named annotation of named class
    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.out.println(
                "Usage: java PrintAnnotation <class> <annotation>");
            System.exit(1);
        }
        String className = args[0];
        String annotationTypeName = args[1]; 
        Class<?> klass = Class.forName(className);
        System.out.println(getAnnotation(klass, annotationTypeName));
    }
}
```