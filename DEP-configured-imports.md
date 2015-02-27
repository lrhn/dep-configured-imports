# Dart Configurations/Configured Imports

## Contact information

Name: Lasse R.H. Nielsen  
E-mail: lrn@google.com  
[DEP Proposal Location][]  
Further stakeholders:  
- Bob Nystrom - rnystrom@google.com
- Lars Bak - bak@google.com

## Summary

Allow imports to depend on the runtime environment or user configuration.

## Motivation

Problems to solve:

* Allow libraries to choose between platform dependent libraries depending on
  the platform it is running on. Example: The http package wants to use an
  implementation based on either "dart:html" or "dart:io", depending on which
  is available.

* Allow libraries to choose between different (platform
  independent) implementations based on a compile-time/deployment-time
  configuration setting, without importing both implementations. Example
  (speculative): A logging package can have different logging strategies or
  formatters, and a deployment will only want to include the ones that it
  actually uses.

The former problem is a blocker for having libraries that work on multiple 
platforms, but which depend on platform-specific implementation for each platform.

## Proposal

### Syntax

Example:

```dart
import dart.platform == "browser"    : "some uri.dart"
    || dart.platform == "standalone" : "other uri.dart"
    || "default uri.dart"
    deferred as foo show x hide y;
```

In more detail, the `uri` production used by an import or export or part is replaced by the following grammar:

```
uri: stringLiteral
   | test ':' stringLiteral uriOpt
uriOpt: '||' uri | <empty>
test: dotNames '==' stringLiteral
    | dotNames
dotNames: identifier | dotNames '.' identifier
```

All instances of `stringLiteral` must be without any interpolations.
Since `uri` is used for both import, export and part declarations, this conditional URI syntax works for all three.

### Semantics

The actual string representing the library to import is found by evaluating a `test` to a boolean result, then picking the following string literal if the test is true, otherwise proceeding to the next (optional) `uri` and trying that. If testing falls through to the empty `uriOpt`, the import fails.

Tests are string comparisons, where a `dotNames` is used to create a key, and that key is used to look up a string value in the environment (using the equivalent of `String.fromEnvironment`). String literals are any normal Dart string literals without any interpolations.

More precisely, the string value of an import `uri` is found using a function `evalUri(uri)` returning a `String` value defined as:

* `evalUri(stringLiteral)` is evaluated as follows

    1. Return the string value of stringLiteral.

* `evalUri(test ':' stringLiteral uriOpt)` is evaluated as follows:

    1. Evaluate `evalTest(test)` to `result`.
    2. If `result` is `true`, return the string value of `stringLiteral`.
    3. Otherwise if `uriOpt` is  `'||' uri`, then return the result of
       `evalUri(uri)`.
    4. Otherwise raise a compile time error.

* `evalTest(dotNames '==' stringLiteral)` is evaluated as follows:

    1. Convert `dotNames` to a string `key`, see below.
    2. Let `lookup` be the result of looking up `key` in the environment, as by
       the constant expression `const String.fromEnvironment(key)`.
    3. Convert `stringLiteral` to a string, `stringValue`, as if it was a
       compile-time constant expression.
    4. If `lookup` is `null` (the key was not in the environment) then return
       `false`.
    5. If `lookup` is equal to `stringValue` (contains the same code units),
       then return `true`.
    6. Otherwise return `false`.

* `evalTest(dotNames)` is equivalent to `evalTest(dotNames '==' "true")`

A `dotNames` is converted to a string by creating a string from the characters of all `identifier` parts of the `dotNames`, separated by `.` characters. That is, as the source characters but with any whitespace removed.

## Platform libraries

The platform and the availability of platform libraries, are signalled by the runtime or compiler by a pre-set environment definitions.

Suggested definitions in the browser:

* `dart.platform=browser`
* `dart.feature.dom=true`

and in the stand-alone VM:

* `dart.platform=server`
* `dart.feature.io=true`

Compilers like dart2js must ensure that the same values are available at compile time and at runtime.

A compilation should target a specific platform. The available libraries are triggered by the platform choice, and the output will only work on that platform. Compiling with dart2js, dart2dart or create_snapshot will generate code for the platform being targeted for deployment.

The boolean flags only exist for features where availability actually differ between recognized platforms, and are only set if the feature is available (since the test `== "true"` works the same for being absent as for having any non-`"true"` value).

Platforms can be added in the future, but tools need to support them. A compiler like dart2js cannot target a platform that it doesn't know. A list of known platforms could be kept in a central location, listing platforms by names and unavailable features.

### User configurations

User configurations can be implemented by setting variables with `-Dfoo.bar=baz` on the command line and testing using `foo.bar == "baz"` in the import expressions.

### Programmatic configuration

A library can use [`const String.fromEnvironment`][env] to check configuration choices at runtime. The same environment values used in imports can also be used in code. This allows, e.g., switching from import based configuration to programmatic configuration if it considers that simpler, or simple configuration could be done programmatically while developing, and then later switch to import based configuration to avoid importing unused libraries.

[env]: https://api.dartlang.org/apidocs/channels/stable/dartdoc-viewer/dart:core.String#id_String-fromEnvironment

### Analysis Tools

An analysis tool, like the Dart Analyzer, can see all possible import strings, and either resolve to a single URI for a given platform/configuration, or examine all the combinations. All the possible imported URIs are directly visible at the import site, as are the tests used to control the access to the imports.

