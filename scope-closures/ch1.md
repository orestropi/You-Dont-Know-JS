# You Don't Know JS Yet: Scope & Closures - 2nd Edition
# Chapter 1: What's the Scope?

By the time you've written your first few programs, you're likely getting somewhat comfortable with creating variables and storing values in them. Working with variables is one of the most foundational things we do in programming!

But you may not have considered very closely the underlying mechanisms used by the engine to organize and manage these variables. I don't mean how the memory is allocated on the computer, but rather: how does JS know which variables are accessible by any given statement, and how does it handle two variables of the same name?

The answers to questions like these take the form of well-defined rules called scope. This book will dig through all aspects of scope—how it works, what it's useful for, gotchas to avoid—and then point toward common scope patterns that guide the structure of programs.

Our first step is to uncover how the JS engine processes our program **before** it runs.

## About This Book

Welcome to book 2 in the *You Don't Know JS Yet* series! If you already finished *Get Started* (the first book), you're in the right spot! If not, before you proceed I encourage you to *start there* for the best foundation.

Our focus will be the first of three pillars in the JS language: the scope system and its function closures, as well as the power of the module design pattern.

JS is typically classified as an interpreted scripting language, so it's assumed by most that JS programs are processed in a single, top-down pass. But JS is in fact parsed/compiled in a separate phase **before execution begins**. The code author's decisions on where to place variables, functions, and blocks with respect to each other are analyzed according to the rules of scope, during the initial parsing/compilation phase. The resulting scope structure is generally unaffected by runtime conditions.

JS functions are themselves first-class values; they can be assigned and passed around just like numbers or strings. But since these functions hold and access variables, they maintain their original scope no matter where in the program the functions are eventually executed. This is called closure.

Modules are a code organization pattern characterized by public methods that have privileged access (via closure) to hidden variables and functions in the internal scope of the module.

## Compiled vs. Interpreted

You may have heard of *code compilation* before, but perhaps it seems like a mysterious black box where source code slides in one end and executable programs pop out the other.

It's not mysterious or magical, though. Code compilation is a set of steps that process the text of your code and turn it into a list of instructions the computer can understand. Typically, the whole source code is transformed at once, and those resulting instructions are saved as output (usually in a file) that can later be executed.

You also may have heard that code can be *interpreted*, so how is that different from being *compiled*?

Interpretation performs a similar task to compilation, in that it transforms your program into machine-understandable instructions. But the processing model is different. Unlike a program being compiled all at once, with interpretation the source code is transformed line by line; each line or statement is executed before immediately proceeding to processing the next line of the source code.

<figure>
    <img src="images/fig1.png" width="650" alt="Code Compilation and Code Interpretation" align="center">
    <figcaption><em>Fig. 1: Compiled vs. Interpreted Code</em></figcaption>
    <br><br>
</figure>

Figure 1 illustrates compilation vs. interpretation of programs.

Are these two processing models mutually exclusive? Generally, yes. However, the issue is more nuanced, because interpretation can actually take other forms than just operating line by line on source code text. Modern JS engines actually employ numerous variations of both compilation and interpretation in the handling of JS programs.

Recall that we surveyed this topic in Chapter 1 of the *Get Started* book. Our conclusion there is that JS is most accurately portrayed as a **compiled language**. For the benefit of readers here, the following sections will revisit and expand on that assertion.

## Compiling Code

But first, why does it even matter whether JS is compiled or not?

Scope is primarily determined during compilation, so understanding how compilation and execution relate is key in mastering scope.

In classic compiler theory, a program is processed by a compiler in three basic stages:

1. **Tokenizing/Lexing:** breaking up a string of characters into meaningful (to the language) chunks, called tokens. For instance, consider the program: `var a = 2;`. This program would likely be broken up into the following tokens: `var`, `a`, `=`, `2`, and `;`. Whitespace may or may not be persisted as a token, depending on whether it's meaningful or not.

    (The difference between tokenizing and lexing is subtle and academic, but it centers on whether or not these tokens are identified in a *stateless* or *stateful* way. Put simply, if the tokenizer were to invoke stateful parsing rules to figure out whether `a` should be considered a distinct token or just part of another token, *that* would be **lexing**.)

