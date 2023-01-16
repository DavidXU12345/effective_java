# Chapter 6: Enums and Annotations
## Item 34: Use enums instead of int constants
```java
package effectivejava.chapter6.item34;

// Enum type with data and behavior  (159-160)
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS  (4.869e+24, 6.052e6),
    EARTH  (5.975e+24, 6.378e6),
    MARS   (6.419e+23, 3.393e6),
    JUPITER(1.899e+27, 7.149e7),
    SATURN (5.685e+26, 6.027e7),
    URANUS (8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);

    private final double mass;           // In kilograms
    private final double radius;         // In meters
    private final double surfaceGravity; // In m / s^2

    // Universal gravitational constant in m^3 / kg s^2
    private static final double G = 6.67300E-11;

    // Constructor
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }

    public double mass()           { return mass; }
    public double radius()         { return radius; }
    public double surfaceGravity() { return surfaceGravity; }

    public double surfaceWeight(double mass) {
        return mass * surfaceGravity;  // F = ma
    }
}
```
Note that ```Planet```, like all enums, has a static ```values``` method that returns an array of its values in the order they were declared.
```java
package effectivejava.chapter6.item34;

// Takes earth-weight and prints table of weights on all planets (Page 160)
public class WeightTable {
   public static void main(String[] args) {
      double earthWeight = Double.parseDouble(args[0]);
      double mass = earthWeight / Planet.EARTH.surfaceGravity();
      for (Planet p : Planet.values())
         System.out.printf("Weight on %s is %f%n",
                 p, p.surfaceWeight(mass));
   }
}
```
There is a way to associate a different bahavior with each enum constant: declare an abstract method in the enum type and provide an implementation for each constant-specific class body. This technique is known as the *constant-specific method implementation*.
```java
package effectivejava.chapter6.item34;
import java.util.*;
import java.util.stream.Stream;

import static java.util.stream.Collectors.toMap;

// Enum type with constant-specific class bodies and data (Pages 163-4)
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    @Override public String toString() { return symbol; }

    public abstract double apply(double x, double y);

    // Implementing a fromString method on an enum type (Page 164)
    private static final Map<String, Operation> stringToEnum =
            Stream.of(values()).collect(
                    toMap(Object::toString, e -> e));

    // Returns Operation for string, if any
    public static Optional<Operation> fromString(String symbol) {
        return Optional.ofNullable(stringToEnum.get(symbol));
    }

    public static void main(String[] args) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        for (Operation op : Operation.values())
            System.out.printf("%f %s %f = %f%n",
                    x, op, y, op.apply(x, y));
    }
}
```
Strategy enum: Move the overtime pay computation into a nested enum and to pass an instance of this strategy enum to the constructor for the ```PayrollDay``` enum.
```java
package effectivejava.chapter6.item34;

import static effectivejava.chapter6.item34.PayrollDay.PayType.*;

// The strategy enum pattern (Page 166)
enum PayrollDay {
    MONDAY(WEEKDAY), TUESDAY(WEEKDAY), WEDNESDAY(WEEKDAY),
    THURSDAY(WEEKDAY), FRIDAY(WEEKDAY),
    SATURDAY(WEEKEND), SUNDAY(WEEKEND);
// It forces you to choose an overtime pay strategy each time you add an enum constant
    private final PayType payType;

    PayrollDay(PayType payType) { this.payType = payType; }

    int pay(int minutesWorked, int payRate) {
        return payType.pay(minutesWorked, payRate);
    }

    // The strategy enum type
    enum PayType {
        WEEKDAY {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked <= MINS_PER_SHIFT ? 0 :
                        (minsWorked - MINS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked * payRate / 2;
            }
        };

        abstract int overtimePay(int mins, int payRate);
        private static final int MINS_PER_SHIFT = 8 * 60;

        int pay(int minsWorked, int payRate) {
            int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }

    public static void main(String[] args) {
        for (PayrollDay day : values())
            System.out.printf("%-10s%d%n", day, day.pay(8 * 60, 1));
    }
}

```
## Item 35: Use instance fields instead of ordinals
```java
package effectivejava.chapter6.item35;

// Enum with integer data stored in an instance field (Page 168)
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);

    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
}
```

## Item 36: Use ```EnumSet``` instead of bit fields
```java
package effectivejava.chapter6.item36;

import java.util.*;

// EnumSet - a modern replacement for bit fields (Page 170)
public class Text {
    public enum Style {BOLD, ITALIC, UNDERLINE, STRIKETHROUGH}

    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) {
        System.out.printf("Applying styles %s to text%n",
                Objects.requireNonNull(styles));
    }

    // Sample use
    public static void main(String[] args) {
        Text text = new Text();
        text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
    }
}
```