A library can be analyzed for imports of "dart:*" libraries that are not available on all platforms. If a library definitely imports a dart library which is not available on a platform, the library can be marked as not supporting that platform. If the import is "guarded by" a check that ensures that the imported library is available, then the importing library is not marked.

Example:

```dart
library foo.bar;
import dart.platform == "browser" : "dart:svg"
    || "mock-svg.dart";
```

Here, an import of "dart:svg" would normally mark library `foo.bar` as incompatible with the standalone platform. Since the import is guarded by a test that guarantees that the import succeeds, `foo.bar` is not considered incompatible with standalone libraries.

The analyzing software should be able to detect simple guards, which users can stick to in order to guarantee that library compatibility is detectable.

The expressions `dart.platform=="browser"` and `dart.feature.dom=="true"` are both recognized as guaranteeing that "dart:svg" can be imported.

If a library is analyzed (and its transitive imports) and it is deduced that it's incompatible with some platforms, this can be displayed for the library in, e.g., pub.

For user configurations, analyzing software can detect tests of the forms:

```
somename : "somevalue"
somename  (equivalent to somename : "true")
```

as possible guarding configurations, and if they occur in imports, the comparison values can also be listed for the package as possible configuration options.

The possible configurations for a library can also be displayed in, e.g., pub.

### Examples

Using different platform features:

```dart
library platform.independent.library;
export dart.feature.dom : "html_based.dart"
    || dart.feature.io : "io_based.dart";
```

The two libraries implement the same API, but have different implementations.
There is no default. If a platform is introduced which supports neither feature, this library will fail to load on that platform, and the incompatibility can be detected by an analyzer, since this library fails to load if neither "dart.feature.dom" nor "dart.feature.io" is available.

User configurations:

```dart
library logger;
import 
     com.example.logger.level == "verbose" : "src/verbose.dart"
  || com.example.logger.level == "simple"  : "src/plain.dart"
  || "src/plain.dart";
```

Here a logging library allows the system to pick a verbose output strategy by setting an environment variable. It defaults to the plain version.
In many cases, importing both and selecting at runtime is not a problem, but if deployed code size is important, this allows picking the strategy at deployment time and not waste space on the unused version.

## Alternatives

The syntax is deliberately simple. It allows tools to read the import statements without needing to understand any Dart semantics, only only simple syntax.
Since string literals have no interpolations, all possible URI references that can be used for the import are statically known. Since tests are simple, the configurations necessary to trigger a specific import are easily derived.

### Condition expressions.
An alternative would be a more expressive condition language, for example adding logical and/or operators to combine multiple tests. 
Any addition to the expression language will enable some new use-cases that would otherwise be harder (not necessarily impossible), but will likely also cause requests for further additions for new edge-cases that are almost handled.

The more complex the language, the harder it will be for tools to analyze a library and determine which configurations it is compatible with, and which configurations would cause errors.

Pure logical and/or operators can already be implemented by conditionally importing intermediate libraries with further conditional exports. It's verbose, but expected to be rarely needed. If it is needed, the extra library might actually correspond to a meaningful abstraction.

### String expressions.
Instead of having separate condition expressions and string literals, the entire `uri` could be a string expression, allowing (some) string interpolations, or even conditional expressions choosing between strings.

If the import `uri` can embed the value of a system environment property, then there is no way for tools to enumerate the possible imports. A tool will at most be able to analyze a library with regard to a set of known configurations, but will not be able to determine the possible configurations.

### Full compile-time constant expressions.
The logical limit for expressiveness, either of the condition expressions or of string expressions, is to allow any compile-time constant expression. There needs to be some limit on which variables/declarations are available to the expression since the value is computed *before* any `import` or `part` declarations are processed. At most, this could allow access to declarations of `dart:core`, with a special case disallowing the use of conditions inside `dart:core`. That would still allow access to `String.fromEnvironment`, so there would be no need for the `dotNames` syntax for accessing the environment.

Using any compile-time constant expression would require all tools that process Dart libraries to have an implementation of the full Dart compile-time constant semantics, which is more complicated than necessary. Also, later language changes may add more compile time constant expressions, at which point all the tools would also need to be updated.

## Deliverables

### Language specification changes

The specification should be updated with the changes specified above.

### Implementation

Any tool which processes import/export/part declarations needs to also understand the new syntax. These tools should already accept "-Dfoo=bar" environment property declarations on the command line. They will need to have `dart.platform` and `dart.feature.*` environment definitions built-in for their current configuration.

The stand-alone VM only needs to know and implement one configuration, and Dartium has another single configuration. 

Other tools may need to support multiple configurations at the same time. The analyzer may be able to check that a library works correctly in all configurations, say which configurations it does not work for, or say which configurations a warning occurs in.

Pub may be able to use the analyzer to detect which configurations a library is incompatible with, and maybe check whether some dependencies are completely incompatible.

There is no implementation so far.

## Patents rights

TC52, the Ecma technical committee working on evolving the open [Dart standard][], operates under a royalty-free patent policy, [RFPP][] (PDF). This means if the proposal graduates to being sent to TC52, you will have to sign the Ecma TC52 [external contributer form][form] and submit it to Ecma.

[DEP Proposal Location]: https://github.com/lrhn/dep-configured-imports/
[dart standard]: http://www.ecma-international.org/publications/standards/Ecma-408.htm
[rfpp]: http://www.ecma-international.org/memento/TC52%20policy/Ecma%20Experimental%20TC52%20Royalty-Free%20Patent%20Policy.pdf
[form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf
