# JSON Style Guides

### Use double quotes

### Array types should have plural property names. All other property names should be singular.

### If value is empty or null then removing field is recommended

### Property names must be camel-cased, ascii strings

```json
{
  "thisPropertyIsAnIdentifier": "identifier value"
}
```

### Enum values should be represented as strings.

```java
public enum Color {
  WHITE,
  BLACK
}
```

```json
{
  "color": "WHITE"
}
```


### Dates: RFC 3339

```json
{
  "lastUpdate": "2007-11-06T16:34:41.000Z"
}
```

### Time duration: ISO 8601

```json
{
  // three years, six months, four days, twelve hours,
  // thirty minutes, and five seconds
  "duration": "P3Y6M4DT12H30M5S"
}
```

### Latitude/Longitude: ISO 6709

```json
{
  // The latitude/longitude location of the statue of liberty.
  "statueOfLiberty": "+40.6894-074.0447"
}
```

### Recommend Format

JSON response should contain either a `data` object **OR** an `error` object

```json
{ 
	"apiVersion": "1.0",
	"data": {
	}
}
```

```json
{ 
    "apiVersion": "1.0",
    "error": {
      "code": 404,
      "message": "msg..."
    }
}
```