## Item 37: Use ```EnumMap``` instead of ordinal indexing

```java
package effectivejava.chapter6.item37;
import java.util.*;
import static java.util.stream.Collectors.groupingBy;
import static java.util.stream.Collectors.toSet;

// Using an EnumMap to associate data with an enum (Pages 171-3)

// Simplistic class representing a plant (Page 171)
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }

    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override public String toString() {
        return name;
    }

    public static void main(String[] args) {
        Plant[] garden = {
            new Plant("Basil",    LifeCycle.ANNUAL),
            new Plant("Carroway", LifeCycle.BIENNIAL),
            new Plant("Dill",     LifeCycle.ANNUAL),
            new Plant("Lavendar", LifeCycle.PERENNIAL),
            new Plant("Parsley",  LifeCycle.BIENNIAL),
            new Plant("Rosemary", LifeCycle.PERENNIAL)
        };

        // Using ordinal() to index into an array - DON'T DO THIS!  (Page 171)
        Set<Plant>[] plantsByLifeCycleArr =
                (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
        for (int i = 0; i < plantsByLifeCycleArr.length; i++)
            plantsByLifeCycleArr[i] = new HashSet<>();
        for (Plant p : garden)
            plantsByLifeCycleArr[p.lifeCycle.ordinal()].add(p);
        // Print the results
        for (int i = 0; i < plantsByLifeCycleArr.length; i++) {
            System.out.printf("%s: %s%n",
                    Plant.LifeCycle.values()[i], plantsByLifeCycleArr[i]);
        }

        // Using an EnumMap to associate data with an enum (Page 172)
        Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =
                new EnumMap<>(Plant.LifeCycle.class);
        for (Plant.LifeCycle lc : Plant.LifeCycle.values())
            plantsByLifeCycle.put(lc, new HashSet<>());
        for (Plant p : garden)
            plantsByLifeCycle.get(p.lifeCycle).add(p);
        System.out.println(plantsByLifeCycle);

        // Naive stream-based approach - unlikely to produce an EnumMap!  (Page 172)
        System.out.println(Arrays.stream(garden)
                .collect(groupingBy(p -> p.lifeCycle)));

        // Using a stream and an EnumMap to associate data with an enum (Page 173)
        System.out.println(Arrays.stream(garden)
                .collect(groupingBy(p -> p.lifeCycle,
                        () -> new EnumMap<>(LifeCycle.class), toSet())));
    }
}
```
The ```EnumMap``` version always makes a nested map for each plant lifecycle while the stream-based versions only make a nested map if the garden contains one or more plants with that lifecycle. So, for example, if the garden contains annuals and perennials but no biennials, the size of ```plantsByLifeCycle``` will be three in the ```EnumMap``` version and two in both of the stream-based versions.

### Multidimensional relationship
```java
package effectivejava.chapter6.item37;
import java.util.*;
import java.util.stream.Stream;

import static java.util.stream.Collectors.*;

// Using a nested EnumMap to associate data with enum pairs - (Pages 174-5)
public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);

//        // Adding a new phase (Page 175)
//        SOLID, LIQUID, GAS, PLASMA;
//        public enum Transition {
//            MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
//            BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
//            SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
//            IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);

        private final Phase from;
        private final Phase to;
        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // Initialize the phase transition map
        private static final Map<Phase, Map<Phase, Transition>>
                m = Stream.of(values()).collect(groupingBy(t -> t.from,
                () -> new EnumMap<>(Phase.class),
                toMap(t -> t.to, t -> t,
                        (x, y) -> y, () -> new EnumMap<>(Phase.class))));
        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }

    // Simple demo program - prints a sloppy table
    public static void main(String[] args) {
        for (Phase src : Phase.values()) {
            for (Phase dst : Phase.values()) {
                Transition transition = Transition.from(src, dst);
                if (transition != null)
                    System.out.printf("%s to %s : %s %n", src, dst, transition);
            }
        }
    }
}
```
## Item 38: Emulate extensible enums with interfaces
Enums cannot be extended. But we can emulate it by implementing arbitrary interfaces.


