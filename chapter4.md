# chapter 4: Classes and Interfaces

## Item 15: Minimize the accessibility of classes and members

- Package-private: The member is accessible from any class in the package where it is declared. Technically known as package access, this is the access level you get if no access modifier is specified (except for interface members, which are public by default)
- If a method overrides a superclass method, it cannot have a more restrictive access level than the overridden method. A special case of this rule is that if a class implements an interface, all of the class methods that are in the interface must be declared public in the class.
- Instance fields of public classes should rarely be public with one exception to constants via public static final fields. It is critical that these fields contain either primitive values or references to immutable objects.
    - Note that a nonzero-length array is always mutable, so it is wrong for a class to have a public static final array field, or an accessor that returns such a field.
```java
// Potential security hole!
public static final Thing[] VALUES = {...};
```
- As of Java 9, public and protected members of unexported packages in a module are inaccessible outside the module. But these two implicit access levels are largely advisory. If you place a module's JAR file on your appliation's class path instead of its module path, the packages in the module revert to their non-modular behavior.


## Item 16: In public classes, use accessor methods, not public fields
Public classes should never expose mutable fileds (should provide accessor methods like getter and setter). It is less harmful, though still questionable, for public classes to expose immutable fields since you can't change the representation of such a class without changing its API, and you can't take auxiliary actions when the field is read, but you can enforce invariants. It is, however, sometimes desirable for package-private or private nested classes to expose fields, whether mutable or immutable.


## Item 17: Minimize mutability
Example: have accessors for each attribute but no corresponding mutators.
```java
package effectivejava.chapter4.item17;

// Immutable complex number class (Pages 81-82)
public final class Complex {
    private final double re;
    private final double im;

    public static final Complex ZERO = new Complex(0, 0);
    public static final Complex ONE  = new Complex(1, 0);
    public static final Complex I    = new Complex(0, 1);

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart()      { return re; }
    public double imaginaryPart() { return im; }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    // Static factory, used in conjunction with private constructor (Page 85)
    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                re * c.im + im * c.re);
    }

    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                (im * c.re - re * c.im) / tmp);
    }

    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Complex))
            return false;
        Complex c = (Complex) o;

        // See page 47 to find out why we use compare instead of ==
        return Double.compare(c.re, re) == 0
                && Double.compare(c.im, im) == 0;
    }
    @Override public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }

    @Override public String toString() {
        return "(" + re + " + " + im + "i)";
    }
}
```
All arithmetic operations create and return a new ```Complex``` instance rather than modifying this instance. It is known as the functional approach because methods return the result of applying a function to their operand, without modifying it.

The major disadvantage of immutable classes is that they require a separat object for each distinct value and creating these objects can be costly.

There are two approaches to coping with the performance problem. The first is to guess which multistep operations will be commonly required and to provide them as primitves. If a multistep operation is provided as a primitive, the immutable does not have to create a separate object at each step. For example, ```BigInteger``` has a package-private mutable "companion class" that it uses to speed up multistep operations such as modular exponentiation. If you cannot predict accurately, then your best bet is to provide a public mutable companion class, which is similar to ```StringBuilder``` for ```String```.

Design alternatives to make an immutable class - make all of its constructors private or package-private and add public static factories in place of the public constructors.

```java
// Immutable class with static factories instead of constructors (Page 84)
public final class Complex {
    private final double re;
    private final double im;

    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }

    // Remainder omitted
}
```

## Item 18: Favor composition over inheritance
Unlike method invocation, inheritance violates encapsulation. Inheriting from ordinary concrete classes across package boundaries is dangerous.

```java
package effectivejava.chapter4.item18;
import java.util.*;

// Broken - Inappropriate use of inheritance! (Page 87)
public class InstrumentedHashSet<E> extends HashSet<E> {
    // The number of attempted element insertions
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
        s.addAll(List.of("Snap", "Crackle", "Pop"));
        System.out.println(s.getAddCount()); // count is 6, not 3!
    }
}
```
It returns 6 because internally ```HashSet```'s ```addAll``` method is implemented on top of its ```add``` method. The "self-use" is an implementation detail, not guaranteed to hold in all implementations of Java platform and subject to change from release to release.

