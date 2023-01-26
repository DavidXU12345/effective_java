# Chapter 12: Serialization

## Item 85: Prefer alternatives to Java serialization
In the process of deserializing a byte stream, this method can execute code from almost any type on the class path, so long as the type implements the ```Serializable``` interface, so the code for all of these types is part of the attack surface.

Attack examples:
- Attackers and security researchers study the serializable types in the Java libraries and in commonly used third-party libraries, looking for methods invoked during deserialization that perform potentially dangerous activities. Such methods are known as gadgets. Multiple gadgets can be used in concert, to form a gadget chain. From time to time, a gadget chain is discovered that is sufficiently powerful to allow an attacker to execute arbitrary native code on the underlying hardware, given only the opportunity to submit a carefully crafted byte stream for deserial- ization.
- Without using any gadgets, you can easily mount a denial-of-service attack by causing the deserialization of a short stream that requires a long time to deserial- ize. Such streams are known as *deserialization bombs*.

```java
package effectivejava.chapter12.item85;
import static effectivejava.chapter12.Util.*;

import java.util.HashSet;
import java.util.Set;

// Deserialization bomb - deserializing this stream takes forever - Page 340
public class DeserializationBomb {
    public static void main(String[] args) throws Exception {
        System.out.println(bomb().length);
        deserialize(bomb());
    }

    static byte[] bomb() {
        Set<Object> root = new HashSet<>();
        Set<Object> s1 = root;
        Set<Object> s2 = new HashSet<>();
        for (int i = 0; i < 100; i++) {
            Set<Object> t1 = new HashSet<>();
            Set<Object> t2 = new HashSet<>();
            t1.add("foo"); // make it not equal to t2
            s1.add(t1);
            s1.add(t2);
            s2.add(t1);
            s2.add(t2);
            s1 = t1;
            s2 = t2;
        }
        return serialize(root);
    }
}
```

In summary, serialization is dangerous and should be avoided. If you are designing a system from scratch, use a cross-platform structured-data representation such as *JSON* or *protobuf* instead. Do not deserialize untrusted data. If you must do so, use object deserialization filtering (prefer whitelisting to blacklisting), but be aware that it is not guaranteed to thwart all attacks. Avoid writing serializable classes. If you must do so, exercise great caution.

## Item 86: Implement ```Serializable``` with great caution
Inner classes (Item 24) should not implement ```Serializable```. They use compiler-generated synthetic fields to store references to enclosing instances and to store values of local variables from enclosing scopes. How these fields corre- spond to the class definition is unspecified, as are the names of anonymous and local classes. Therefore, the default serialized form of an inner class is ill-defined. A static member class can, however, implement Serializable.

## Item 87: Consider using a custom serialized form
The default serialized form is likely to be appropriate if an object’s phys- ical representation is identical to its logical content. For example, the default serialized form would be reasonable for the following class, which simplistically represents a person’s name:

