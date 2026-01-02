# Java 21 Migration & Refactoring Proposal for JTexGen

## Migration Status

✅ **Completed**: Project successfully migrated to Java 21
- Updated `maven-compiler-plugin` to version 3.13.0
- Changed compiler configuration to use `<release>21</release>`
- Project compiles successfully with 1 deprecation warning

## Proposed Refactorings

The following refactorings leverage modern Java features (Java 8-21) to improve code clarity, safety, and maintainability while preserving the library's functional design.

---

### 1. **Use Diamond Operator for Generic Instantiation** (Java 7)

**Priority**: Low | **Impact**: Code Clarity | **Effort**: Trivial

**Current Code** (src/main/java/org/fluttercode/texgen/utils/ColorGradient.java:20):
```java
private final List<RGBAColor> colors = new ArrayList<RGBAColor>();
```

**Refactored**:
```java
private final List<RGBAColor> colors = new ArrayList<>();
```

**Files to Update**: All files with generic instantiation (search for `new ArrayList<`, `new HashMap<`, etc.)

---

### 2. **Replace StringBuilder with Modern String Operations** (Java 15+)

**Priority**: Low | **Impact**: Code Clarity | **Effort**: Trivial

**Current Code** (src/main/java/org/fluttercode/texgen/utils/RGBAColor.java:137-140):
```java
public String toString() {
    return new StringBuilder().append("RGB Color : ").append("Red = ")
            .append(red).append(", Green = ").append(green).append(
                    ", Blue = ").append(blue).append(", Alpha= ").append(
                    alpha).toString();
}
```

**Refactored**:
```java
public String toString() {
    return "RGB Color : Red = " + red + ", Green = " + green +
           ", Blue = " + blue + ", Alpha= " + alpha;
}
```

Or using String.format for better readability:
```java
public String toString() {
    return String.format("RGB Color : Red = %d, Green = %d, Blue = %d, Alpha= %.2f",
                         red, green, blue, alpha);
}
```

---

### 3. **Use `var` for Local Variables** (Java 10)

**Priority**: Medium | **Impact**: Code Clarity | **Effort**: Low

**Benefits**: Reduces verbosity while maintaining type safety

**Current Code**:
```java
ColorGradient result = new ColorGradient();
RGBAColor res = new RGBAColor();
```

**Refactored**:
```java
var result = new ColorGradient();
var res = new RGBAColor();
```

**Recommendation**: Apply selectively where the type is obvious from the right-hand side. Avoid for method returns or where it reduces clarity.

---

### 4. **Replace Wrapper Constructors with valueOf** (Java 9+)

**Priority**: High | **Impact**: Performance & Deprecation | **Effort**: Low

**Current Code** (src/main/java/org/fluttercode/texgen/textures/demo/CloudTexture.java:71):
```java
new Double(value)  // Deprecated in Java 9, marked for removal
```

**Refactored**:
```java
Double.valueOf(value)
// Or simply use the primitive directly if possible
```

**Files to Update**: Search for `new Integer(`, `new Double(`, `new Long(` etc.

---

### 5. **Add `Objects.requireNonNull` for Null Validation** (Java 7+)

**Priority**: High | **Impact**: Code Clarity & NPE Prevention | **Effort**: Low

**Current Code** (src/main/java/org/fluttercode/texgen/textures/ChainedChannelSignal.java:24-28):
```java
public ChainedChannelSignal(ChannelSignal source) {
    super();
    if (source == null) {
        throw new IllegalArgumentException(
                "Source signal in a chained channel signal cannot be null");
    }
    this.source = source;
}
```

**Refactored**:
```java
public ChainedChannelSignal(ChannelSignal source) {
    super();
    this.source = Objects.requireNonNull(source,
        "Source signal in a chained channel signal cannot be null");
}
```

---

### 6. **Consider Sealed Interfaces** (Java 17)

**Priority**: Low | **Impact**: Type Safety & Documentation | **Effort**: Medium

**Benefit**: Makes the type hierarchy explicit and enables exhaustive pattern matching

**Current Code** (src/main/java/org/fluttercode/texgen/textures/Texture.java):
```java
public interface Texture {
    RGBAColor getColor(double u, double v);
    void getColor(double u, double v, RGBAColor value);
}
```

**Refactored** (if desired to restrict implementations):
```java
public sealed interface Texture
    permits AbstractTexture, /* other direct implementers */ {
    RGBAColor getColor(double u, double v);
    void getColor(double u, double v, RGBAColor value);
}
```

