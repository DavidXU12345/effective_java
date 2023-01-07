# Chapter 3: Methods common to all objects

## Item 10: Obey the general contract when overriding equals

The equals method implements an *equivalence relation*. It has these properties:
- Reflexive: For any non-null reference value x, ```x.equals(x)``` must return true.
- Symmetric: For any non-null reference values x and y, ```x.equals(y)``` must return true if and only if ```y.equals(x)``` returns true.
- Transitive: For any non-null reference values x, y, and z, if ```x.equals(y)``` returns true and ```y.equals(z)``` returns true, then ```x.equals(z)``` must return true.
- Consistent: For any non-null reference values x and y, multiple invocations of ```x.equals(y)``` consistently return true or consistently return false, provided no information used in equals comparisons on the objects is modified.
- For any non-null reference value x, ```x.equals(null)``` must return false.

Here is a recipe for the high-quality equals method:
- Use the ```==``` operator to check if the argument is a reference to this object. (To check whether the object is itself)
- Use the ```instanceof``` operator to check if the argument has the correct type. (Don't use ```getClass``` to check the type of the argument, since it is possible that the argument is a subclass of the class of this object.)
- Cast the argument to the correct type.
- For each "significant" field in the class, check if that field of the argument matches the corresponding field of this object.
    - For primitive fields whose type is not ```float``` or ```double```, use the ```==``` operator for comparisons; for object reference fields, use the ```equals``` method recursively; for ```float``` fields, use the static ```Float.compare(float, float)``` method; for ```double``` fields, use the static ```Double.compare(double, double)``` method. 
    - Since ```Float.equals``` and ```Double.equals``` would entail autoboxing on every comparison, which would have poor performance.

Example:
```java
package effectivejava.chapter3.item10;

// Class with a typical equals method (Page 48)
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix   = rangeCheck(prefix,   999, "prefix");
        this.lineNum  = rangeCheck(lineNum, 9999, "line num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }

    // Remainder omitted - note that hashCode is REQUIRED (Item 11)!
}
```
Recommend to use **Google's open source AutoValue** framework to generate these methods for you, triggered by a single annotation on the class.

If you want to add a value component to a class that overrides ```equals```, don't extend it. Write an unrelated class containing an instance of the first class. Then provide a "view" method that returns the contained instance.

Example:
```java
// Adds a value component without violating the equals contract (Page 44)
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    // Returns the point-view of this color point
    public Point asPoint() {
        return point;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint)o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```

## Item 11: Always override hashCode when you override equals

Equal objects must have equal hash codes while it is not required for unequal objects to have distinct hash codes.

Recipe to write a hash function that distributes any reasonable collection of unequal instances uniformly across all ```int``` values:
1. Declare an ```int``` variable named ```result```, and initialize it to the hash code c of the first significant field in the object, as computed in step 2.1..
2. For every remaining significant field f in the object, do the following:
    1. Compute an ```int``` hash code ```c``` for the field:
        1. If the field is of a primitive type, compute ```Type.hashCode(f)```, where ```Type``` is the boxed primitive class corresponding to f's type.
        2. If the field is an object reference and this class's equals method compares the field by recursively invoking equals, recursively invoke hashCode on the field. If a more complex comparison is required, compute a "canonical representation" for this field and invoke hashCode on the canonical representation. If the value of the field is null, return 0 (or some other constant, but 0 is traditional).
        3. If the field is an array, treat it as if each element were a separate field. That is, compute a hash code for each significant element by applying these rules recursively, and combine these values per step 2.2. If every element in an array field is significant, use ```Arrays.hashCode```.
    2. Combine the hash code ```c``` computed in step 2.1. into ```result``` as follows: ```result = 31 * result + c```.
3. Return ```result```.

We can also use a static method under ```Objects``` class to generate the hash code for us, but they run more slowly because they entail array creation to pass a variable number of arguments as well as boxing and unboxing:
```java
@override public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

Example of hashCode recipe:
```java
package effectivejava.chapter3.item11;
import java.util.*;

// Shows the need for overriding hashcode when you override equals (Pages 50-53 )
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix   = rangeCheck(prefix,   999, "prefix");
        this.lineNum  = rangeCheck(lineNum, 9999, "line num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }


    // Broken with no hashCode; works with any of the three below

//    // Typical hashCode method (Page 52)
//    @Override public int hashCode() {
//        int result = Short.hashCode(areaCode);
//        result = 31 * result + Short.hashCode(prefix);
//        result = 31 * result + Short.hashCode(lineNum);
//        return result;
//    }

//    // One-line hashCode method - mediocre performance  (page 53)
//    @Override public int hashCode() {
//        return Objects.hash(lineNum, prefix, areaCode);
//    }

//    // hashCode method with lazily initialized cached hash code  (page 53)
//    private int hashCode; // Automatically initialized to 0
//
//    @Override public int hashCode() {
//        int result = hashCode;
//        if (result == 0) {
//            result = Short.hashCode(areaCode);
//            result = 31 * result + Short.hashCode(prefix);
//            result = 31 * result + Short.hashCode(lineNum);
//            hashCode = result;
//        }
//        return result;
//    }

    public static void main(String[] args) {
        Map<PhoneNumber, String> m = new HashMap<>();
        m.put(new PhoneNumber(707, 867, 5309), "Jenny");
        System.out.println(m.get(new PhoneNumber(707, 867, 5309)));
    }
}
```
Reference: com.google.common.hash.Hashing -> Guava

## Item 12: Always override toString

```java
package effectivejava.chapter3.item12;

// Adding a toString method to PhoneNumber (page 52)
public final class PhoneNumber {
    ... omitted ...

    /**
     * Returns the string representation of this phone number.
     * The string consists of twelve characters whose format is
     * "XXX-YYY-ZZZZ", where XXX is the area code, YYY is the
     * prefix, and ZZZZ is the line number. Each of the capital
     * letters represents a single decimal digit.
     *
     * If any of the three parts of this phone number is too small
     * to fill up its field, the field is padded with leading zeros.
     * For example, if the value of the line number is 123, the last
     * four characters of the string representation will be "0123".
     */
    @Override public String toString() {
        return String.format("%03d-%03d-%04d",
                areaCode, prefix, lineNum);
    }

    public static void main(String[] args) {
        PhoneNumber jenny = new PhoneNumber(707, 867, 5309);
        System.out.println("Jenny's number: " + jenny);
    }
}
```

## Item 13: Override clone judiciously
Suppose you want to implement ```Cloneable``` in a class whose superclass provides a well-behaved ```clone``` method. First call ```super.clone()```. The object you get back will be a fully functional replica of the original. Any fields declared in your class will have values identical to those of the original. If every field contains a primitive value or a refernce to an immutable object, the returned object may be exactly what you want.

```java
    // Clone method for class with no references to mutable state (Page 59)
    @Override public PhoneNumber clone() {
        try {
            return (PhoneNumber) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();  // Can't happen
        }
    }
```

But if an object contains fields that refer to mutable objects, we need to do deep copy by calling clone recursively or iteratively.

```java
package effectivejava.chapter3.item13;
import java.util.Arrays;

// A cloneable version of Stack (Pages 60-61)
public class Stack implements Cloneable {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size ==0;
    }

    // Clone method for class with references to mutable state
    @Override public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone(); //array.clone is deep copy
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }

    // Ensure space for at least one more element.
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
    
    // To see that clone works, call with several command line arguments
    public static void main(String[] args) {
        Stack stack = new Stack();
        for (String arg : args)
            stack.push(arg);
        Stack copy = stack.clone();
        while (!stack.isEmpty())
            System.out.print(stack.pop() + " ");
        System.out.println();
        while (!copy.isEmpty())
            System.out.print(copy.pop() + " ");
    }
}
```
```java
// Recursively clone method for class with complex mutable state (Page 62)
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }

        // Recursively copy the linked list headed by this Entry
        Entry deepCopy() {
            return new Entry(key, value,
                    next == null ? null : next.deepCopy());
        }
    }

    @Override public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```
Or we can do the deep copy iteratively to prevent stack overflow:
```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