Moreover, the superclass can acquire new methods in subsequent releases. Suppose a program depends for its security on the fact that all elements inserted into some collection satisfy some predicate. This works fine until a new method capable of inserting an elment is added to the superclass which may possibly add an "illegal" element.

Instead of extending an existing class, give your new class a private field that references an instance of the existing class. The design is called **composition** because the exsting class becomes a component of the new one. Each instance method in the new class invokes the corresponding method on the existing object and returns the result. This is known as **forwarding** and the methods in the new class are known as forwarding methods.

```java
package effectivejava.chapter4.item18;
import java.util.*;

// Reusable forwarding class (Page 90)
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }

    public void clear()               { s.clear();            }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty()          { return s.isEmpty();   }
    public int size()                 { return s.size();      }
    public Iterator<E> iterator()     { return s.iterator();  }
    public boolean add(E e)           { return s.add(e);      }
    public boolean remove(Object o)   { return s.remove(o);   }
    public boolean containsAll(Collection<?> c)
                                   { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c)
                                   { return s.addAll(c);      }
    public boolean removeAll(Collection<?> c)
                                   { return s.removeAll(c);   }
    public boolean retainAll(Collection<?> c)
                                   { return s.retainAll(c);   }
    public Object[] toArray()          { return s.toArray();  }
    public <T> T[] toArray(T[] a)      { return s.toArray(a); }
    @Override public boolean equals(Object o)
                                       { return s.equals(o);  }
    @Override public int hashCode()    { return s.hashCode(); }
    @Override public String toString() { return s.toString(); }
}
```
```java
package effectivejava.chapter4.item18;
import java.util.*;

// Wrapper class - uses composition in place of inheritance  (Page 90)
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        InstrumentedSet<String> s = new InstrumentedSet<>(new HashSet<>());
        s.addAll(List.of("Snap", "Crackle", "Pop"));
        System.out.println(s.getAddCount()); // return 3 now!
    }
}
```


## Item 19: Design and document for inheritance or else prohibit it

The class must document its self-use of overridable methods (By overridable, we mean nonfinal and either public or protected). Also, a class may have to provide hooks into its internal workings in the form of judiciously chosen methods.

For example, consider the ```removeRange``` method from ```java.util.AbstractList```:

```
protected void removeRange(int fromIndex, int toIndex)
    Removes from this list all of the elements whose index is between fromIndex, inclusive, and toIndex, exclusive. Shifts any succeeding elements to the left (reduces their index). This call shortens the list by (toIndex - fromIndex) elements. (If toIndex==fromIndex, this operation has no effect.)

    This method is called by the clear operation on this list and its sublists. Overriding this method to take advantage of the internals of the list implementation can <b>substantially</b> improve the performance of the clear operation on this list and its sublists.

    Implementation Requirements: This implementation gets a list iterator positioned before fromIndex, and repeatedly calls ListIterator.next followed by ListIterator.remove, until the entore range has been removed. Note: If ListIterator.remove requires linear time, this implementation requires quadratic time.
```

Constructors must not invoke overridable methods:
```java
package effectivejava.chapter4.item19;

// Class whose constructor invokes an overridable method. NEVER DO THIS! (Page 95)
public class Super {
    // Broken - constructor invokes an overridable method
    public Super() {
        overrideMe();
    }

    public void overrideMe() {
    }
}
```

