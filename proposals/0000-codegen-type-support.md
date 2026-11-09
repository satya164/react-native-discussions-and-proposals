---
title: Improve Codegen type support
author:
  - Satyajit Sahoo
date: 2026-10-30
---

# RFC0000: Improve Codegen type support

## Summary

When we write native components or TurboModules that use React Native Codegen, we often need a separate type and wrapper for the public API, along with extra boilerplate to work around limitations in the spec.

> [!NOTE]
> The behavior described here is based on React Native and `@react-native/codegen` 0.88.1.

## Motivation

For complex components, wrappers and extra boilerplate make sense. But for simpler APIs, we should be able to describe the public contract in the native spec without a separate wrapper.

## Detailed design

The examples were tested with TypeScript specs. We also need to test each change with Flow specs.

### Literal unions require a default for native components

#### Current behavior

It's not possible to use required or optional literal unions without a default. Consider we want the following API for props:

```ts
interface NativeProps extends ViewProps {
  align: 'start' | 'end';
  placement?: 'top' | 'right' | 'bottom' | 'left';
}
```

We need to mark all props as optional and provide a default using `CodegenTypes.WithDefault`:

```ts
interface NativeProps extends ViewProps {
  align?: CodegenTypes.WithDefault<'start' | 'end', 'start'>;
  level?: CodegenTypes.WithDefault<1 | 2 | 3, 1>;
}
```

We can work around this by using a wrapper that uses our desired types while using the spec for the native component internally, but we now need to maintain almost the same types twice, and provide default values even if they are never actually used, and may not make sense.

#### Proposed behavior

- Preserve whether a union prop is required without forcing it to be optional. The generated code could throw an error if it's absent at runtime.
- Don't require a default for optional unions. Provide a native representation for absence in optional props without a default.

### Prop absence is not preserved for native components

#### Current behavior

If we have an optional prop without a default, it's not possible to detect when the prop is absent on the native side.

For example, if we have the following props:

```ts
interface NativeProps extends ViewProps {
  angle?: CodegenTypes.Float;
  count?: CodegenTypes.Int32;
  enabled?: boolean;
}
```

We get the following C++ fields for the optional props:

```cpp
Float angle{0.0};
int count{0};
bool enabled{false};
```

From these C++ fields, we can't distinguish absence from an explicit zero or `false`.

We can generate optional fields with `generateOptionalProperties` and `generateOptionalObjectProperties`:

```ts
interface NativeProps extends ViewProps {
  angle?: CodegenTypes.WithDefault<CodegenTypes.Float, null>;
  enabled?: CodegenTypes.WithDefault<boolean, null>;
  label?: string;
  config?: {
    ratio?: CodegenTypes.WithDefault<CodegenTypes.Float, null>;
  };
}

export default codegenNativeComponent<NativeProps>('NativeExample', {
  generateOptionalProperties: true,
  generateOptionalObjectProperties: true,
});
```

In C++, we get optional values for `angle`, `enabled`, `label`, `config`, and `config.ratio`.

However:

- The inline options object fails type checking on React Native 0.88.1. Neither option is declared in `NativeComponentOptions`.
- On updates, an omitted prop can retain its previous value. From an optional native field, we can't tell whether the prop appeared in every React render.
- For types without optional native support, we still need a separate presence prop or an unused value in the public range to represent absence.

#### Proposed behavior

- Expose supported generation options in the public TypeScript types.
- Don't require `CodegenTypes.WithDefault<..., null>` to represent optional props.
- Document which prop kinds support optional native values.
- Define what native code receives when a prop is absent, and whether omitting it from an update retains or resets its value.

### Direct objects and literal unions are not supported in command arguments

#### Current behavior

If we want a command that accepts an options object or a literal union, we can't describe those parameters directly in the spec:

```ts
interface NativeCommands {
  scrollTo(
    viewRef: React.ComponentRef<HostComponent<NativeProps>>,
    options: { offset: CodegenTypes.Float },
    edge: 'top' | 'bottom',
  ): void;
}
```

We get `Found TSTypeLiteral` for the object and `Found TSUnionType` for the union.

We need to flatten the object into positional arguments and use a string for the union. We can use booleans, strings, numeric types, `RootTag`, and supported arrays, but the types we use in the spec no longer describe our desired API.

We can use `Options[]` when `Options` is an object type alias:

```ts
type Options = Readonly<{ offset: CodegenTypes.Float }>;

interface NativeCommands {
  scrollTo(
    viewRef: React.ComponentRef<HostComponent<NativeProps>>,
    values: Options[],
  ): void;
}
```

However, we lose the field types on the native side:

- On Android, we get an untyped `ReadableArray`.
- On Objective-C, we get `const NSArray *`.

The equivalent inline type, `{offset: CodegenTypes.Float}[]`, fails.