```java
// Good candidate for default serialized form - Page 346
public class Name implements Serializable{
    /**
     * Last name. Must be non-null.
     * @serial
     */
    private final String lastName;

    /**
     * First name. Must be non-null.
     * @serial
     */
    private final String firstName;

    /**
     * Middle name, or null if there is none.
     * @serial
     */
    private final String middleName;

    ... // Remainder omitted
}

Near the opposite end of the spectrum from Name, consider the following class, which represents a list of strings:

```java
// Awful candidate for default serialized form - Page 347
public class StringList implements Serializable {
    private int size = 0;
    private Entry head = null;

    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }

    ... // Remainder omitted
}
```

The serialized form will painstakingly mirror every entry in the linked list and all the links between the entries, in both directions.

A reasonable serialized form for ```StringList``` is simply the number of strings in the list, followed by the strings themselves. This constitutes the logical data represented by a ```StringList```, stripped of the details of its physical representation. Here is a revised version of ```StringList``` with ```writeObject``` and ```readObject``` methods that implement this serialized form. As a reminder, the transient modifier indicates that an instance field is to be omitted from a class’s default serialized form:

```java
// StringList with a reasonable custom serialized form - Page 349
public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;

    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }

    // Appends the specified string to the list
    public final void add(String s) { ... }

    /**
     * Serialize this {@code StringList} instance.
     *
     * @serialData The size of the list (the number of strings
     * it contains) is emitted ({@code int}), followed by all of
     * its elements (each a {@code String}), in the proper
     * sequence.
     */
    private void writeObject(ObjectOutputStream s)
            throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (Entry e = head; e != null; e = e.next)
            s.writeObject(e.data);
    }

    /**
     * Deserialize this {@code StringList} instance.
     *
     * @serialData The size of the list (the number of strings
     * it contains) is emitted ({@code int}), followed by all of
     * its elements (each a {@code String}), in the proper
     * sequence.
     */
    private void readObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();

        // Read in all elements and insert them in list
        for (int i = 0; i < numElements; i++)
            add((String) s.readObject());
    }

    ... // Remainder omitted
}
```

Declare an explicit serial version UID in every serialiable class you write.

```java
private static final long serialVersionUID = randomLongValue;
```

If you modify an existing class that lacks a serial version UID, and you want the new version to accept existing serialized instances, you must use the value that was automatically generated for the old version. You can get this number by running the ```serialver``` utility on the old version of the class—the one for which serialized instances exist.


## Item 88: Write readObject methods defensively
Loosely speaking, ```readObject``` is a constructor that takes a byte stream as its sole parameter. The problem arises when ```readObject``` is presented with a byte stream that is artificially constructed to generate an object that violatyes the invariants of its class.

To fix this problem, provide a readObject method for Period that calls defaultReadObject and then checks the validity of the deserialized object. If the validity check fails, the readObject method throws ```InvalidObjectException```, preventing the deserialization from completing:

```java
// readObject method with validity checking - Page 355
private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {
    s.defaultReadObject();

    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start + " after " + end);
}
```

However, it is possible to create a mutable Period instance by fabricating a byte stream that begins with a valid Period instance and then appends extra references to the private Date fields internal to the Period instance. The attacker reads the Period instance from the ```ObjectInputStream``` and then reads the “rogue object references” that were appended to the stream. These references give the attacker access to the objects referenced by the private ```Date``` fields within the ```Period``` object. By mutating these Date instances, the attacker can mutate the Period instance.

When an object is deserialized, it is critical to defensively copy any field containing an object reference that a client must not possess.

```java
// readObject method with defensive copying - Page 357
private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {
    s.defaultReadObject();

    // Defensively copy our mutable components
    start = new Date(start.getTime());
    end = new Date(end.getTime());

    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start + " after " + end);
}
```

## Item 89: For instance control, prefer enum types to readResolve

The ```readResolve``` feature allows you to substitute another instance for the one created by ```readObject```. If the class of an object being deserialzed defines a ```readResolve``` method with the proper declaration, this method is invoked on the newly created object after it is deserialized. The object reference returned by this method is then returned in place of the newly created object. In most uses of this feature, no reference to the newly created object is retained, so it immediately becomes eligible for garbage collection.

If you depend on ```readResolve``` for instance control, all instance fields with object reference types must be declared ```transient```. Otherwise, it is possible for a determined attacker to secure a reference to the deserialized object before its readResolve method is run, using a technique that is somewhat similar to the MutablePeriod attack in Item 88.


```java
// Broken singleton - has nontransient object reference field!
public class Elvis implements Serializable{
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {}

    private String[] favoriteSongs = { "Hound Dog", "Heartbreak Hotel" };

    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }

    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }

    private Object readResolve() {
        return INSTANCE;
    }
}
```
You could fix the problem by declaring the favoriteSongs field transient, but you’re better off fixing it by making Elvis a single-element enum type (Item 3).

If you write your serializable instance-controlled class as an enum, Java guarantees you that there can be no instances besides the declared constants, unless an attacker abuses a privileged method such as ```AccessibleObject.setAccessible```. Any attacker who can do that already has sufficient privileges to execute arbitrary native code, and all bets are off. Here’s how our Elvis example looks as an enum:

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;

    private String[] favoriteSongs = { "Hound Dog", "Heartbreak Hotel" };

    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
}
```


## Item 90: Consider serialization proxies instead of serialized instances

 First, design a private static nested class that concisely represents the logical state of an instance of the enclosing class. This nested class is known as the serialization proxy of the enclosing class. It should have a single constructor, whose parameter type is the enclosing class. This constructor merely copies the data from its argument: it need not do any consistency checking or defensive copying. By design, the default serialized form of the serialization proxy is the perfect serialized form of the enclosing class. Both the enclosing class and its serialization proxy must be declared to implement ```Serializable```.

For example, consider the immutable Period class written in Item 50 and made serializable in Item 88. Here is a serialization proxy for this class. Period is so simple that its serialization proxy has exactly the same fields as the class:

```java
// Serialization proxy for Period class - Page 363
private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;

    SerializationProxy(Period p) {
        this.start = p.start;
        this.end = p.end;
    }

    private static final long serialVersionUID = 234098243823485285L; // Any number will do (Item 87)
}
```

Next, add the following ```writeReplace``` method to the **enclosing class**. This method can be copied verbatim into any class with a serialization proxy:

```java
// writeReplace method for the serialization proxy pattern
private Object writeReplace() {
    return new SerializationProxy(this);
}
```

With this ```writeReplace``` method in place, the serialization system will never generate a serialized instance of the enclosing class, but an attacker might fabricate one in an attempt to violate the class’s invariants. To guarantee that such an attack would fail, merely add this readObject method to the **enclosing class**:

```java
// readObject method for the serialization proxy pattern
private void readObject(ObjectInputStream stream)
        throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required");
}
```
Finally, provide a ```readResolve``` method on the SerializationProxy class that returns a logically equivalent instance of the enclosing class. The presence of this method causes the serialization system to translate the serialization proxy back into an instance of the enclosing class upon deserialization.

```java
// readResolve method for Period.SerializationProxy - Page 364
private Object readResolve() {
    return new Period(start, end); // Uses public constructor
}
```