```java
package effectivejava.chapter4.item19;

import java.time.Instant;

// Demonstration of what can go wrong when you override a method  called from constructor (Page 96)
public final class Sub extends Super {
    // Blank final, set by constructor
    private final Instant instant;

    Sub() {
        instant = Instant.now();
    }

    // Overriding method invoked by superclass constructor
    @Override public void overrideMe() {
        System.out.println(instant);
    }

    public static void main(String[] args) {
        Sub sub = new Sub();
        sub.overrideMe(); // print null, instant
    }
}
```
Thus the class never invokes any of its overridable methods and to document this fact. You can eliminate a class's self-use of overridable methods mechanically, without changing its behavior. Move the body of each overridable method to a private "helper method" and have each overridable method invoke its private helper method. Then replace each self-use of an overridable method with a direct invocation of the overridable method's private helper method.

## Item 20: Prefer interfaces to abstract classes
There are limits on how much implementation assistance you can provide with default methods. Although many interfaces specify the behavior of Object methods such as ```equals``` and ```hashCode```, you are not permitted to provide default methods for them. Also, interfaces are not permitted to contain instance fields or nonpublic static members (with the exception of private static methods). Finally, you can't add default methods to an interface that you don't control.

You can, however, combine the advantages of interfaces and abstract classes by providing an abstract skeletal implementation class to go with an interface. 

```java
package effectivejava.chapter4.item20;
import java.util.*;

// Concrete implementation built atop skeletal implementation (Page 101)
public class IntArrays {
    static List<Integer> intArrayAsList(int[] a) {
        Objects.requireNonNull(a);

        // The diamond operator is only legal here in Java 9 and later
        // If you're using an earlier release, specify <Integer>
        return new AbstractList<>() {
            @Override public Integer get(int i) {
                return a[i];  // Autoboxing (Item 6)
            }

            @Override public Integer set(int i, Integer val) {
                int oldVal = a[i];
                a[i] = val;     // Auto-unboxing
                return oldVal;  // Autoboxing
            }

            @Override public int size() {
                return a.length;
            }
        };
    }

    public static void main(String[] args) {
        int[] a = new int[10];
        for (int i = 0; i < a.length; i++)
            a[i] = i;

        List<Integer> list = intArrayAsList(a);
        Collections.shuffle(list);
        System.out.println(list);
    }
}
```
```java
package effectivejava.chapter4.item20;
import java.util.*;

// Skeletal implementation class (Pages 102-3)
public abstract class AbstractMapEntry<K,V>
        implements Map.Entry<K,V> {
    // Entries in a modifiable map must override this method
    @Override public V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    
    // Implements the general contract of Map.Entry.equals
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry) o;
        return Objects.equals(e.getKey(),   getKey())
                && Objects.equals(e.getValue(), getValue());
    }

    // Implements the general contract of Map.Entry.hashCode
    @Override public int hashCode() {
        return Objects.hashCode(getKey())
                ^ Objects.hashCode(getValue());
    }

    @Override public String toString() {
        return getKey() + "=" + getValue();
    }
}
```

## Item 21: Design interfaces for posterity
Adding new default method to existing interfaces is fraught with risk since it is not always possible to write a default method that maintains all invariants of every conceivable implementation.

## Item 22: Use interfaces only to define types
The constant interface pattern is a poor use of interfaces.

```java
package effectivejava.chapter4.item22.constantinterface;

// Constant interface antipattern - do not use!
public interface PhysicalConstants {
    // Avogadro's number (1/mol)
    static final double AVOGADROS_NUMBER   = 6.022_140_857e23;

    // Boltzmann constant (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;

    // Mass of the electron (kg)
    static final double ELECTRON_MASS      = 9.109_383_56e-31;
}
```
It represents a commitment: if in a future release the class is modified so that it no longer needs to use the constants, it still must implement the interface to ensure binary compatibility. If a nonfinal class implements a cnstant interface, all of its subclasses will have their namespaces polluted by the constants in the interface.

You should export the constants with a noninstantiable utility class. If you make heavy use of the constants exported by the utiliy class, you can import them statically.

```java
package effectivejava.chapter4.item22.constantutilityclass;

// Constant utility class (Page 108)
public class PhysicalConstants {
  private PhysicalConstants() { }  // Prevents instantiation

  // Avogadro's number (1/mol)
  public static final double AVOGADROS_NUMBER = 6.022_140_857e23;

  // Boltzmann constant (J/K)
  public static final double BOLTZMANN_CONST  = 1.380_648_52e-23;

  // Mass of the electron (kg)
  public static final double ELECTRON_MASS    = 9.109_383_56e-31;

  public static void main(String[] args) {
    double atoms = PhysicalConstants.AVOGADROS_NUMBER;
    double k = PhysicalConstants.BOLTZMANN_CONST;
    double m = PhysicalConstants.ELECTRON_MASS;
    System.out.println(atoms);
    System.out.println(k);
    System.out.println(m);
  }
}
```
 
## Item 23: Prefer class hierarchies to tagged classes
```java
package effectivejava.chapter4.item23.taggedclass;

// Tagged class - vastly inferior to a class hierarchy! (Page 109)
class Figure {
    enum Shape { RECTANGLE, CIRCLE };

    // Tag field - the shape of this figure
    final Shape shape;

    // These fields are used only if shape is RECTANGLE
    double length;
    double width;

    // This field is used only if shape is CIRCLE
    double radius;

    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}
```
Change it to class hierarchy.

```java
package effectivejava.chapter4.item23.hierarchy;

// Class hierarchy replacement for a tagged class  (Page 110-11)
abstract class Figure {
    abstract double area();
}
```
```java
package effectivejava.chapter4.item23.hierarchy;

// Class hierarchy replacement for a tagged class  (Page 110-11)
class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width  = width;
    }
    @Override double area() { return length * width; }
}
```
```java
package effectivejava.chapter4.item23.hierarchy;

// Class hierarchy replacement for a tagged class  (Page 110-11)
class Square extends Rectangle {
    Square(double side) {
        super(side, side);
    }
}
```


## Item 24: Favor static member classes over nonstatic
There are four kinds of nested classes: static member classes, nonstatic member classes, anonymous classes and local classes.

If an instance of a nested class can exist in isolation from an instance of its enclosing class, then the nested class must be a static member class. If you declare it as a nonstatic member class, it will have an implicit reference to its enclosing instance. This reference will prevent the enclosing instance from being garbage collected.

```java
// Typical use of a nonstatic member class (Page 113)
public class MySet<E> extends AbstractSet<E> {
    ...
    private class MyIterator implements Iterator<E> {
        ...
    }
    @Override public Iterator<E> iterator() { return new MyIterator(); }
}
```
A common use of private static member classes is to represent components of the object represented by their enclosing class. For example, consider a Map instanfce, which associates keys with values. Many Map implementations have an internal Entry object for each key-value pair in the map. While each entry is associated with a map, the methods on an entry (getKey, getVlue and setValue) do not need access to the map. Therefore, a private static member class is best.

If a nested class needs to be visible outside of a single method or is too long to fit comfortably inside a method, use a member class. If each instance of a member class needs a reference to its enclosing instance, make it nonstatic; otherwise, make it static. Assuming the class belongs inside a method, if you need to create instance from only one location and there is a preexisting type that characterizes the class, make it an anonymous class; otherwise, make it a local class.

## Item 25: Limit source files to a single top-level class
```java
// Utensil.java
package effectivejava.chapter4.item25;

// Two classes defined in one file. Don't ever do this!  (Page 115)
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```
```java
// Dessert.java
package effectivejava.chapter4.item25;

// Two classes defined in one file. Don't ever do this!  (Page 115)
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
}
```
The compilation may fail, or print pancake, or print potpie.

If you're tempted to put multiple top-level classes into a single source file, consider using static member class as an alternative to splitting the classes into separate source files.

```java
package effectivejava.chapter4.item25;

// Static member classes instead of multiple top-level classes (Page 116)
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {
        static final String NAME = "cake";
    }
}
```