#### Proposed behavior

- Handle equivalent inline and aliased declarations consistently.
- Generate typed access to object fields.
- Support direct objects and literal unions in command arguments.
- Report other unsupported parameter types during spec validation and show a supported alternative.

### React Native color types are not supported in TurboModule specs

#### Current behavior

It's not possible to use `ColorValue` or `ProcessedColorValue` in a TurboModule spec. We get `Unrecognized generic type` for both. Using a union such as `number | CodegenTypes.UnsafeObject` also fails parsing.

Color props are processed automatically. For module methods, we need to call `processColor` ourselves.

If we need both numeric and platform color values in an Android or Objective-C module, we can work around this by wrapping the processed color in an object with an untyped field:

```ts
export interface Spec extends TurboModule {
  setTint(color: { value: CodegenTypes.UnsafeMixed }): void;
}
```

```ts
function setTint(color: ColorValue) {
  const processedColor = processColor(color);

  if (processedColor == null) {
    throw new Error(`Invalid color value: ${String(color)}`);
  }

  NativeModule.setTint({ value: processedColor });
}
```

We then need to read `value` in native code and validate the representations we support.

- If we only need resolved numeric colors, we can accept `number` directly.
- For a C++-only module, we can use `UnsafeMixed` as described in the next section.

#### Proposed behavior

- Support React Native color types in TurboModule arguments without requiring an untyped object wrapper.
- Define how numeric colors and platform color references are represented on the native side.
- Define where color processing happens so it doesn't have to be repeated in each library's wrapper.

### UnsafeMixed has different native representations for TurboModules

#### Current behavior

If we use `Object`, `UnsafeObject`, or `UnsafeMixed` in an ordinary Android or Objective-C module spec, we get an untyped dictionary:

```java
public abstract void setTint(ReadableMap color);
```

```objc
- (void)setTint:(NSDictionary *)color;
```

In generated C++, the argument is also converted through `asObject()` when we use that ordinary module schema. So we can't pass a bare number through `UnsafeMixed` in this case. We need to wrap it in an object first.

With a module name or spec filename ending in `Cxx`:

- With `UnsafeMixed`, we get `MixedTypeAnnotation` in the schema and `jsi::Value` in generated C++. We can pass a bare number there.
- With `UnsafeObject`, we still get an object representation.

The generated types are the same in 0.86.0 and 0.88.1.

#### Proposed behavior

- Document which values we can pass through each untyped spec type for ordinary and C++-only modules.
- Use consistent names for public types that distinguish arbitrary values from dictionaries.
- If `UnsafeMixed` starts accepting primitives on Android and Objective-C, document the changes required in existing dictionary-based implementations.

### Unrecognized literal union values abort the process

#### Current behavior

If we use a literal union for a component prop, we get a native enum in generated C++. For a string literal union, the conversion includes an unconditional abort when the value isn't recognized:

```cpp
auto string = (std::string)value;
if (string == "start") {
  result = NativeExampleAlign::Start;
  return;
}
if (string == "end") {
  result = NativeExampleAlign::End;
  return;
}
abort();
```

Unknown values for string and numeric literal union props terminated the process with `SIGABRT` in debug and release checks of the generated conversion code.

For example, if we add a union member in a JavaScript update but the native build still has the old set of values, the app can terminate when we pass the new value. We need to use a string prop and validate it ourselves to control how unknown values are handled.

#### Proposed behavior

- Report the prop name and unrecognized value without aborting the process.
- Use the explicit default when the union prop has one.
- Preserve absence for an optional union prop without a default.
- Define how to handle an unknown value for a required union prop without a default.

### Optional native values are not generated for every prop type

#### Current behavior

Even with `generateOptionalProperties`, we don't get optional native values for every prop type. For example, if we use a null default for an integer or a literal union:

```ts
interface NativeProps extends ViewProps {
  count?: CodegenTypes.WithDefault<CodegenTypes.Int32, null>;
  align?: CodegenTypes.WithDefault<'start' | 'end', null>;
}
```

- For `count`, we still get `int count{0}`, so we can't distinguish absence from zero.
- For `align`, generation fails with `Cannot read properties of null (reading 'split')`.

For a numeric literal union with a null default, generation succeeds, but we get an initializer referring to a native enum member that doesn't exist:

```cpp
std::optional<NativeExampleValue> value{NativeExampleValue::Valuenull};
```

We also get the TypeScript option error described under prop absence if we pass `generateOptionalProperties` in an inline options object on 0.88.1.

#### Proposed behavior

- Generate optional native values consistently for supported prop types, including integers and literal unions.
- Generate an empty optional value when the default is null.
- Report unsupported combinations during validation, with the prop name and type, rather than failing during generation or producing invalid C++.

### Prop absence has different native APIs across platforms

