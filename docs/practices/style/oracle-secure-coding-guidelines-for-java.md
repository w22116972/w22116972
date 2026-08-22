# Oracle Secure Coding Guidelines for Java

## Table of Contents
- [Design `class`](#design-class)
  - [`class` is either `final` or `abstract`](#class-is-either-final-or-abstract)
  - [`class`, `interface` should be `package-private`](#class-interface-should-be-package-private-unless-it-is-part-of-a-published-api)
  - [Only use non-overridable methods in constructor](#only-use-non-overridable-methods-in-constructor-eg-final-static-or-private-methods)
  - [Fields of objects should be private](#fields-of-objects-should-be-private)
  - [If `class` or object is mutable](#if-class-or-objecteg-custom-list-is-mutable)
  - [Use setter for input validation](#use-setter-for-input-validation)
  - [If `class` is sensitive](#if-class-is-sensitive)
- [Input Validation](#input-validation)
  - [Use the created/returned object instead of original input](#use-the-createdreturned-object-instead-of-original-input-for-validation)
  - [Validate output from untrusted upcall](#validate-output-from-untrusted-upcall-invoking-a-method-of-higher-level-code)
  - [Parse and validate should be separate](#parse-and-validate-should-be-separate)
  - [Not use dynamic SQL query](#not-use-dynamic-sql-query)
  - [Don't rely on API for safety checking](#dont-rely-on-api-for-safety-checking)
  - [Use sanitized data for validation](#use-sanitized-data-for-validation-not-the-original-input)
- [Release resources](#release-resources)
  - [Use Lambda to implement Execute Around Method pattern](#use-lambda-to-implement-execute-around-method-pattern)
  - [Use try-with-resources](#use-try-with-resources)
  - [Use `finally` block](#if-no-lambda-and-no-try-with-resources-use-finally-block)
- [When catching exceptions](#when-catching-exceptions)
  - [Catch exceptions that might expose sensitive information](#catch-exceptions-that-might-expose-sensitive-information-and-rethrow-a-safer-more-general-exception)
- [Handling integer overflow](#handling-integer-overflow)
  - [Reorder the expression](#reorder-the-expression)
  - [Use `java.math.BigInteger`](#use-javamathbiginteger)
  - [Use `Math.???Exact(long value)`](#use-mathexactlong-value)
  - [Use `Objects.checkIndex` or `Objects.checkFromToIndex`](#use-objectscheckindex-or-objectscheckfromtoindex-to-check-array-index)
- [Handling float-point numbers](#handling-float-point-numbers)
- [Handling mutable data](#handling-mutable-data)
  - [Create copies of mutable data](#create-copies-of-mutable-data)
  - [`public static final` declaration](#public-static-final-should-be-declared-on-object-that-is-really-unmodifiable)
- [References](#references)

## Design `class`

#### `class` is either `final` or `abstract`

To prevent malicious subclasses from overriding methods(e.g. finalizers, cloning, etc.)

Limit the extensibility of classes and methods

note: why subclass of non-final class overrides `clone` is bad because cloning results are in a shallow copy, sharing referenced objects but having different fields and separate intrinsic locks.

#### `class`, `interface` should be `package-private` unless it is part of a published API

- method should be `package-private` or `private`

#### Only use non-overridable methods in constructor e.g. `final`, `static`, or `private` methods

To avoid attackers from overriding the method to perform malicious activities during object construction

#### Fields of objects should be private

```java
// Good 
class User {
    private String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }
}
```

#### if `class` or object(e.g. custom list) is mutable

To avoid being modified by the caller, provide a safe copy of the object. Unless it is desired to be shared.

Use 1 of the following ways to create safe copies of its instances
- static factory creation method
- copy constructor
- public copy method 

Passing mutable object to untrusted, use defensive copying that passing a copy of the object

#### use setter for input validation

```java
private String state;

public void setState(String input) {
    // validate input
  this.state = input;
}
```

### If `class` is sensitive

#### Prefer static factory methods + private constructor over public constructors

Pros
- return cached instances to avoid creating duplicate instances
- stop instances being created based on the result of safety checks
  - `new Object()` is always created
- public constructor is easily being overridden or inherited by subclasses

---

## Input Validation

### Use the created/returned object instead of original input for validation

Should always use the safe created object and not the original input. Because the original input may be unsafe.

```java
public class Main {
    public static void main(String[] args) {
        String unsafeInput = "hello<script>alert('xss')</script>";
        SafeString safeString = new SafeString(unsafeInput);
        
        System.out.println("Original input: " + unsafeInput);
        System.out.println("Sanitized value: " + safeString.getValue());
    }
}
```

### Validate output from untrusted upcall (invoking a method of higher level code)

```java
public class JsonParser {
    private ObjectMapper objectMapper;

    public JsonParser() {
        this.objectMapper = new ObjectMapper();
    }

    public MyData parse(String jsonData) throws JsonProcessingException {
        // validate input
        if (jsonData == null || jsonData.trim().isEmpty()) {
            throw new IllegalArgumentException("JSON data cannot be null or empty");
        }
        
        // invoking a method of higher level code
        MyData data = objectMapper.readValue(jsonData, MyData.class);

        // validate output from untrusted upcall
        if (data == null || data.getId() == null || data.getName() == null) {
            throw new IllegalStateException("Parsed data is invalid");
        }

        return data;
    }
}
```

### Parse and validate should be separate

#### e.g. To expect date format `YYYY-MM-DD`

```java
// Bad
// Fail on "2027-10/27"
public static boolean isValidDate(String input) {
  Pattern pattern = Pattern.compile("^(\\d{4})[-\\/](\\d{2})[-\\/](\\d{2})$");
  Matcher matcher = pattern.matcher(input);
  if (matcher.matches()) {
    try {
      int year = Integer.parseInt(matcher.group(1));
      int month = Integer.parseInt(matcher.group(2));
      int day = Integer.parseInt(matcher.group(3));
      
      // Will have too many logics
      if (year >= 1900 && year <= 2100 && month >= 1 && month <= 12 && day >= 1 && day <= 31) {
        return true;
      }
    } catch (NumberFormatException e) {
      // 處理解析錯誤
      return false;
    }
  }
  return false;
}
```

```java
// Good
public static LocalDate parseDate(String input) {
    try {
        // validate `yyyy-mm-dd`
        return LocalDate.parse(input, DateTimeFormatter.ISO_LOCAL_DATE);
    } catch (DateTimeParseException e1) {
        try {
            return LocalDate.parse(input, DateTimeFormatter.ofPattern("yyyy/MM/dd"));
        } catch (DateTimeParseException e2) {
            return null;
        }
    }
}
public static boolean isValidDate(LocalDate date) {
  if (date == null) {
    return false;
  }
  int year = date.getYear();
  int month = date.getMonthValue();
  int day = date.getDayOfMonth();
  
  return year >= 1900 && year <= 2100 && month >= 1 && month <= 12 && day >= 1 && day <= date.lengthOfMonth();
}
```

### Not use dynamic SQL query

```java
// Good
String sql = "SELECT * FROM User WHERE userId = ?";
PreparedStatement stmt = con.prepareStatement(sql);
stmt.setString(1, userId);
ResultSet rs = prepStmt.executeQuery();
```

### Don't rely on API for safety checking

- `new URL(url)` only does format checking, not safety checking
- `new File(path)` not checking if it contains `../`
  - On some platforms or legacy systems (especially in native code), `\0` may truncate the path (e.g., `"foo\0.txt"` becomes `"foo"`

```java
// Good
if (userProvidedFileName.contains("\0") || userProvidedFileName.contains("../")) {
    throw new SecurityException("Invalid filename");
}
Path basePath = Paths.get("user_files").toAbsolutePath().normalize();
Path filePath = basePath.resolve(userProvidedFileName).normalize();

// Check that the resolved path is still within the base directory
if (!filePath.startsWith(basePath)) {
    throw new SecurityException("Access denied");
}

if (Files.exists(filePath)) {
    // Read and return the file content...
}
```

### use sanitized data for validation not the original input

```java
public void handleFileUpload(String fileName, InputStream content) {
    String sanitizedName = FileNameUtils.sanitize(fileName);
    
    // Bad
    if (!isRestrictedFileName(fileName)) {
        saveFile(sanitizedName, content);
    }
    
    // Good
    if (!isRestrictedFileName(sanitizedName)) { 值
        saveFile(sanitizedName, content);
    }
}
```

---

## Release resources

#### use Lambda to implement **Execute Around Method** pattern

```java
long sum = readFileBuffered(InputStream in -> {
   long current = 0;
   for (;;) {
        int b = in.read();
        if (b == -1) {
            return current;
        }
        current += b;
    }
});
```

#### use try-with-resources 

```java
try (final InputStream in = Files.newInputStream(path)) {
    handler.handle(new BufferedInputStream(in));
}
```

#### if no Lambda and no try-with-resources, use `finally` block

```java
lock.lock();
try {
    return action.run();
} finally {
    lock.unlock();
}
```

---

## When catching exceptions 

- Handling if it can be fixed or recovered
  - e.g. if `FileNotFoundException` then catch it and load the default file
- Propagate if it cannot be fixed or recovered
  - e.g. if `OutOfMemoryError` then propagate it to the caller
  - is ok to re-wrap the exception to add more context information

### Catch exceptions that might expose sensitive information and rethrow a safer, more general exception

- e.g. `java.io.FileInputStream` can leak the file path in the exception message
- it is ok to print the exception message to the log but not to the user
  - unless it is super highly sensitive information like SSN, credit card number, etc.
- don't rely on `exception.message` for checking

#### How to possibly reduce the risk of exposing sensitive information in the memory

- store sensitive information in `char[]` instead of `String`
  - after using it, overwrite the `char[]` with empty characters
- store them in local scope variables
- only hold them when necessary, release them as soon as possible


---

## Handling integer overflow

#### Reorder the expression

From `a + b > max` to `b < 0 || a > max - b`
- check if `b` is negative because `max - b` could be overflow

#### use `java.math.BigInteger`

```java
import java.math.BigInteger;
public class BigIntegerOverflowCheck {
    public static void main(String[] args) {
        int a = Integer.MAX_VALUE;
        int b = 2;

        // Convert to BigInteger
        BigInteger bigA = BigInteger.valueOf(a);
        BigInteger bigB = BigInteger.valueOf(b);
        BigInteger result = bigA.multiply(bigB);

        // Check if result fits in int
        if (result.compareTo(BigInteger.valueOf(Integer.MIN_VALUE)) < 0 ||
                result.compareTo(BigInteger.valueOf(Integer.MAX_VALUE)) > 0) {
            System.out.println("Overflow would occur!");
        } else {
            System.out.println("Safe result: " + result.intValue());
        }
    }
}
```

#### use `Math.???Exact(long value)`

```java
public class AddExactOverflowCheck {
    public static void main(String[] args) {
        int a = Integer.MAX_VALUE;
        int b = 1;

        try {
            int result = Math.addExact(a, b); // This will throw
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Overflow occurred during addition: " + e.getMessage());
        }
    }
}

// This will throw if you try to negate Integer.MIN_VALUE (which has no positive counterpart in int)
public class NegateExactOverflowCheck {
    public static void main(String[] args) {
        int value = Integer.MIN_VALUE;

        try {
            int negated = Math.negateExact(value); // This will throw
            System.out.println("Negated value: " + negated);
        } catch (ArithmeticException e) {
            System.out.println("Overflow occurred during negation: " + e.getMessage());
        }
    }
}
```

#### use `Objects.checkIndex` or `Objects.checkFromToIndex` to check array index

- throws `IndexOutOfBoundsException` if the index is out of bounds
- List collection already has this check, only primitive arrays need this check

```java
public class StringListProcessor {

    private int[] data;

    public String getElementAt(int index) {
        Objects.checkIndex(index, data.length);
        return data[index];
    }

    public List<String> getSubList(int fromIndex, int toIndex) {
        Objects.checkFromToIndex(fromIndex, toIndex, data.length);
        return Arrays.copyOfRange(data, fromIndex, toIndex);
    }

    public List<String> getSubListWithSize(int fromIndex, int size) {
        Objects.checkFromIndexSize(fromIndex, size, data.length);
        int toIndex = fromIndex + size;
        return Arrays.copyOfRange(data, fromIndex, toIndex);
    }
}
```

## Handling float-point numbers

- When converting `NaN` to an integer, it becomes `0`
  - `0.0 / 0.0` and `Infinity - Infinity` are `NaN` in Java
- When converting `Postive INF` -> `Integer.MAX_VALUE`, `Negative INF` -> `Integer.MIN_VALUE`
- When comparing anything with `NaN`, the result is always `false`
  - Not good if using it as key in hash table

```java
// Good
if (Double.isNaN(untrusted_double_value)) {
    // specific action for non-number case
}
if (Double.isInfinite(untrusted_double_value)){
    // specific action for infinite case
}
// normal processing starts here
```

---

## Handling mutable data

### Create copies of mutable data

If a method returns a reference to an internal mutable object, client code can change the instance's internal state. To avoid this, unless you intend to share the state, copy the mutable object and return the copy.

```java
public class CopyOutput {
    private final java.util.Date date;
    
    public java.util.Date getDate() {
        return (java.util.Date)date.clone();
    }
}

public final class CopyMutableInput {
    private final Date date;
    
    // Constructors should complete the deep copy before assigning values to a field
    public CopyMutableInput(Date date) {
        this.date = new Date(date.getTime());
    }
}
```

For collection, if the elements are mutable, then **deep copy**.

```java
// Date is mutable.
public void deepCopy(Collection<Date> dates) {
    Collection<Date> datesCopy = new ArrayList<>(dates.size());
    for (Date date : dates) {
        datesCopy.add(new java.util.Date(date.getTime()));
    }
    doLogic(datesCopy);
}
```

For collection, if the elements are immutable, then shallow copy (can be modified independently) or read-only view.

Use `Collections.unmodifiableList` to create a read-only view of the list, ensuring that any changes to the original list will be reflected in the unmodifiable list. However, the unmodifiable list itself cannot be modified.
This approach is particularly suitable for use in getter methods.

```java
public List<String> getItems() {
    return Collections.unmodifiableList(items);
}

// Shallow copy
public void shallowCopy(Collection<String> strs) {
    strs = new ArrayList<>(strs);
    doLogic(strs);
}
```


When designing a mutable value class, provide one of the following ways to create safe copies of its instances
```java
public class MutableValue {
    // 1. Constructors for creating safe copies
    public MutableValue(List<String> items) {
        this.items = new ArrayList<>(items);
    }
    
    public MutableValue(MutableValue other) {
        this.items = new ArrayList<>(other.items);
    }

    // 2. Static methods for creating safe copies
    public static MutableValue create(List<String> items) {
        return new MutableValue(items);
    }
    
    // 3. Public copy method
    public MutableValue copy() {
        return new MutableValue(new ArrayList<>(this.items));
    }
}
```

### `public static final` should be declared on object that is really unmodifiable

- should not declare on list collection
  - or use `asList`, `List.of` to create unmodifiable list

---

# References

> https://www.oracle.com/java/technologies/javase/seccodeguide.html
`