2. **Parsing:** taking a stream (array) of tokens and turning it into a tree of nested elements, which collectively represent the grammatical structure of the program. This is called an Abstract Syntax Tree (AST).

    For example, the tree for `var a = 2;` might start with a top-level node called `VariableDeclaration`, with a child node called `Identifier` (whose value is `a`), and another child called `AssignmentExpression` which itself has a child called `NumericLiteral` (whose value is `2`).

3. **Code Generation:** taking an AST and turning it into executable code. This part varies greatly depending on the language, the platform it's targeting, and other factors.

    The JS engine takes the just described AST for `var a = 2;` and turns it into a set of machine instructions to actually *create* a variable called `a` (including reserving memory, etc.), and then store a value into `a`.

| NOTE: |
| :--- |
| The implementation details of a JS engine (utilizing system memory resources, etc.) is much deeper than we will dig here. We'll keep our focus on the observable behavior of our programs and let the JS engine manage those deeper system-level abstractions. |

The JS engine is vastly more complex than *just* these three stages. In the process of parsing and code generation, there are steps to optimize the performance of the execution (i.e., collapsing redundant elements). In fact, code can even be re-compiled and re-optimized during the progression of execution.

So, I'm painting only with broad strokes here. But you'll see shortly why *these* details we *do* cover, even at a high level, are relevant.

JS engines don't have the luxury of an abundance of time to perform their work and optimizations, because JS compilation doesn't happen in a build step ahead of time, as with other languages. It usually must happen in mere microseconds (or less!) right before the code is executed. To ensure the fastest performance under these constraints, JS engines use all kinds of tricks (like JITs, which lazy compile and even hot re-compile); these are well beyond the "scope" of our discussion here.

### Required: Two Phases

To state it as simply as possible, the most important observation we can make about processing of JS programs is that it occurs in (at least) two phases: parsing/compilation first, then execution.

The separation of a parsing/compilation phase from the subsequent execution phase is observable fact, not theory or opinion. While the JS specification does not require "compilation" explicitly, it requires behavior that is essentially only practical with a compile-then-execute approach.

There are three program characteristics you can observe to prove this to yourself: syntax errors, early errors, and hoisting.

#### Syntax Errors from the Start

Consider this program:

```js
var greeting = "Hello";

console.log(greeting);

greeting = ."Hi";
// SyntaxError: unexpected token .
```

This program produces no output (`"Hello"` is not printed), but instead throws a `SyntaxError` about the unexpected `.` token right before the `"Hi"` string. Since the syntax error happens after the well-formed `console.log(..)` statement, if JS was executing top-down line by line, one would expect the `"Hello"` message being printed before the syntax error being thrown. That doesn't happen.

In fact, the only way the JS engine could know about the syntax error on the third line, before executing the first and second lines, is by the JS engine first parsing the entire program before any of it is executed.

#### Early Errors

Next, consider:

```js
console.log("Howdy");

saySomething("Hello","Hi");
// Uncaught SyntaxError: Duplicate parameter name not
// allowed in this context

function saySomething(greeting,greeting) {
    "use strict";
    console.log(greeting);
}
```

The `"Howdy"` message is not printed, despite being a well-formed statement.

Instead, just like the snippet in the previous section, the `SyntaxError` here is thrown before the program is executed. In this case, it's because strict-mode (opted in for only the `saySomething(..)` function here) forbids, among many other things, functions to have duplicate parameter names; this has always been allowed in non-strict-mode.

The error thrown is not a syntax error in the sense of being a malformed string of tokens (like `."Hi"` prior), but in strict-mode is nonetheless required by the specification to be thrown as an "early error" before any execution begins.

But how does the JS engine know that the `greeting` parameter has been duplicated? How does it know that the `saySomething(..)` function is even in strict-mode while processing the parameter list (the `"use strict"` pragma appears only later, in the function body)?

Again, the only reasonable explanation is that the code must first be *fully* parsed before any execution occurs.

#### Hoisting

Finally, consider:

```js
function saySomething() {
    var greeting = "Hello";
    {
        greeting = "Howdy";  // error comes from here
        let greeting = "Hi";
        console.log(greeting);
    }
}

saySomething();
// ReferenceError: Cannot access 'greeting' before
// initialization
```

The noted `ReferenceError` occurs from the line with the statement `greeting = "Howdy"`. What's happening is that the `greeting` variable for that statement belongs to the declaration on the next line, `let greeting = "Hi"`, rather than to the previous `var greeting = "Hello"` statement.