#### Current behavior

If we have a required string prop, we get a C++ field with a default value:

```cpp
std::string label{};
int count{0};
```

For the same prop, we get a nullable setter argument on Android:

```java
void setLabel(T view, @Nullable String value);
void setCount(T view, int value);
```

We also get nullable Android setters for required object and color props. In C++, an empty color can be represented by `SharedColor`'s `UndefinedColor` state.

If we explicitly pass null through the generated Android delegate:

- A string setter receives null.
- An integer setter receives zero.

However, if we omit a prop on initial mount, the setter isn't guaranteed to be called with null. Only the supplied property entries are passed to the delegate in `ViewManager.updateProperties`.

We need platform-specific absence checks. Setter nullability alone doesn't tell us whether a prop is set on mount or reset on removal.

#### Proposed behavior

- Define how initial absence, explicit null, removal, and reset are handled for each supported prop type.
- Make the generated native APIs and documentation match that behavior across platforms.

### Array types have different support across specs

#### Current behavior

- `ReadonlyArray<T>` works in component props, but fails in event payloads and command arguments with an unsupported-type error.
- `T[]` works in all three places.
- `Array<T>` works in props and commands, but fails in event payloads.

The command parser has a branch for `ReadOnlyArray` with a capital `O`, which differs from TypeScript's standard `ReadonlyArray` spelling.

#### Proposed behavior

- Accept `T[]`, `Array<T>`, and `ReadonlyArray<T>` consistently in props, event payloads, and commands.
- Normalize them to the same array schema before checking the element type. Read-only annotations are part of the JavaScript type and don't need a separate native collection representation.

### ImageSourcePropType is not supported in component props

#### Current behavior

It's not possible to use `ImageSourcePropType` in a native component spec. We get `Unknown prop type for "source": "ImageSourcePropType"`.

We need to use `ImageSource` instead, even though both names are exported in the default 0.88.1 TypeScript API and `ImageSourcePropType` is an alias for `ImageSource`.

With `ImageSource`, we get `resolveAssetSource` in the generated view config:

- Registered numeric asset IDs are resolved before native delivery.
- Existing object and array sources pass through.

We can use `ImageSource` in the native spec and keep `ImageSourcePropType` in the public API.

#### Proposed behavior

- Accept `ImageSource` and `ImageSourcePropType` as names for the same reserved prop type.

### Mixed primitive unions fail during TurboModule generation

#### Current behavior

For a native cache, we may want to set an explicit capacity or let native code choose it:

```ts
export interface Spec extends TurboModule {
  setCacheSize(size: number | 'auto'): void;
}
```

- For Android and C++, generation fails with `Non-homogenous union member types`.
- For Objective-C, generation succeeds with an untyped parameter:

```objc
- (void)setCacheSize:(NSObject *)size;
```

We can encounter this during a native build because Codegen is a build step. The failure happens during generation, before native compilation. The iOS runtime behavior wasn't tested.

We need to normalize the public input to a single supported type in a wrapper to use the same API across these targets.

#### Proposed behavior

- Reject mixed primitive unions during spec validation on targets that don't support them.
- Include the method, parameter, union members, and unsupported target in the error.

### Object fields have different native types for TurboModules

#### Current behavior

If we use an inline object parameter in a TurboModule spec:

```ts
export interface Spec extends TurboModule {
  setOptions(options: { name: string; size?: CodegenTypes.Int32 }): void;
}
```

We get a typed struct reference on Objective-C++:

```objc
- (void)setOptions:(JS::NativeExample::SpecSetOptionsOptions &)options;
```

But we get an untyped map on Android:

```java
public abstract void setOptions(ReadableMap options);
```

In generated C++, the argument is converted through `args[0].asObject(rt)`. For this inline declaration, we don't get a struct containing its fields.

In a C++-only module, we can use a named object alias to generate struct and bridging helpers:

```ts
export type Options = {
  name: string;
  size?: CodegenTypes.Int32;
};

export interface Spec extends TurboModule {
  setOptions(options: Options): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeExampleCxx');
```