```java
package effectivejava.chapter6.item38;
import java.util.*;

// Emulated extensible enum (Pages 176-9)
public enum ExtendedOperation implements Operation {
    EXP("^") {
        public double apply(double x, double y) {
            return Math.pow(x, y);
        }
    },
    REMAINDER("%") {
        public double apply(double x, double y) {
            return x % y;
        }
    };
    private final String symbol;
    ExtendedOperation(String symbol) {
        this.symbol = symbol;
    }
    @Override public String toString() {
        return symbol;
    }

//    // Using an enum class object to represent a collection of extended enums (page 178)
//    public static void main(String[] args) {
//        double x = Double.parseDouble(args[0]);
//        double y = Double.parseDouble(args[1]);
//        test(ExtendedOperation.class, x, y);
//    }
//    private static <T extends Enum<T> & Operation> void test(
//            Class<T> opEnumType, double x, double y) {
//        for (Operation op : opEnumType.getEnumConstants())
//            System.out.printf("%f %s %f = %f%n",
//                    x, op, y, op.apply(x, y));
//    }

    // Using a collection instance to represent a collection of extended enums (page 178)
    public static void main(String[] args) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        test(Arrays.asList(ExtendedOperation.values()), x, y);
    }
    private static void test(Collection<? extends Operation> opSet,
                             double x, double y) {
        for (Operation op : opSet)
            System.out.printf("%f %s %f = %f%n",
                    x, op, y, op.apply(x, y));
    }
}
```

## Item 39: Prefer annotations to naming patterns
### Test annotation
```java
package effectivejava.chapter6.item39.markerannotation;
import java.lang.annotation.*;

// Marker annotation type declaration - Page 180
import java.lang.annotation.*;

// Marker annotation type declaration (Page 180)

/**
 * Indicates that the annotated method is a test method.
 * Use only on parameterless static methods.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
}
```
```java
package effectivejava.chapter6.item39.markerannotation;

// Program to process marker annotations (Page 182)

import java.lang.reflect.*;

public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("Invalid @Test: " + m);
                }
            }
        }
        System.out.printf("Passed: %d, Failed: %d%n",
                passed, tests - passed);
    }
}
```
```java
package effectivejava.chapter6.item39.markerannotation;

// Program containing marker annotations (Page 181)
public class Sample {
    @Test
    public static void m1() { }        // Test should pass
    public static void m2() { }
    @Test public static void m3() {    // Test should fail
        throw new RuntimeException("Boom");
    }
    public static void m4() { }  // Not a test
    @Test public void m5() { }   // INVALID USE: nonstatic method
    public static void m6() { }
    @Test public static void m7() {    // Test should fail
        throw new RuntimeException("Crash");
    }
    public static void m8() { }
}
```
### Annotation with a parameter
```java
package effectivejava.chapter6.item39.annotationwithparameter;

// Annotation type with a parameter (Page 183)
import java.lang.annotation.*;
/**
 * Indicates that the annotated method is a test method that
 * must throw the designated exception to succeed.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}
```
```java
package effectivejava.chapter6.item39.annotationwithparameter;

import effectivejava.chapter6.item39.markerannotation.Test;
import java.lang.reflect.*;

// Program to process marker annotations and annotations with a parameter (Page 184)
public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("Invalid @Test: " + m);
                }
            }

            if (m.isAnnotationPresent(ExceptionTest.class)) {
                tests++;
                try {
                    m.invoke(null);
                    System.out.printf("Test %s failed: no exception%n", m);
                } catch (InvocationTargetException wrappedEx) {
                    Throwable exc = wrappedEx.getCause();
                    Class<? extends Throwable> excType =
                            m.getAnnotation(ExceptionTest.class).value();
                    if (excType.isInstance(exc)) {
                        passed++;
                    } else {
                        System.out.printf(
                                "Test %s failed: expected %s, got %s%n",
                                m, excType.getName(), exc);
                    }
                } catch (Exception exc) {
                    System.out.println("Invalid @ExceptionTest: " + m);
                }
            }
        }

        System.out.printf("Passed: %d, Failed: %d%n",
                passed, tests - passed);
    }
}
```
```java
package effectivejava.chapter6.item39.annotationwithparameter;
import effectivejava.chapter6.item39.annotationwithparameter.ExceptionTest;

import java.util.*;

// Program containing annotations with a parameter (Page 183)
public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() {  // Test should pass
        int i = 0;
        i = i / i;
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m2() {  // Should fail (wrong exception)
        int[] a = new int[0];
        int i = a[1];
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m3() { }  // Should fail (no exception)
}
```
### Repeatable annotations
```java
package effectivejava.chapter6.item39.repeatableannotation;

import java.lang.annotation.*;

// Repeatable annotation type (Page 186)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(ExceptionTestContainer.class)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}
```
```java
package effectivejava.chapter6.item39.repeatableannotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

//Container annotation for the repeatable ExceptionTest annotation (Page 186)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestContainer {
    ExceptionTest[] value();
}
```
```java
package effectivejava.chapter6.item39.repeatableannotation;

import effectivejava.chapter6.item39.markerannotation.Test;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

// Program to process marker annotations and repeatable annotations (Page 187)
public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("INVALID @Test: " + m);
                }
            }

            // Processing repeatable annotations (Page 187)
            if (m.isAnnotationPresent(ExceptionTest.class)
                    || m.isAnnotationPresent(ExceptionTestContainer.class)) {
                tests++;
                try {
                    m.invoke(null);
                    System.out.printf("Test %s failed: no exception%n", m);
                } catch (Throwable wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    int oldPassed = passed;
                    ExceptionTest[] excTests =
                            m.getAnnotationsByType(ExceptionTest.class);
                    for (ExceptionTest excTest : excTests) {
                        if (excTest.value().isInstance(exc)) {
                            passed++;
                            break;
                        }
                    }
                    if (passed == oldPassed)
                        System.out.printf("Test %s failed: %s %n", m, exc);
                }
            }
        }
        System.out.printf("Passed: %d, Failed: %d%n",
                          passed, tests - passed);
    }
}
```
```java
package effectivejava.chapter6.item39.repeatableannotation;

import java.util.ArrayList;
import java.util.List;

// Program containing repeatable annotations (Page 186)
public class Sample4 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() {  // Test should pass
        int i = 0;
        i = i / i;
    }

    @ExceptionTest(ArithmeticException.class)
    public static void m2() {  // Should fail (wrong exception)
        int[] a = new int[0];
        int i = a[1];
    }

    @ExceptionTest(ArithmeticException.class)
    public static void m3() { }  // Should fail (no exception)

    // Code containing a repeated annotation (Page 186)
    @ExceptionTest(IndexOutOfBoundsException.class)
    @ExceptionTest(NullPointerException.class)
    public static void doublyBad() {
        List<String> list = new ArrayList<>();

        // The spec permits this staticfactory to throw either
        // IndexOutOfBoundsException or NullPointerException
        list.addAll(5, null);
    }
}
/*
Test public static void effectivejava.chapter6.item39.repeatableannotation.Sample4.m2() failed: java.lang.ArrayIndexOutOfBoundsException: Index 1 out of bounds for length 0 
Test public static void effectivejava.chapter6.item39.repeatableannotation.Sample4.m3() failed: no exception
Passed: 2, Failed: 2
*/
```