The only way the JS engine could know, at the line where the error is thrown, that the *next statement* would declare a block-scoped variable of the same name (`greeting`) is if the JS engine had already processed this code in an earlier pass, and already set up all the scopes and their variable associations. This processing of scopes and declarations can only accurately be accomplished by parsing the program before execution.

The `ReferenceError` here technically comes from `greeting = "Howdy"` accessing the `greeting` variable **too early**, a conflict referred to as the Temporal Dead Zone (TDZ). Chapter 5 will cover this in more detail.

| WARNING: |
| :--- |
| It's often asserted that `let` and `const` declarations are not hoisted, as an explanation of the TDZ behavior just illustrated. But this is not accurate. We'll come back and explain both the hoisting and TDZ of `let`/`const` in Chapter 5. |

Hopefully you're now convinced that JS programs are parsed before any execution begins. But does it prove they are compiled?

This is an interesting question to ponder. Could JS parse a program, but then execute that program by *interpreting* operations represented in the AST **without** first compiling the program? Yes, that is *possible*. But it's extremely unlikely, mostly because it would be extremely inefficient performance wise.

It's hard to imagine a production-quality JS engine going to all the trouble of parsing a program into an AST, but not then converting (aka, "compiling") that AST into the most efficient (binary) representation for the engine to then execute.

Many have endeavored to split hairs with this terminology, as there's plenty of nuance and "well, actually..." interjections floating around. But in spirit and in practice, what the engine is doing in processing JS programs is **much more alike compilation** than not.

Classifying JS as a compiled language is not concerned with the distribution model for its binary (or byte-code) executable representations, but rather in keeping a clear distinction in our minds about the phase where JS code is processed and analyzed; this phase observably and indisputedly happens *before* the code starts to be executed.

We need proper mental models of how the JS engine treats our code if we want to understand JS and scope effectively.

## Compiler Speak

With awareness of the two-phase processing of a JS program (compile, then execute), let's turn our attention to how the JS engine identifies variables and determines the scopes of a program as it is compiled.

First, let's examine a simple JS program to use for analysis over the next several chapters:

```js
var students = [
    { id: 14, name: "Kyle" },
    { id: 73, name: "Suzy" },
    { id: 112, name: "Frank" },
    { id: 6, name: "Sarah" }
];

function getStudentName(studentID) {
    for (let student of students) {
        if (student.id == studentID) {
            return student.name;
        }
    }
}

var nextStudent = getStudentName(73);

console.log(nextStudent);
// Suzy
```

Other than declarations, all occurrences of variables/identifiers in a program serve in one of two "roles": either they're the *target* of an assignment or they're the *source* of a value.

(When I first learned compiler theory while earning my computer science degree, we were taught the terms "LHS" (aka, *target*) and "RHS" (aka, *source*) for these roles, respectively. As you might guess from the "L" and the "R", the acronyms mean "Left-Hand Side" and "Right-Hand Side", as in left and right sides of an `=` assignment operator. However, assignment targets and sources don't always literally appear on the left or right of an `=`, so it's probably clearer to think in terms of *target* / *source* rather than *left* / *right*.)

How do you know if a variable is a *target*? Check if there is a value that is being assigned to it; if so, it's a *target*. If not, then the variable is a *source*.

For the JS engine to properly handle a program's variables, it must first label each occurrence of a variable as *target* or *source*. We'll dig in now to how each role is determined.

### Targets

What makes a variable a *target*? Consider:

```js
students = [ // ..
```

This statement is clearly an assignment operation; remember, the `var students` part is handled entirely as a declaration at compile time, and is thus irrelevant during execution; we left it out for clarity and focus. Same with the `nextStudent = getStudentName(73)` statement.

But there are three other *target* assignment operations in the code that are perhaps less obvious. One of them:

```js
for (let student of students) {
```

That statement assigns a value to `student` for each iteration of the loop. Another *target* reference:

```js
getStudentName(73)
```

But how is that an assignment to a *target*? Look closely: the argument `73` is assigned to the parameter `studentID`.

And there's one last (subtle) *target* reference in our program. Can you spot it?

..

..

..

Did you identify this one?

```js
function getStudentName(studentID) {
```

