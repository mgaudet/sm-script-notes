# How functions live and die in SpiderMonkey

In the beginning, there is source. 

## Source Representation

* Source is stored by a `ScriptSource` object. This indirection allows us to manipulate the source, like compressing it for example. 
* Most users of the source will actually see a `ScriptSourceObject`, which holds GC pointers related to ScriptSource, and is used to properly account for cross zone pointers.  

## Source to Executable

We start by [parsing into a parse tree][pt], then converting [the parse tree into bytecode][bc]. 

Overall, our parsing and bytecode emission are quite strongly coupled (which is one of the aspects that makes the parser complicated). In my notes I will attempt to distinguish between the parser and bytecode compilers, however, the code sometimes muddles this distinction. 

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

becomes (call `ParseNode::dump()`)

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

In order to handle this, SpiderMonkey has what's called Lazy parsing. Lazy parsing will syntax parse a function, as there are certain errors that are required to be reported eagerly, but will not generate a full parse tree. In the parse tree dump  a lazy parsed function shows up as `#NULL`:

```
(StatementList [(Function #NULL)
                (Function #NULL)
                (ExpressionStmt (CallExpr topLevelFunction2
                                          (Arguments [10])))])
```

This is because lazy parsed FunctionNodes don't get a `body()`. 

Lazy parsed functions get `JSFunctions` created for them, and an associated `LazyScript` which holds the information required to generate bytecode should the `JSFunction` get invoked. The process of taking a lazy `JSFunction` and creating bytecode for it before execution (or inspection) is called delazification. 

##### Syntax Only Parsing