## Item 40: Consistently use the Override annotation
```java
package effectivejava.chapter6.item40;
import java.util.*;

// Can you spot the bug? (Page 188)
public class Bigram {
    private final char first;
    private final char second;

    public Bigram(char first, char second) {
        this.first  = first;
        this.second = second;
    }

    public boolean equals(Bigram b) {
        return b.first == first && b.second == second;
    }

    public int hashCode() {
        return 31 * first + second;
    }

    public static void main(String[] args) {
        Set<Bigram> s = new HashSet<>();
        for (int i = 0; i < 10; i++)
            for (char ch = 'a'; ch <= 'z'; ch++)
                s.add(new Bigram(ch, ch));
        System.out.println(s.size()); // Prints 260
    }
}
```
To override ```Object.equals```, you must define an ```equals``` method whose parameter is of type ```Object```, but the parameter of ```Bigram```'s ```equals``` method is not of type ```Object```, so ```Bigram``` inherits the ```equals``` method from ```Object```.

Correct version:
```java
package effectivejava.chapter6.item40;

import java.util.HashSet;
import java.util.Set;

// Fixed Bigram class (Page 189)
public class Bigram2 {
    private final char first;
    private final char second;

    public Bigram2(char first, char second) {
        this.first  = first;
        this.second = second;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof Bigram2))
            return false;
        Bigram2 b = (Bigram2) o;
        return b.first == first && b.second == second;
    }

    public int hashCode() {
        return 31 * first + second;
    }

    public static void main(String[] args) {
        Set<Bigram2> s = new HashSet<>();
        for (int i = 0; i < 10; i++)
            for (char ch = 'a'; ch <= 'z'; ch++)
                s.add(new Bigram2(ch, ch));
        System.out.println(s.size()); // Prints 26
    }
}
```

## Item 41: Use marker interfaces to define types
A marker interface is an interface that contains no method declarations but merely designates a class that implements the interface ass having some property.

Marker interfaces define a type that is implemented by instances of the marked class; marker annotations do not. The existence of a marker interface type allows you to catch errors at compile time that you couldn't catch until runtime if you used a marker annotation.