A `function` declaration is a special case of a *target* reference. You can think of it sort of like `var getStudentName = function(studentID)`, but that's not exactly accurate. An identifier `getStudentName` is declared (at compile time), but the `= function(studentID)` part is also handled at compilation; the association between `getStudentName` and the function is automatically set up at the beginning of the scope rather than waiting for an `=` assignment statement to be executed.

| NOTE: |
| :--- |
| This automatic association of function and variable is referred to as "function hoisting", and is covered in detail in Chapter 5. |

### Sources

So we've identified all five *target* references in the program. The other variable references must then be *source* references (because that's the only other option!).

In `for (let student of students)`, we said that `student` is a *target*, but `students` is a *source* reference. In the statement `if (student.id == studentID)`, both `student` and `studentID` are *source* references. `student` is also a *source* reference in `return student.name`.

In `getStudentName(73)`, `getStudentName` is a *source* reference (which we hope resolves to a function reference value). In `console.log(nextStudent)`, `console` is a *source* reference, as is `nextStudent`.

| NOTE: |
| :--- |
| In case you were wondering, `id`, `name`, and `log` are all properties, not variable references. |

What's the practical importance of understanding *targets* vs. *sources*? In Chapter 2, we'll revisit this topic and cover how a variable's role impacts its lookup (specifically, if the lookup fails).

## Cheating: Runtime Scope Modifications

It should be clear by now that scope is determined as the program is compiled, and should not generally be affected by runtime conditions. However, in non-strict-mode, there are technically still two ways to cheat this rule, modifying a program's scopes during runtime.

Neither of these techniques *should* be used—they're both dangerous and confusing, and you should be using strict-mode (where they're disallowed) anyway. But it's important to be aware of them in case you run across them in some programs.

The `eval(..)` function receives a string of code to compile and execute on the fly during the program runtime. If that string of code has a `var` or `function` declaration in it, those declarations will modify the current scope that the `eval(..)` is currently executing in:

```js
function badIdea() {
    eval("var oops = 'Ugh!';");
    console.log(oops);
}
badIdea();   // Ugh!
```

If the `eval(..)` had not been present, the `oops` variable in `console.log(oops)` would not exist, and would throw a `ReferenceError`. But `eval(..)` modifies the scope of the `badIdea()` function at runtime. This is bad for many reasons, including the performance hit of modifying the already compiled and optimized scope, every time `badIdea()` runs.

The second cheat is the `with` keyword, which essentially dynamically turns an object into a local scope—its properties are treated as identifiers in that new scope's block:

```js
var badIdea = { oops: "Ugh!" };

with (badIdea) {
    console.log(oops);   // Ugh!
}
```

The global scope was not modified here, but `badIdea` was turned into a scope at runtime rather than compile time, and its property `oops` becomes a variable in that scope. Again, this is a terrible idea, for performance and readability reasons.

At all costs, avoid `eval(..)` (at least, `eval(..)` creating declarations) and `with`. Again, neither of these cheats is available in strict-mode, so if you just use strict-mode (you should!) then the temptation goes away!

## Lexical Scope

We've demonstrated that JS's scope is determined at compile time; the term for this kind of scope is "lexical scope". "Lexical" is associated with the "lexing" stage of compilation, as discussed earlier in this chapter.

To narrow this chapter down to a useful conclusion, the key idea of "lexical scope" is that it's controlled entirely by the placement of functions, blocks, and variable declarations, in relation to one another.

If you place a variable declaration inside a function, the compiler handles this declaration as it's parsing the function, and associates that declaration with the function's scope. If a variable is block-scope declared (`let` / `const`), then it's associated with the nearest enclosing `{ .. }` block, rather than its enclosing function (as with `var`).

Furthermore, a reference (*target* or *source* role) for a variable must be resolved as coming from one of the scopes that are *lexically available* to it; otherwise the variable is said to be "undeclared" (which usually results in an error!). If the variable is not declared in the current scope, the next outer/enclosing scope will be consulted. This process of stepping out one level of scope nesting continues until either a matching variable declaration can be found, or the global scope is reached and there's nowhere else to go.

It's important to note that compilation doesn't actually *do anything* in terms of reserving memory for scopes and variables. None of the program has been executed yet.

Instead, compilation creates a map of all the lexical scopes that lays out what the program will need while it executes. You can think of this plan as inserted code for use at runtime, which defines all the scopes (aka, "lexical environments") and registers all the identifiers (variables) for each scope.