**Consideration**: Only apply if you want to restrict subclassing. For a library, this may be too restrictive. Recommended for `ChannelSignal.Channel` enum alternatives if needed.

---

### 7. **Use Switch Expressions for Enum Handling** (Java 14)

**Priority**: Medium | **Impact**: Code Clarity | **Effort**: Low

**Potential Use Case**: If there's code that switches on `ChannelSignal.Channel` enum:

**Current Pattern**:
```java
String channelName;
switch (channel) {
    case RED:
        channelName = "Red";
        break;
    case GREEN:
        channelName = "Green";
        break;
    case BLUE:
        channelName = "Blue";
        break;
    case ALPHA:
        channelName = "Alpha";
        break;
    default:
        channelName = "Unknown";
}
```

**Refactored**:
```java
String channelName = switch (channel) {
    case RED -> "Red";
    case GREEN -> "Green";
    case BLUE -> "Blue";
    case ALPHA -> "Alpha";
};
```

---

### 8. **Static Factory Methods with Better Names** (Best Practice)

**Priority**: Low | **Impact**: API Clarity | **Effort**: Low

**Current Code** (src/main/java/org/fluttercode/texgen/utils/ColorGradient.java):
```java
public static ColorGradient buildSpectrum() { ... }
public static ColorGradient buildFire() { ... }
```

**Refactored** (more modern naming):
```java
public static ColorGradient spectrum() { ... }
public static ColorGradient fire() { ... }
```

**Note**: This is a breaking API change. Only implement if major version bump is acceptable.

---

### 9. **Use Method References and Functional Interfaces** (Java 8)

**Priority**: Medium | **Impact**: Modern Java Idioms | **Effort**: Medium

**Opportunity**: Add functional interfaces for texture composition

**Example Addition**:
```java
@FunctionalInterface
public interface TextureTransform {
    Texture apply(Texture source);
}

// Usage
public class TextureBuilder {
    public Texture transform(Texture source, TextureTransform... transforms) {
        var result = source;
        for (var transform : transforms) {
            result = transform.apply(result);
        }
        return result;
    }
}

// Client code becomes more fluent:
var texture = builder.transform(
    baseTexture,
    t -> new UVRotate(t, 45),
    t -> new UVScale(t, 2.0)
);
```

---

### 10. **Immutability Improvements for Value Objects**

**Priority**: Medium | **Impact**: Thread Safety & Correctness | **Effort**: High

**Challenge**: `RGBAColor` is currently mutable (has setters and mutation methods)

**Current Design** (performance-oriented):
```java
public class RGBAColor {
    private int red, green, blue;
    private double alpha;

    public void setRed(int red) { this.red = red; }
    public void interpolate(RGBAColor color, double level) { /* mutates this */ }
    public void merge(RGBAColor color) { /* mutates this */ }
}
```

**Option A**: Keep current design (optimal for performance)
- Textures frequently reuse `RGBAColor` objects to avoid allocations
- Mutation is intentional for performance in tight rendering loops

**Option B**: Make RGBAColor truly immutable
```java
public record RGBAColor(int red, int green, int blue, double alpha) {
    public RGBAColor interpolate(RGBAColor color, double level) {
        return new RGBAColor(
            (int) Gradient.cosInterpolate(red, color.red, level),
            (int) Gradient.cosInterpolate(green, color.green, level),
            (int) Gradient.cosInterpolate(blue, color.blue, level),
            Gradient.cosInterpolate(alpha, color.alpha, level)
        );
    }
    // ... other methods return new instances
}
```

**Recommendation**:
- Keep current mutable design for performance-critical paths
- Add immutable alternatives (e.g., `ImmutableRGBAColor` record) for thread-safe use cases
- Consider adding `@NotThreadSafe` annotation to document the design choice

---

### 11. **Enhanced Documentation with Modern Javadoc** (Java 9+)

**Priority**: Low | **Impact**: Documentation Quality | **Effort**: Low

**Current Javadoc**:
```java
/**
 *
 * @param u
 * u Component used to determine the output signal value
 * @param v
 * v Component used to determine the output signal value
 * @return
 * the value of the output signal
 */
double getValue(double u, double v);
```

**Refactored** (improved formatting):
```java
/**
 * Calculates the signal value at the specified UV coordinates.
 *
 * @param u the U component (horizontal) in range [0, 1]
 * @param v the V component (vertical) in range [0, 1]
 * @return the signal value in range [0, 1]
 */
double getValue(double u, double v);
```

