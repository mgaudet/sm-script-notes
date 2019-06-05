# For each FunctionBox allocation, where is the associated function allocated?  
## `FunctionNode* Parser<FullParseHandler, Unit>::standaloneFunction`

* Has a Function argument, Called from 
* `FunctionNode* frontend::StandaloneFunctionCompiler<Unit>::parse` which has a function argument, called from 
* `static bool CompileStandaloneFunction` which has a function argument. Called from 
  * `bool frontend::CompileStandaloneGenerator`
  * `bool frontend::CompileStandaloneAsyncFunction`
  * `bool frontend::CompileStandaloneAsyncGenerator`
     * All of the above `CompileStandAlone`s, plus the below `CompileStandAloneFunction` are called from `CreateDynamicFunction`: the function is allocated there. The above three are -only- called from `CreateDynamicFunction`. 
  * `bool frontend::CompileStandaloneFunction` which has a function argument. Called from 
     * `JSFunction* FunctionCompiler::finish` which [allocates the function](https://searchfox.org/mozilla-central/rev/952521e6164ddffa3f34bc8cfa5a81afc5b859c4/js/src/vm/CompilationAndEvaluation.cpp#348-353)
     * `CompileStandAloneFunction` is called from ` HandleInstantiationFailure`, which allocates a JSFunction for when AsmJS compilation fails for some reason.

## `bool Parser<FullParseHandler, Unit>::skipLazyInnerFunction`

* Function comes from ` RootedFunction fun(cx_, handler_.nextLazyInnerFunction());` I can't quite figure out how this works (how do you get the 'right' lazyInnerFunction, does it matter?) -- is this infix related. 

## `bool Parser<FullParseHandler, Unit>::trySyntaxParseInnerFunction`

* Has a function argument, called from 
* `GeneralParser<ParseHandler, Unit>::functionDefinition` which [allocates the function](https://searchfox.org/mozilla-central/rev/952521e6164ddffa3f34bc8cfa5a81afc5b859c4/js/src/frontend/Parser.cpp#2579-2580)

## `GeneralParser<ParseHandler, Unit>::innerFunction`

* Has function argument, called from 
  * `bool Parser<FullParseHandler, Unit>::trySyntaxParseInnerFunction`, has function argument. Called from: See above (`GeneralParser<>::functionDefinition`) 
    * An aside: This happens if we fail to do a syntax parse.
  * `bool Parser<SyntaxParseHandler, Unit>::trySyntaxParseInnerFunction`, has function argument, called from: see above (`GeneralParser<>::functionDefinition`)

## `Parser<FullParseHandler, Unit>::standaloneLazyFunction`

* Has function argument called from 
* `static bool CompileLazyFunctionImpl`. Uses `Rooted<JSFunction*> fun(cx, lazy->functionNonDelazifying());` as the function
  * Not sure how that works 

## `GeneralParser<ParseHandler, Unit>::synthesizeConstructor`

* allocates a new function directly

## `GeneralParser<ParseHandler, Unit>::fieldInitializerOpt(` 

* Allocates a new function directly. 
