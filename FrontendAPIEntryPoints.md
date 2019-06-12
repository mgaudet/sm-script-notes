# Frontend API Entry Points

## Discussion of Encoding in Interfaces 

* Current standard seems to be to provide `JS::SourceText<T>`, where `T` is either `char16_t` or `mozilla::Utf8Unit`. 
* Some interfaces have variants; `Compile...Utf8`, which will typically inflate UTF-8 to `char16_t` before compiling, and `Compile...DontInflate` which uses the new UTF-8 parsing that Jeff has been working on.
* Longer term plan is that `DontInflate` just goes away once the parser is reliably working without inflation, and the inflating interface can just be replaced with the don't inflate interface. 

## Semi-external interfaces

### `CompileGlobalScript`

```
extern JSScript* CompileGlobalScript(
    GlobalScriptInfo& info, JS::SourceText<T>& srcBuf,
    ScriptSourceObject** sourceObjectOut = nullptr);
```
* (Not actually a template, but just deduplicating by hand; exsists for both Utf8Unit and char16_t)
* The debugger is the only 'external' consumer of this interface IMO, and so it's labelled here as 'semi-external'.

## Internal Interfaces

### `CompileStandAloneFunction` (and friends)

```cpp
//
// Compile a single function. The source in srcBuf must match the ECMA-262
// FunctionExpression production.
//
// If nonzero, parameterListEnd is the offset within srcBuf where the parameter
// list is expected to end. During parsing, if we find that it ends anywhere
// else, it's a SyntaxError. This is used to implement the Function constructor;
// it's how we detect that these weird cases are SyntaxErrors:
//
//     Function("/*", "*/x) {")
//     Function("x){ if (3", "return x;}")
//
MOZ_MUST_USE bool CompileStandaloneFunction(
    JSContext* cx, MutableHandleFunction fun,
    const JS::ReadOnlyCompileOptions& options, JS::SourceText<char16_t>& srcBuf,
    const mozilla::Maybe<uint32_t>& parameterListEnd,
    HandleScope enclosingScope = nullptr);

MOZ_MUST_USE bool CompileStandaloneGenerator(
    JSContext* cx, MutableHandleFunction fun,
    const JS::ReadOnlyCompileOptions& options, JS::SourceText<char16_t>& srcBuf,
    const mozilla::Maybe<uint32_t>& parameterListEnd);

MOZ_MUST_USE bool CompileStandaloneAsyncFunction(
    JSContext* cx, MutableHandleFunction fun,
    const JS::ReadOnlyCompileOptions& options, JS::SourceText<char16_t>& srcBuf,
    const mozilla::Maybe<uint32_t>& parameterListEnd);

MOZ_MUST_USE bool CompileStandaloneAsyncGenerator(
    JSContext* cx, MutableHandleFunction fun,
    const JS::ReadOnlyCompileOptions& options, JS::SourceText<char16_t>& srcBuf,
    const mozilla::Maybe<uint32_t>& parameterListEnd);
```

* Ends up in `Parser<>::standAloneFunction`
* Only `<char16_t>` today. 

### `createScriptForLazilyInterpretedFunction`

Used during delazification. Ends up in `Parser<>::standAloneLazyFunction`.


## External Interfaces

### `JS::Compile`

```cpp
/**
 * Compile the provided script using the given options.  Return the script on
 * success, or return null on failure (usually with an error reported).
 */
extern JS_PUBLIC_API JSScript* Compile(JSContext* cx,
                                       const ReadOnlyCompileOptions& options,
                                       SourceText<T>& srcBuf);
```

(Not actually a template, but just deduplicating by hand; exsists for both `Utf8Unit` and `char16_t`)

* Tunnels down via `CompileSourceBuffer` to `CompileGlobalScript`. 

### `CompileForNonSyntacticScope`

```cpp
/**
 * Compile the provided UTF-8 data into a script in a non-syntactic scope.  It
 * is an error if the data contains invalid UTF-8.  Return the script on
 * success, or return null on failure (usually with an error reported).
 */
extern JS_PUBLIC_API JSScript* CompileForNonSyntacticScope(
    JSContext* cx, const ReadOnlyCompileOptions& options,
    SourceText<mozilla::Utf8Unit>& srcBuf);
```

(There's also a `char16_t` overload) 

* Tunnels down via `CompileSourceBuffer` to `CompileGlobalScript`.
* Really the same as `JS::Compile`, but compies and mutates the `ReadOnlyCompileOptions` to set the NonSyntacticScope flag. 

### `CompileModule`

```
/**
 * Parse the given source buffer as a module in the scope of the current global
 * of cx and return a source text module record.
 */
extern JS_PUBLIC_API JSObject* CompileModule(
    JSContext* cx, const ReadOnlyCompileOptions& options,
    SourceText<T>& srcBuf);
```

## Off thread Variants

### `CompileOffThread`

```cpp
/*
 * Off thread compilation control flow.
 *
 * After successfully triggering an off thread compile of a script, the
 * callback will eventually be invoked with the specified data and a token
 * for the compilation. The callback will be invoked while off thread,
 * so must ensure that its operations are thread safe. Afterwards, one of the
 * following functions must be invoked on the runtime's main thread:
 *
 * - FinishOffThreadScript, to get the result script (or nullptr on failure).
 * - CancelOffThreadScript, to free the resources without creating a script.
 *
 * The characters passed in to CompileOffThread must remain live until the
 * callback is invoked, and the resulting script will be rooted until the call
 * to FinishOffThreadScript.
 */

extern JS_PUBLIC_API bool CompileOffThread(
    JSContext* cx, const ReadOnlyCompileOptions& options,
    SourceText<char16_t>& srcBuf, OffThreadCompileCallback callback,
    void* callbackData);
```

* The Utf8 overload of this is a DontInflateVariant. 

### `CompileOffThreadModule` 

```cpp
extern JS_PUBLIC_API bool CompileOffThreadModule(
    JSContext* cx, const ReadOnlyCompileOptions& options,
    SourceText<char16_t>& srcBuf, OffThreadCompileCallback callback,
    void* callbackData);
```


## Discussion

There's some good news I think in this; Outside of lazy script copmilation, I think nearly all the interfaces end up tunnelling through either `CompileGlobalScript` or `CompileStandaloneFunction`


## Incremental Parsing Interface

Here's a funny interface we use to implement repls.

```cpp
/**
 * Given a buffer, return false if the buffer might become a valid JavaScript
 * script with the addition of more lines, or true if the validity of such a
 * script is conclusively known (because it's the prefix of a valid script --
 * and possibly the entirety of such a script).
 *
 * The intent of this function is to enable interactive compilation: accumulate
 * lines in a buffer until JS_Utf8BufferIsCompilableUnit is true, then pass it
 * to the compiler.
 *
 * The provided buffer is interpreted as UTF-8 data.  An error is reported if
 * a UTF-8 encoding error is encountered.
 */
extern JS_PUBLIC_API bool JS_Utf8BufferIsCompilableUnit(
    JSContext* cx, JS::Handle<JSObject*> obj, const char* utf8, size_t length);
```

Under the covers it just does a compilation, throwing it away, and returning true if it worked and false if it got an `isUnexpectedEOF` result. 

Seems like there's room for improvement! 