In other words, while scopes are identified during compilation, they're not actually created until runtime, each time a scope needs to run. In the next chapter, we'll sketch out the conceptual foundations for lexical scope.

You Don't Know JS Yet: Scope & Closures - 2nd Edition
Chapter 2: Illustrating Lexical Scope
In Chapter 1, we explored how scope is determined during code compilation, a model called "lexical scope." The term "lexical" refers to the first stage of compilation (lexing/parsing).

To properly reason about our programs, it's important to have a solid conceptual foundation of how scope works. If we rely on guesses and intuition, we may accidentally get the right answers some of the time, but many other times we're far off. This isn't a recipe for success.

Like way back in grade school math class, getting the right answer isn't enough if we don't show the correct steps to get there! We need to build accurate and helpful mental models as foundation moving forward.

This chapter will illustrate scope with several metaphors. The goal here is to think about how your program is handled by the JS engine in ways that more closely align with how the JS engine actually works.

Marbles, and Buckets, and Bubbles... Oh My!
One metaphor I've found effective in understanding scope is sorting colored marbles into buckets of their matching color.

Imagine you come across a pile of marbles, and notice that all the marbles are colored red, blue, or green. Let's sort all the marbles, dropping the red ones into a red bucket, green into a green bucket, and blue into a blue bucket. After sorting, when you later need a green marble, you already know the green bucket is where to go to get it.

In this metaphor, the marbles are the variables in our program. The buckets are scopes (functions and blocks), which we just conceptually assign individual colors for our discussion purposes. The color of each marble is thus determined by which color scope we find the marble originally created in.

Let's annotate the running program example from Chapter 1 with scope color labels:

// outer/global scope: RED

var students = [
    { id: 14, name: "Kyle" },
    { id: 73, name: "Suzy" },
    { id: 112, name: "Frank" },
    { id: 6, name: "Sarah" }
];

function getStudentName(studentID) {
    // function scope: BLUE

    for (let student of students) {
        // loop scope: GREEN

        if (student.id == studentID) {
            return student.name;
        }
    }
}

var nextStudent = getStudentName(73);
console.log(nextStudent);   // Suzy
We've designated three scope colors with code comments: RED (outermost global scope), BLUE (scope of function getStudentName(..)), and GREEN (scope of/inside the for loop). But it still may be difficult to recognize the boundaries of these scope buckets when looking at a code listing.

Figure 2 helps visualize the boundaries of the scopes by drawing colored bubbles (aka, buckets) around each:

Colored Scope Bubbles

Fig. 2: Colored Scope Bubbles
Bubble 1 (RED) encompasses the global scope, which holds three identifiers/variables: students (line 1), getStudentName (line 8), and nextStudent (line 16).

Bubble 2 (BLUE) encompasses the scope of the function getStudentName(..) (line 8), which holds just one identifier/variable: the parameter studentID (line 8).

Bubble 3 (GREEN) encompasses the scope of the for-loop (line 9), which holds just one identifier/variable: student (line 9).

NOTE:
Technically, the parameter studentID is not exactly in the BLUE(2) scope. We'll unwind that confusion in "Implied Scopes" in Appendix A. For now, it's close enough to label studentID a BLUE(2) marble.
Scope bubbles are determined during compilation based on where the functions/blocks of scope are written, the nesting inside each other, and so on. Each scope bubble is entirely contained within its parent scope bubble—a scope is never partially in two different outer scopes.

Each marble (variable/identifier) is colored based on which bubble (bucket) it's declared in, not the color of the scope it may be accessed from (e.g., students on line 9 and studentID on line 10).

NOTE:
Remember we asserted in Chapter 1 that id, name, and log are all properties, not variables; in other words, they're not marbles in buckets, so they don't get colored based on any the rules we're discussing in this book. To understand how such property accesses are handled, see the third book in the series, Objects & Classes.
As the JS engine processes a program (during compilation), and finds a declaration for a variable, it essentially asks, "Which color scope (bubble or bucket) am I currently in?" The variable is designated as that same color, meaning it belongs to that bucket/bubble.

The GREEN(3) bucket is wholly nested inside of the BLUE(2) bucket, and similarly the BLUE(2) bucket is wholly nested inside the RED(1) bucket. Scopes can nest inside each other as shown, to any depth of nesting as your program needs.

