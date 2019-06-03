# How functions live and die in SpiderMonkey

In the beginning, there is source. 

## Source Representation

* Source is stored by a `ScriptSource` object. This indirection allows us to manipulate the source, like compressing it for example. 
* Most users of the source will actually see a `ScriptSourceObject`, which holds GC pointers related to ScriptSource, and is used to properly account for cross zone pointers.  

## Source to Executable

We start by [parsing into a parse tree][pt], then converting [the parse tree into bytecode][bc]

[pt]: https://searchfox.org/mozilla-central/rev/7556a400affa9eb99e522d2d17c40689fa23a729/js/src/frontend/BytecodeCompiler.cpp#531-539
[bc]: https://searchfox.org/mozilla-central/rev/7556a400affa9eb99e522d2d17c40689fa23a729/js/src/frontend/BytecodeCompiler.cpp#555-557. 

### Parsing 

There are a couple of entrances into parsing, and a big complication in the form of Lazy parsing. For now, let's talk about regular parsing: 

#### Structure: 

Top level of the parse is a StatementList; each statement has child trees. 

Here's how the first few lines of Utilities.js starts to parse: 

```JS
var std_Symbol = Symbol;

function List() {
    this.length = 0;
}
MakeConstructible(List, {__proto__: null});
```

becomes 

```
(StatementList [(VarStmt [(AssignExpr std_Symbol
                                      Symbol)])
                (Function (ParamsBody [(LexicalScope []
                                         (StatementList [(ExpressionStmt (AssignExpr (.length (ThisExpr .this))
                                                                                     0))]))]))
                (ExpressionStmt (CallExpr MakeConstructible
                                          (Arguments [List
                                                      (ObjectExpr [(MutateProto #null)])])))
```

I've elided the rest of the files, because it goes on a long time. Different expression types get different node types


##### Parse Node Lifetimes

As described in [this comment][comment] ParseNodes live only as long as the parser that generates them.

[comment]: https://searchfox.org/mozilla-central/source/js/src/frontend/ParseNode.h#20

#### Lazy Parsing 

Lazy Parsing is a solution to the empirical fact that an appreciable fraction of the JS downloaded into a browser isn't actually executed in a given session. Imagine the broad range of functionality required in a web app, and the narrow path an average session would go across. 

In order to handle this, SpiderMonkey has what's called Lazy parsing. Lazy parsing will syntax parse a function, as there are certain errors that are required to be reported eagerly, but will not generate a full parse tree. In the parse tree dump a lazy parsed function shows up as `#NULL`: 

```
(StatementList [(Function #NULL)
                (Function #NULL)
                (ExpressionStmt (CallExpr topLevelFunction2
                                          (Arguments [10])))])
```

This is because lazy parsed FunctionNodes don't get a `body()`. 

Lazy parsed functions get `JSFunctions` created for them, and an associated `LazyScript` which holds the information required to generate bytecode should the `JSFunction` get invoked. The process of taking a lazy `JSFunction` and creating bytecode for it before execution (or inspection) is called delazification. 

#### Relazification 

In some circumstances it is possible to throw away the JSScript (bytecode) for a JSFunction, and return to being a Lazy JSFunction. This process is referred to as relazification. 

* ["Only functions without inner functions or direct eval are re-lazified."][relazification]

[relazification]: https://searchfox.org/mozilla-central/rev/7556a400affa9eb99e522d2d17c40689fa23a729/js/src/vm/JSFunction.cpp#1555-1556

### Bytecode Emission


### A tour through function related types in SpiderMonkey

* `FunctionNode`: Parser representation of a function definition. Body may be null if lazily parsed. 
* `FunctionBox`: A `SharedContext` struct, pointed to by a `FunctionNode`, which holds a reference to an allocated `JSFunction`
* `JSFunction`: A runtime representation of a Javascript function; invokable
* `JSScript`: Bytecode used for execution on the JSVM 
* `LazyScript`: The information required to cook up bytecode for a JSFunction after a syntax only parse  

#### JSFunction

JSFunction can be thought of as a variant over JSScript, LazyScript and Native.

```
  // Interpreted functions may either have an explicit JSScript (hasScript())
  // or be lazy with sufficient information to construct the JSScript if
  // necessary (isInterpretedLazy()).
  //
  // A lazy function will have a LazyScript if the function came from parsed
  // source, or nullptr if the function is a clone of a self hosted function.
  //
```

`Native` means a JSFunction backed by C++ (or generated code in the case of WASM?) that conforms to the JSNative signature; 

```Cpp
/* Typedef for native functions called by the JS VM. */
using JSNative = bool (*)(JSContext* cx, unsigned argc, JS::Value* vp);
```

##### Canonical Functions

In the case of lambdas, there's a 'canonical' JSFunction, which corresponds to its origin, and then new copies are created at various invokation points (Correct?). The original JSFunction, the canonical one, holds the JSScript. The Canonical function is the one allocated by the Front end. 

Q: What about 'Native' code? 


## Questions, so many questions:

* Why is `GlobalScriptInfo` a `BytecodeCompiler` subclass? 
  * Has-a GlobalSharedContext, and holds onto the ScriptSourceObject via inheritance from BytecodeCompiler 
* What is the division of responsiblities between JSFunction and JSScript?
* What is a closed over binding? 
* How to build a function tree
  * What lifetime would function trees have? Ted thinks parse tree would be inside functions
* Why do we emit in infix order, and how do we... not?
* When is the 'end' of parsing; for link steps 


## Useful References: 

* [Bug 678037 (LazyBytecode)](https://bugzilla.mozilla.org/show_bug.cgi?id=678037)