A **better** approach to object coping is to provdie a copy constructor or copy factory:
```java
// Copy constructor
public Yum(Yum yum) {
    this.salt = yum.salt;
    this.sugar = yum.sugar;
    this.flavor = yum.flavor;
}

// Copy factory
public static Yum newInstance(Yum yum) {
    return new Yum(yum);
}
```
As a rule, copy functionality is best provided by constructors or factories. A notable exception to this rule is arrays, which are best copied with the clone method.

## Item 14: Consider implementing Comparable
```compareTo``` method is not declared in ```Object```. Rather, it is the sole method in the ```Comparable``` interface. Classes that depend on comparison include the sorted collections ```TreeSet``` and ```TreeMap``` and the utility classes ```Collections``` and ```Arrays```, which contains searching and sorting algorithms.

- The implementor must ensure that ```sgn(x.compareTo(y)) == -sgn(y.compareTo(x))``` for all ```x``` and ```y```. (This implies that ```x.compareTo(y)``` must throw an exception if and only if ```y.compareTo(x)``` throws an exception.)
- The implementor must also ensure that the relation is transitive: ```(x.compareTo(y) > 0 && y.compareTo(z) > 0)``` implies ```x.compareTo(z) > 0```.
- Finally, the implementor must ensure that ```x.compareTo(y) == 0``` implies that ```sgn(x.compareTo(z)) == sgn(y.compareTo(z))```, for all ```z```.
- It is strongly recommended, but not strictly required that ```(x.compareTo(y)==0) == (x.equals(y))```. Generally speaking, any class that implements the ```Comparable``` interface and violates this condition should clearly indicate this fact. The recommended language is "Note: this class has a natural ordering that is inconsistent with equals."

```java
// Single-field Comparable with object reference field (Page 69)
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    @Override public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }

    // Remainder omitted
}
```
```java
   // Multiple-field Comparable with primitive fields (page 69)
   public int compareTo(PhoneNumber pn) {
       int result = Short.compare(areaCode, pn.areaCode);
       if (result == 0)  {
           result = Short.compare(prefix, pn.prefix);
           if (result == 0)
               result = Short.compare(lineNum, pn.lineNum);
       }
       return result;
   }
```
```java
    // Comparable with comparator construction methods (page 70)
    private static final Comparator<PhoneNumber> COMPARATOR =
            comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt(pn -> pn.prefix)
                    .thenComparingInt(pn -> pn.lineNum);

    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
```
```java
// BROKEN difference-based compareTo method (Page 71)
static Comparator<object> hashCodeOrder = new Comparator<Object>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```
Difference-based compareTo method is not recommended since it may cause integer overflow.

In Java 7, static compare methods were added to all of Java's boxed primitive classes. Thus use of the relational operators < and > in compareTo methods is now discouraged.
```java
// Comparator based on static compare method
static Comparator<Object> hashCodeOrder = new Comparator<Object>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};
```