References (non-declarations) to variables/identifiers are allowed if there's a matching declaration either in the current scope, or any scope above/outside the current scope, but not with declarations from lower/nested scopes.

An expression in the RED(1) bucket only has access to RED(1) marbles, not BLUE(2) or GREEN(3). An expression in the BLUE(2) bucket can reference either BLUE(2) or RED(1) marbles, not GREEN(3). And an expression in the GREEN(3) bucket has access to RED(1), BLUE(2), and GREEN(3) marbles.

We can conceptualize the process of determining these non-declaration marble colors during runtime as a lookup. Since the students variable reference in the for-loop statement on line 9 is not a declaration, it has no color. So we ask the current BLUE(2) scope bucket if it has a marble matching that name. Since it doesn't, the lookup continues with the next outer/containing scope: RED(1). The RED(1) bucket has a marble of the name students, so the loop-statement's students variable reference is determined to be a RED(1) marble.

The if (student.id == studentID) statement on line 10 is similarly determined to reference a GREEN(3) marble named student and a BLUE(2) marble studentID.

NOTE:
The JS engine doesn't generally determine these marble colors during runtime; the "lookup" here is a rhetorical device to help you understand the concepts. During compilation, most or all variable references will match already-known scope buckets, so their color is already determined, and stored with each marble reference to avoid unnecessary lookups as the program runs. More on this nuance in Chapter 3.
The key take-aways from marbles & buckets (and bubbles!):

Variables are declared in specific scopes, which can be thought of as colored marbles from matching-color buckets.

Any variable reference that appears in the scope where it was declared, or appears in any deeper nested scopes, will be labeled a marble of that same color—unless an intervening scope "shadows" the variable declaration; see "Shadowing" in Chapter 3.

The determination of colored buckets, and the marbles they contain, happens during compilation. This information is used for variable (marble color) "lookups" during code execution.

A Conversation Among Friends
Another useful metaphor for the process of analyzing variables and the scopes they come from is to imagine various conversations that occur inside the engine as code is processed and then executed. We can "listen in" on these conversations to get a better conceptual foundation for how scopes work.

Let's now meet the members of the JS engine that will have conversations as they process our program:

Engine: responsible for start-to-finish compilation and execution of our JavaScript program.

Compiler: one of Engine's friends; handles all the dirty work of parsing and code-generation (see previous section).

Scope Manager: another friend of Engine; collects and maintains a lookup list of all the declared variables/identifiers, and enforces a set of rules as to how these are accessible to currently executing code.

For you to fully understand how JavaScript works, you need to begin to think like Engine (and friends) think, ask the questions they ask, and answer their questions likewise.

To explore these conversations, recall again our running program example:

var students = [
    { id: 14, name: "Kyle" },
    { id: 73, name: "Suzy" },
    { id: 112, name: "Frank" },
    { id: 6, name: "Sarah" }
];

function getStudentName(studentID) {
    for (let student of students) {
        if (student.id == studentID) {
            return student.name;
        }
    }
}

var nextStudent = getStudentName(73);

console.log(nextStudent);
// Suzy
Let's examine how JS is going to process that program, specifically starting with the first statement. The array and its contents are just basic JS value literals (and thus unaffected by any scoping concerns), so our focus here will be on the var students = [ .. ] declaration and initialization-assignment parts.

We typically think of that as a single statement, but that's not how our friend Engine sees it. In fact, JS treats these as two distinct operations, one which Compiler will handle during compilation, and the other which Engine will handle during execution.

The first thing Compiler will do with this program is perform lexing to break it down into tokens, which it will then parse into a tree (AST).

Once Compiler gets to code generation, there's more detail to consider than may be obvious. A reasonable assumption would be that Compiler will produce code for the first statement such as: "Allocate memory for a variable, label it students, then stick a reference to the array into that variable." But that's not the whole story.

Here's the steps Compiler will follow to handle that statement:

Encountering var students, Compiler will ask Scope Manager to see if a variable named students already exists for that particular scope bucket. If so, Compiler would ignore this declaration and move on. Otherwise, Compiler will produce code that (at execution time) asks Scope Manager to create a new variable called students in that scope bucket.

Compiler then produces code for Engine to later execute, to handle the students = [] assignment. The code Engine runs will first ask Scope Manager if there is a variable called students accessible in the current scope bucket. If not, Engine keeps looking elsewhere (see "Nested Scope" below). Once Engine finds a variable, it assigns the reference of the [ .. ] array to it.