We can reuse the generated field conversions by choosing the C++ field types and registering a bridging specialization, as shown in the [custom C++ types guide](https://reactnative.dev/docs/the-new-architecture/custom-cxx-types):

```cpp
namespace facebook::react {

using Options = NativeExampleCxxOptions<std::string, std::optional<int32_t>>;

template <>
struct Bridging<Options> : NativeExampleCxxOptionsBridging<Options> {};

}
```

For a union of object types, we also lose the typed Objective-C++ struct:

```ts
export interface Spec extends TurboModule {
  setOptions(options: { name: string } | { size: CodegenTypes.Int32 }): void;
}
```

```objc
- (void)setOptions:(NSDictionary *)options;
```

On Android, we need to read and validate object fields ourselves. Typed accessors could replace those field reads while keeping the `ReadableMap` method parameter. Replacing the parameter with a new native type would also require changes to argument conversion.

#### Proposed behavior

- Generate typed field accessors for Android object parameters, including optional fields, while keeping `ReadableMap` method parameters.
- Support inline object declarations and named aliases consistently in C++-only modules, reusing the existing struct and bridging helpers.
- Document which object declarations generate C++ helpers and how to use them.
- Design typed access to object unions separately from accessors for a single object type.

### Method overloads produce invalid native code for TurboModules

#### Current behavior

If we have method overloads in a TurboModule spec, parsing and generation both succeed:

```ts
export interface Spec extends TurboModule {
  load(source: string): void;
  load(source: CodegenTypes.Int32): void;
}
```

However, the generated code fails native compilation:

- In C++, the same `__load` function is defined twice in the generated class.
- In the Android Java Native Interface (JNI) code, the same `__hostFunction_NativeExampleSpecJSI_load` function is defined twice. The Java declarations themselves are valid overloads.
- In Objective-C, the `load:` selector is declared with incompatible parameter types. The generated implementation also repeats its host function.

We need to use a single signature and normalize the public input in a wrapper, or use different method names.

#### Proposed behavior

- Reject duplicate method names during spec validation until overload dispatch is supported across targets.
- Point to both declarations in the error so we can fix the spec before compiling native code.

## Validation and sources

We checked the behavior in fresh local projects with pinned package versions. Relevant examples were recreated as minimal spec strings from the [original document](https://gist.github.com/satya164/0821124eae406fe5cf76115b853dacc5/30040dae8ad938ebfe30796680555748049bb808).

The checks covered:

- 54 parser cases per version, comparing 0.86.0 with 0.88.1. The parser and generator outcomes were the same. Successful schemas were passed to the component or module generators, with separate calls so a failure for one target couldn't hide another result.
- 30 native checks, including execution of generated enum conversion bodies and compilation of isolated generated C++, JNI, Objective-C protocol, and enum-member fragments. Small stand-ins were used for React Native dependencies, with real Foundation headers for the Objective-C checks. Matching valid controls passed.
- Six TypeScript checks, using strict checking with dependency declarations skipped through `skipLibCheck`.

All expected outcomes passed. Additional generator checks covered the `number | 'auto'` example in both versions and named C++-only object aliases in 0.88.1.

The validation has these limits:

- Full Android and iOS app builds, device tests, Fabric mount and reset behavior, native event dispatch, and over-the-air updates were not exercised.
- Asset resolution and the Android property update loop were checked in source.

The relevant 0.88.1 sources are:

- [Component prop parsing](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/parsers/typescript/components/componentsUtils.js).
- [Event parsing](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/parsers/typescript/components/events.js) and [command parsing](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/parsers/typescript/components/commands.js).
- [C++ component prop generation](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/generators/components/GeneratePropsH.js).
- [Shared module type parsing](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/parsers/parsers-primitives.js) and [C++-only module detection](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/parsers/utils.js).
- [C++ module generation](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/generators/modules/GenerateModuleH.js), [Android module generation](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/generators/modules/GenerateModuleJavaSpec.js), and [Android JNI generation](https://github.com/react/react-native/blob/v0.88.1/packages/react-native-codegen/src/generators/modules/GenerateModuleJniCpp.js).
- [Native component options](https://github.com/react/react-native/blob/v0.88.1/packages/react-native/Libraries/Utilities/codegenNativeComponent.js), [color processing](https://github.com/react/react-native/blob/v0.88.1/packages/react-native/Libraries/StyleSheet/processColor.js), and [SharedColor](https://github.com/react/react-native/blob/v0.88.1/packages/react-native/ReactCommon/react/renderer/graphics/Color.h).
- [Asset resolution](https://github.com/react/react-native/blob/v0.88.1/packages/react-native/Libraries/Image/resolveAssetSource.js) and [Android property updates](https://github.com/react/react-native/blob/v0.88.1/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/uimanager/ViewManager.java).

## Drawbacks

- For colors, optional values, and typed objects, we need changes to native conversions, generated APIs, and runtime behavior as well as parsing. We'd need to maintain the new native conversions and test each supported type on each target.
- Some fixes change existing native declarations. An integer becoming `std::optional<int>` would require changes in library implementations. Android object accessors can be added without changing the existing `ReadableMap` parameters.
- Using a fallback for unknown union values can hide a JavaScript/native version mismatch. Warn during development when JavaScript passes a value the native build doesn't recognize.
- We can address several gaps in wrappers today. Review each change separately. For some cases, keeping a wrapper may cost less than adding native support.

## Alternatives

TBD

## Adoption strategy

TBD

## How we teach this

TBD
