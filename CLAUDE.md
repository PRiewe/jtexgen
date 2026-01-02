# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JTexGen is a procedural texture generation library for Java that creates textures programmatically using UV coordinates, signals, and composable texture objects. Version 1.2, built with Maven, targeting Java 8.

## Build Commands

**Compile:**
```bash
mvn compile
```

**Package JAR:**
```bash
mvn package
```

**Clean and rebuild:**
```bash
mvn clean package
```

## Core Architecture

JTexGen uses a dual-abstraction system with two primary interfaces:

### 1. Core Interfaces

- **`Texture`** (`src/main/java/org/fluttercode/texgen/textures/Texture.java`): Main interface with `getColor(u, v)` method returning `RGBAColor` from normalized UV coordinates (0-1)
- **`ChannelSignal`** (`src/main/java/org/fluttercode/texgen/textures/ChannelSignal.java`): Generates scalar values (0-1) from UV coordinates via `getValue(u, v)`

### 2. Abstract Base Classes

- **`AbstractTexture`**: Base for all texture implementations. Provides helper methods and access to a shared `PerlinNoise` instance
- **`AbstractChannelSignal`**: Base for signal implementations. Provides noise access and value calculation helpers
- **`ChainedChannelSignal`**: For signal modifiers that transform other signals

### 3. Package Organization

**`org.fluttercode.texgen.signals`** - Signal implementations (scalar value generators):
- Core signals: `NoiseSignal`, `USignal`, `VSignal`, `RadialSignal`, `MarbleSignal`, `MandelbrotSignal`, `SineWave`, `LightSignal`
- **`signals.modifier`** subpackage: Signal transformers that wrap and modify other signals
  - Value modifiers: `SignalMultiply`, `SignalSum`, `SignalScale`, `SignalThreshold`, `InvertSignal`, `ClampSignal`, `NoisySignal`, `SignalBand`, `SignalPower`
  - UV transformers: `SignalUVRotate`, `SignalUVScale`, `SignalUVTranslate`

**`org.fluttercode.texgen.textures`** - Texture implementations organized by purpose:
- **`textures.color`**: Color-based textures (`SolidTexture`, `SignalTexture`, gradients, `ColorFilter`, `NegateTexture`, `Bars`, `LightTexture`, `ImageTexture`, `UVColor`)
- **`textures.pattern`**: Geometric patterns (`Checker`, `BrickTexture`, `GridPattern`, `Marble`, `Mandelbrot`, `Fur`, `RoundedCornerTexture`)
- **`textures.composite`**: Texture composition (`MergedTexture`, `MultiMergeTexture`, `MixTexture`, `TextureMixer`, `SignalMixer`)
- **`textures.signal`**: Signal-to-texture converters (`GradientSignalTexture`, `UVSignalTexture`, `Threshold`, `SmoothThreshold`, `AlphaSignal`)
- **`textures.uv`**: UV coordinate transformers (`UVRotate`, `UVScale`, `UVTranslate`, `UVNoise`, `UVNoiseTranslate`)
- **`textures.complex`**: Pre-built complex textures (`Flame`, `Fireball`, `ComplexMarble`, `Camouflage`, `DirtyBrick`, `Dirty`, `NoisyTexture`, `Polkadot`, `Map`, `MarbleChessboard`, `SunEclipse`)
- **`textures.demo`**: Example textures for demonstration purposes

**`org.fluttercode.texgen.utils`** - Utility classes:
- `RGBAColor`: Color representation with interpolation, merging, scaling, negation methods
- `PerlinNoise`: Perlin/fractal noise implementation (shared singleton via static field)
- `Gradient`: Mathematical interpolation functions (linear, cosine, smoothstep)
- `ColorGradient`: Color gradient interpolation with preset gradients (spectrum, fire, map, black-white)

**`org.fluttercode.texgen.gui`** - Visualization framework:
- `TextureViewer`: Simple API for viewing textures: `TextureViewer.show(texture)`
- `TextureWindow`: Main rendering window with controls for anti-aliasing and gradual rendering
- Supporting classes: `TexturePanel`, `TextureImage`, `RenderThread`, `RenderListener`

## Component Interaction Patterns

### Signal → Texture Conversion
```
ChannelSignal (scalar 0-1)
  → SignalTexture or GradientSignalTexture
  → Texture (RGBA color)
```

### Signal Modifier Chains
```
Source Signal
  → SignalModifier (scale/threshold/multiply/etc.)
  → SignalModifier (chain multiple)
  → Final Signal Value
```

### Texture Composition
```
Base Textures
  → UV Transformers (rotate/scale/translate coordinates)
  → Composite Textures (merge/mix multiple textures)
  → Final Rendered Texture
```

### Example: Complex Texture Construction
The `Flame` texture demonstrates typical composition:
```
MarbleSignal (uses Perlin noise)
  → GradientSignalTexture (applies fire color gradient)
  → Flame texture
```

## Design Patterns Used

- **Decorator Pattern**: Signal modifiers and UV transformers wrap other objects
- **Strategy Pattern**: Polymorphic texture implementations via `Texture` interface
- **Template Method**: `AbstractTexture`/`AbstractChannelSignal` provide common functionality
- **Composition over Inheritance**: Complex textures built from simpler signal and texture components
- **Immutability**: Most texture/signal classes are immutable and thread-safe
- **Shared Resources**: Single `PerlinNoise` instance shared across all textures/signals

## Key Architectural Principles

1. **UV Coordinate Based**: All textures and signals operate on normalized UV coordinates (0-1 range)
2. **Functional Composition**: Textures and signals compose together functionally without side effects
3. **Two-Level Abstraction**: Signals generate scalars, textures generate colors - they work complementarily
4. **Gradient System**: `ColorGradient` maps signal values (0-1) to color spectrums
5. **Transformation Pipeline**: UV transformers modify coordinates before texture sampling
6. **Thread-Safe**: Immutable design supports concurrent rendering
7. **Anti-Aliasing Support**: Built into rendering pipeline via `TextureWindow`

## Running and Viewing Textures

To view a texture, use the `TextureViewer` utility:

```java
public static void main(String[] args) {
    TextureViewer.show(new ComplexMarble());
}
```

This opens a window with Start/Stop controls and supports anti-aliasing and gradual rendering options.

## Common Development Patterns

### Creating a Custom Signal
Extend `AbstractChannelSignal` and implement `getValue(double u, double v)`:
```java
public class MySignal extends AbstractChannelSignal {
    public double getValue(double u, double v) {
        // Return value 0-1 based on u,v coordinates
        return Math.sin(u * Math.PI) * Math.cos(v * Math.PI);
    }
}
```

### Creating a Custom Texture
Extend `AbstractTexture` and implement `getColor(double u, double v)`:
```java
public class MyTexture extends AbstractTexture {
    public RGBAColor getColor(double u, double v) {
        // Return RGBAColor based on u,v coordinates
        return new RGBAColor(u, v, 0.5, 1.0);
    }
}
```

### Composing Textures
Combine existing components:
```java
ChannelSignal noise = new NoiseSignal(8, 10);
ChannelSignal scaled = new SignalScale(noise, 2.0);
Texture gradientTex = new GradientSignalTexture(scaled, ColorGradient.buildFire());
Texture rotated = new UVRotate(gradientTex, 45);
```

## Project Structure Notes

- No test suite currently exists in the repository
- Java 8 compatibility (configured in pom.xml)
- LGPL licensed (see lgpl.txt)
- Reference documentation available in `reference/` directory