In conversational form, the first phase of compilation for the program might play out between Compiler and Scope Manager like this:

Compiler: Hey, Scope Manager (of the global scope), I found a formal declaration for an identifier called students, ever heard of it?

(Global) Scope Manager: Nope, never heard of it, so I just created it for you.

Compiler: Hey, Scope Manager, I found a formal declaration for an identifier called getStudentName, ever heard of it?

(Global) Scope Manager: Nope, but I just created it for you.

Compiler: Hey, Scope Manager, getStudentName points to a function, so we need a new scope bucket.

(Function) Scope Manager: Got it, here's the scope bucket.

Compiler: Hey, Scope Manager (of the function), I found a formal parameter declaration for studentID, ever heard of it?

(Function) Scope Manager: Nope, but now it's created in this scope.

Compiler: Hey, Scope Manager (of the function), I found a for-loop that will need its own scope bucket.

...

The conversation is a question-and-answer exchange, where Compiler asks the current Scope Manager if an encountered identifier declaration has already been encountered. If "no," Scope Manager creates that variable in that scope. If the answer is "yes," then it's effectively skipped over since there's nothing more for that Scope Manager to do.

Compiler also signals when it runs across functions or block scopes, so that a new scope bucket and Scope Manager can be instantiated.

Later, when it comes to execution of the program, the conversation will shift to Engine and Scope Manager, and might play out like this:

Engine: Hey, Scope Manager (of the global scope), before we begin, can you look up the identifier getStudentName so I can assign this function to it?

(Global) Scope Manager: Yep, here's the variable.

Engine: Hey, Scope Manager, I found a target reference for students, ever heard of it?

(Global) Scope Manager: Yes, it was formally declared for this scope, so here it is.

Engine: Thanks, I'm initializing students to undefined, so it's ready to use.

Hey, Scope Manager (of the global scope), I found a target reference for nextStudent, ever heard of it?

(Global) Scope Manager: Yes, it was formally declared for this scope, so here it is.

Engine: Thanks, I'm initializing nextStudent to undefined, so it's ready to use.

Hey, Scope Manager (of the global scope), I found a source reference for getStudentName, ever heard of it?

(Global) Scope Manager: Yes, it was formally declared for this scope. Here it is.

Engine: Great, the value in getStudentName is a function, so I'm going to execute it.

Engine: Hey, Scope Manager, now we need to instantiate the function's scope.

...

This conversation is another question-and-answer exchange, where Engine first asks the current Scope Manager to look up the hoisted getStudentName identifier, so as to associate the function with it. Engine then proceeds to ask Scope Manager about the target reference for students, and so on.

To review and summarize how a statement like var students = [ .. ] is processed, in two distinct steps:

Compiler sets up the declaration of the scope variable (since it wasn't previously declared in the current scope).

While Engine is executing, to process the assignment part of the statement, Engine asks Scope Manager to look up the variable, initializes it to undefined so it's ready to use, and then assigns the array value to it.

Nested Scope
When it comes time to execute the getStudentName() function, Engine asks for a Scope Manager instance for that function's scope, and it will then proceed to look up the parameter (studentID) to assign the 73 argument value to, and so on.

The function scope for getStudentName(..) is nested inside the global scope. The block scope of the for-loop is similarly nested inside that function scope. Scopes can be lexically nested to any arbitrary depth as the program defines.

Each scope gets its own Scope Manager instance each time that scope is executed (one or more times). Each scope automatically has all its identifiers registered at the start of the scope being executed (this is called "variable hoisting"; see Chapter 5).

At the beginning of a scope, if any identifier came from a function declaration, that variable is automatically initialized to its associated function reference. And if any identifier came from a var declaration (as opposed to let/const), that variable is automatically initialized to undefined so that it can be used; otherwise, the variable remains uninitialized (aka, in its "TDZ," see Chapter 5) and cannot be used until its full declaration-and-initialization are executed.

In the for (let student of students) { statement, students is a source reference that must be looked up. But how will that lookup be handled, since the scope of the function will not find such an identifier?

To explain, let's imagine that bit of conversation playing out like this:

Engine: Hey, Scope Manager (for the function), I have a source reference for students, ever heard of it?

(Function) Scope Manager: Nope, never heard of it. Try the next outer scope.

Engine: Hey, Scope Manager (for the global scope), I have a source reference for students, ever heard of it?

(Global) Scope Manager: Yep, it was formally declared, here it is.

...

One of the key aspects of lexical scope is that any time an identifier reference cannot be found in the current scope, the next outer scope in the nesting is consulted; that process is repeated until an answer is found or there are no more scopes to consult.

Lookup Failures
When Engine exhausts all lexically available scopes (moving outward) and still cannot resolve the lookup of an identifier, an error condition then exists. However, depending on the mode of the program (strict-mode or not) and the role of the variable (i.e., target vs. source; see Chapter 1), this error condition will be handled differently.

Undefined Mess
If the variable is a source, an unresolved identifier lookup is considered an undeclared (unknown, missing) variable, which always results in a ReferenceError being thrown. Also, if the variable is a target, and the code at that moment is running in strict-mode, the variable is considered undeclared and similarly throws a ReferenceError.

The error message for an undeclared variable condition, in most JS environments, will look like, "Reference Error: XYZ is not defined." The phrase "not defined" seems almost identical to the word "undefined," as far as the English language goes. But these two are very different in JS, and this error message unfortunately creates a persistent confusion.

"Not defined" really means "not declared"—or, rather, "undeclared," as in a variable that has no matching formal declaration in any lexically available scope. By contrast, "undefined" really means a variable was found (declared), but the variable otherwise has no other value in it at the moment, so it defaults to the undefined value.

To perpetuate the confusion even further, JS's typeof operator returns the string "undefined" for variable references in either state:

var studentName;
typeof studentName;     // "undefined"

typeof doesntExist;     // "undefined"
These two variable references are in very different conditions, but JS sure does muddy the waters. The terminology mess is confusing and terribly unfortunate. Unfortunately, JS developers just have to pay close attention to not mix up which kind of "undefined" they're dealing with!

Global... What!?
If the variable is a target and strict-mode is not in effect, a confusing and surprising legacy behavior kicks in. The troublesome outcome is that the global scope's Scope Manager will just create an accidental global variable to fulfill that target assignment!

Consider:

function getStudentName() {
    // assignment to an undeclared variable :(
    nextStudent = "Suzy";
}

getStudentName();

console.log(nextStudent);
// "Suzy" -- oops, an accidental-global variable!
Here's how that conversation will proceed:

Engine: Hey, Scope Manager (for the function), I have a target reference for nextStudent, ever heard of it?

(Function) Scope Manager: Nope, never heard of it. Try the next outer scope.

Engine: Hey, Scope Manager (for the global scope), I have a target reference for nextStudent, ever heard of it?

(Global) Scope Manager: Nope, but since we're in non-strict-mode, I helped you out and just created a global variable for you, here it is!

Yuck.

This sort of accident (almost certain to lead to bugs eventually) is a great example of the beneficial protections offered by strict-mode, and why it's such a bad idea not to be using strict-mode. In strict-mode, the Global Scope Manager would instead have responded:

(Global) Scope Manager: Nope, never heard of it. Sorry, I've got to throw a ReferenceError.

Assigning to a never-declared variable is an error, so it's right that we would receive a ReferenceError here.

Never rely on accidental global variables. Always use strict-mode, and always formally declare your variables. You'll then get a helpful ReferenceError if you ever mistakenly try to assign to a not-declared variable.

Building On Metaphors
To visualize nested scope resolution, I prefer yet another metaphor, an office building, as in Figure 3:

Scope "Building"

Fig. 3: Scope "Building"

The building represents our program's nested scope collection. The first floor of the building represents the currently executing scope. The top level of the building is the global scope.

You resolve a target or source variable reference by first looking on the current floor, and if you don't find it, taking the elevator to the next floor (i.e., an outer scope), looking there, then the next, and so on. Once you get to the top floor (the global scope), you either find what you're looking for, or you don't. But you have to stop regardless.

Continue the Conversation
By this point, you should be developing richer mental models for what scope is and how the JS engine determines and uses it from your code.

Before continuing, go find some code in one of your projects and run through these conversations. Seriously, actually speak out loud. Find a friend and practice each role with them. If either of you find yourself confused or tripped up, spend more time reviewing this material.

As we move (up) to the next (outer) chapter, we'll explore how the lexical scopes of a program are connected in a chain.