---

### 12. **Use `final` Keyword Consistently**

**Priority**: Medium | **Impact**: Immutability & Intent | **Effort**: Low

**Current Code** (many constructor parameters and local variables):
```java
public SignalScale(ChannelSignal source, double scale) {
    super(source, new ConstantSignal(scale));
}
```

**Refactored**:
```java
public SignalScale(final ChannelSignal source, final double scale) {
    super(source, new ConstantSignal(scale));
}
```

**Recommendation**: Use `final` for:
- All method parameters (prevents accidental reassignment)
- All fields that aren't mutated
- Local variables where appropriate

---

### 13. **Use Math.clamp() for Value Clamping** (Java 21)

**Priority**: High | **Impact**: Code Simplification | **Effort**: Trivial

**Current Code** (src/main/java/org/fluttercode/texgen/utils/RGBAColor.java:127-133):
```java
protected int clampValue(int value) {
    if (value > 255) {
        value = 255;
    }
    if (value < 0) {
        value = 0;
    }
    return value;
}
```

**Refactored**:
```java
protected int clampValue(int value) {
    return Math.clamp(value, 0, 255);
}
```

**Also applies to**: `GraphUtils.clamp()` and similar methods throughout the codebase.

---

### 14. **Enhanced Type Safety with Explicit Generics**

**Priority**: Low | **Impact**: Type Safety | **Effort**: Low

**Ensure all collections use explicit type parameters**:
```java
List colors = new ArrayList();  // Avoid raw types
```

Should be:
```java
List<RGBAColor> colors = new ArrayList<>();
```

**Recommendation**: Enable compiler warning `-Xlint:unchecked` and fix all warnings.

---

### 15. **Optional for Nullable Returns** (Java 8)

**Priority**: Low-Medium | **Impact**: API Clarity | **Effort**: Medium

**Consideration**: Methods that may return null could use Optional

**Current Pattern**:
```java
protected RGBAColor calculateColorFromTexture(double u, double v, Texture texture) {
    if (texture != null) {
        return texture.getColor(u, v);
    }
    return new RGBAColor(0, 0, 0, 1);  // Default black
}
```

**Alternative** (if nulls were returned):
```java
protected Optional<RGBAColor> calculateColorFromTexture(double u, double v, Texture texture) {
    return Optional.ofNullable(texture)
                   .map(t -> t.getColor(u, v));
}
```

**Recommendation**: Current code doesn't return nulls (returns black default), so Optional is not necessary. Keep current pattern.

---

## Implementation Priority

### High Priority (Immediate)
1. ✅ Maven compiler plugin update
2. Replace deprecated wrapper constructors (`new Double()` → `Double.valueOf()`)
3. Use `Math.clamp()` for clamping operations
4. Add `Objects.requireNonNull()` for null checks

### Medium Priority (Next Phase)
5. Apply `var` for obvious local variables
6. Use switch expressions where applicable
7. Make fields `final` where appropriate
8. Consistent use of diamond operator

### Low Priority (Future Enhancement)
9. Refactor toString() methods
10. Improve Javadoc formatting
11. Consider sealed interfaces for closed hierarchies
12. Evaluate immutability improvements

---

## Breaking Changes to Avoid

The following would require major version bump:
- Converting `RGBAColor` to a record (changes mutability contract)
- Sealing public interfaces like `Texture` or `ChannelSignal`
- Renaming static factory methods
- Changing method signatures to use `Optional`

---

## Testing Recommendations

Since there are no existing tests:
1. Create basic smoke tests for each texture type
2. Add tests for edge cases (UV values outside [0,1], null handling)
3. Performance benchmarks for hot paths (color calculation, signal processing)
4. Visual regression tests for complex textures

---

## Build Configuration Enhancements

Consider adding to `pom.xml`:
```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>21</maven.compiler.release>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <release>21</release>
                <compilerArgs>
                    <arg>-Xlint:all</arg>
                    <arg>-Xlint:-serial</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

This enables all compiler warnings except serialization warnings.

---

## Conclusion

The migration to Java 21 is complete and successful. The proposed refactorings are categorized by priority and impact. I recommend implementing the high-priority items first, as they provide immediate value with minimal risk. The medium and low-priority items can be implemented gradually over subsequent releases.

The library's functional, immutable-by-design architecture is already well-suited to modern Java practices. Most improvements are syntactic sugar that improve readability without changing the fundamental design.