The parser is [parameterized at certain levels in its inheritance hierarchy](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/Parser.h#54-65,101-111) to distinguish between a full parse and a syntax only parse. 

In a FullParse parser, nodes in the parse tree are full structs, holding on to sufficient information to produce bytecode. 

In a SyntaxOnly parser, nodes in the parse tree are simply enum values. This is sufficient to produce the required syntax errors of a lazy parse. 

At the end of `PerHandlerParser<SyntaxParseHandler>::finishFunction`, a LazyScript is allocated to ensure we can generate a full parse of this function when we execute. 

It's worth noting that not all failures to syntax parse indicate an unparsable script. There are some constructs that the SyntaxOnly parser has insufficient information to resolve, and so in those circumstances we must fall back to a full parse. 

#### Relazification 

In some circumstances it is possible to throw away the JSScript (bytecode) for a JSFunction, and return to being a Lazy JSFunction. This process is referred to as relazification. 

* ["Only functions without inner functions or direct eval are re-lazified."][relazification]

[relazification]: https://searchfox.org/mozilla-central/rev/7556a400affa9eb99e522d2d17c40689fa23a729/js/src/vm/JSFunction.cpp#1555-1556

#### Parser Interfaces of Note

The parser has a lot of entry points. Some notable ones: 

* [`ListNode* Parser<FullParseHandler, Unit>::globalBody`](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/Parser.cpp#1423) called from [`JSScript* frontend::ScriptCompiler<Unit>::compileScript`](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/BytecodeCompiler.cpp#516)
* [`frontend::CompileLazyFunction`](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/BytecodeCompiler.cpp#1015,1020) (which in turn calls [`standaloneLazyFunction`](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/BytecodeCompiler.cpp#973))

#### Assorted Parser notes

* [This comment](https://searchfox.org/mozilla-central/source/js/src/frontend/Parser.h#13) is an excellent summary of the class hierarchy behind the parser. 

### Compilation Interfaces of Note: 

* [`JSScript* frontend::ScriptCompiler<Unit>::compileScript`](https://searchfox.org/mozilla-central/rev/9eb2f739c165b4e294929f7b99fbb4d90f8a396b/js/src/frontend/BytecodeCompiler.cpp#516)


### A tour through function related types in SpiderMonkey

* `FunctionNode`: Parser representation of a function definition. Body may be null if lazily parsed. 
* `FunctionBox`: A `SharedContext` struct, pointed to by a `FunctionNode`, which holds a reference to an allocated `JSFunction`
* `JSFunction`: A runtime representation of a Javascript function; invokable
  * `FunctionExtended` is a subclass of JSFunction that has two additional extended slots, used for saving extra information for specific variants of `JSFunction` (ie. arrow functions, methods, WASM and ASMJS functions). These get allocated with the `allocKind` of `FUNCTION_EXTENDED`. 
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

##### GC and Ownership

* In the lazy case, the LazyScript holds on to the associated JSFunction, and traces it for GC purposes. It is through the LazyScript that the function is obtained for delazification.


##### Canonical Functions

In the case of lambdas, there's a 'canonical' JSFunction, which corresponds to its origin, and then new copies are created at various invocation points (Correct?). The original JSFunction, the canonical one, holds the JSScript. The Canonical function is the one allocated by the Front end. 

#### `JSScript` 

##### SharedScriptData

This is data that can be shared by many scripts across a runtime. 

##### PrivateScriptData

Holds data private to a single script. Stored as a header object + trailing array. 

## Parsing + Garbage Collection


The parser currently interacts with the garbage collector because it allocates collected types. 

1. `JSFunctions` 
2. Regular Expression Objects. 
3. BigInts
4. Atoms

### Keeping Allocated objects alive

* RegEx objects are held onto by an `ObjectBox`. 
* `JSFunction`s are held onto by `FunctionBox`, which is (in addition to being a subclass of `SharedContext`) a subclass of `ObjectBox`. 
* BigInts are held onto by `BigIntBox`. 

The three objects above all inherit from TraceListNode, which provides a list for the GC to trace when collecting. It's the parser equivalent of `Rooted`. 

Atoms are kept alive by an `AutoKeepAtoms` that is a member of the Parser. 

### Reference Compaction

During bytecode emission, the references to all our [emitted objects are compacted into a table](https://searchfox.org/mozilla-central/rev/fe7dbedf223c0fc4b37d5bd72293438dfbca6cec/js/src/frontend/BytecodeSection.cpp#45)

Object Box objects end up in `PrivateScriptData`. 

### Parser Realm 

Objects allocated in the parser are allocated in the Parser realm. At the end of a parse, objects created during the parse must be transferred from the parser realm to the desired destination realm. This is accomplished via a fairly complicated helper called [`mergeRealms`](https://searchfox.org/mozilla-central/rev/fe7dbedf223c0fc4b37d5bd72293438dfbca6cec/js/src/gc/GC.cpp#8228-8374). 

## Bytecode Emission 

A high level overview of Bytecode Emission (Ignoring a lot of special cases) 

1. Kickoff seems to be [`BytecodeEmitter::emitScript`](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/frontend/BytecodeCompiler.cpp#539). 
2. After setting up scoping rules etc, [`emitTree`](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/frontend/BytecodeEmitter.cpp#2452) is called. `emitTree` knows how to emit each parse node kind, as well as walk over list-types. (Since a high level representation of a function body is a `StatmentList`, this naturally works out. 
3. A `JSOP_RETVAL` is emitted
4. [Names are provided to anonymous functions rooted at this parse tree](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/frontend/BytecodeEmitter.cpp#2472-2474)
5. [We initialize the script from the emitter](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/frontend/BytecodeEmitter.cpp#2476-2478); this consists of: 
  - Setting [immutable script flags from the emitter](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/vm/JSScript.cpp#3985-3996) 
  - Pulling flags off of the `FunctionBox`
  - Initializing the `PrivateScriptData`
  	 - This is where the [GC things are copied out of the emitter into the `PrivateScriptData`](https://searchfox.org/mozilla-central/rev/da3f3eaaacb6fb344fd21ac29ace2da0e33f12d3/js/src/vm/JSScript.cpp#3685-3687) 
  - Initializing the `SharedScriptData` 
  - Shared script data is deduplicated by the `ScriptDataTable`
  - If the shared context behind the `BytecodeEmitter` being used to generate this script is a `FunctionBox` (When it's not, it must be a Module, I believe), the function is informed of the script.
  - All the inner functions pointed to by the per-script data have their enclosing scope set.
6. Finally, we tell the Debugger about the script. 

  
### GC Notes

Bytecode emission also creates GC things. A non-exhaustive list includes 

* Scopes
* Template Objects (for `JSOP_NEWOBJECT`)
* Module Objects

## Questions & Some Answers

* Why is `GlobalScriptInfo` a `BytecodeCompiler` subclass? 
  * Has-a GlobalSharedContext, and holds onto the ScriptSourceObject via inheritance from BytecodeCompiler 
* What is the division of responsibilities between JSFunction and JSScript?
* What is a closed over binding? 
  * `var x; function f() { ... x ... }` - variables used in a sub-function. 
  * `noteDeclaredName` and `noteUsedName` are used to compute closed over bindings. 
  * "`noteUsedName` and friends manipulate the `UsedNameTracke`r class, and then `handler_.finishLexicalScope` finalizes a scope and figures out what variables are in that scope, then the BytecodeEmitter computes where variables are"
* Why do we emit in infix order, and how do we... not?
  * One big reason has to do with the 2 pass compilation model we have, with parsing and emission being the only two real passes. 
* Do FunctionBoxes outlive a parser, or do they too die when the parser dies? 
  * No, the function box dies. The information it has collected is forwarded onto either the script, the function, or the lazy script. 
* If the top level function of a nested tree of functions is lazily parsed, do all the inner functions get JSFunctions and LazyScripts, or only the top most LazyScript? 
* Are the objects allocated during parser emission allocated into the parser realm, and merge-realm'd, or are they allocated directly into the correct realm? 
* Why does `JSScript` store `sourceStart_`, `sourceEnd_` etc?
* During delazification we create a FunctionBox to hold the already existing JSFunction (`initFromLazyFunction`). I noticed this when investigating a problem and realized that the mapping between FunctionBoxes and JSFunctions isn't 1-1, as you get multiple FunctionBoxes that point to the same JSFunction. Conceptually what does this mean? 


## Useful References: 

* [Bug 678037 (LazyBytecode)](https://bugzilla.mozilla.org/show_bug.cgi?id=678037)

## Thanks

Thanks to Ashley, Ted, Jeff for their help!
