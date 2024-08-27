# Micro

Micro is a small library (~600 lines of code) for creating languages adapted to your needs, allowing you to describe whatever data you'd like in a elegant and concise way. Here's an example of a Micro script designed to express todo lists :

```
todo("Main todo list") {
    #critical "Pay my taxes";
    #important "Clean my room";

    currentShow = "Arcane";
    #important "Watch " + ?currentShow;
};

todo("For my holidays") {
    #critical uppercase("Don't forget the keys !!!");
};
```

Micro scripts are all based on the same generic yet very polymorpheous syntax that you can adapt to your needs. Micro then allows you to quickly parse that language to generate usable JS values out of it.

The tool itself is written in Javascript, and requires you to know this language before you can use it (You can learn Javascript [here](https://www.learn-js.org/)). Furthermore, I will quite often use typescript syntax to describe what types are expected.

Without further ado, let's get started !

## The basics

### A small compiler to begin with 
 
We first need to create what is called an Micro compiler. For that, you need to import the `MicroCompiler` class in a JS file, and to write the following

> myCompiler.js
```js
let compiler = new MicroCompiler(
    [
        { name: "*", arity: 2 },
        { name: "+", arity: 2 },
    ],
    []
    _ => {},
    (_, _, _, _) => {},
)
```

The arguments passed to the `MicroCompiler<T>` (`T` being the output type for the compiler) constructor work as follow :
- A `(OpDeclaration | OpDeclaration[])[]`, to declare the operators our scripts will be able to use. Here, we declare the `+` and `*` operators.
- A `MacroDeclaration[]`, which declares the macros our script will be able to use.
- A `(literal: string) => T` function, called _lift_.
- A `MacroReducer<T>`, which is a function known as the _script macro_.

A `MicroCompiler<T>` is a javascript object meant to take in _scripts_ (`string`) as input, and to output an object of type `T`.

Don't worry about what the arguments mean, we'll come back to it later. Just know that you thereby defined a compiler, that will compile your own version of the Micro language.

You can compile any valid Micro script with `compiler.compile(src)`, that returns a `T`.

### Write an Micro script

Micro is a small language that has a pretty simple, yet very volatile syntax. An Micro script consists solely of a list of _expressions_. We separe expressions by semicolons, the last one being optional, and the script may include comments, wrapped inside a `[[ ... ]]` pair.

This is one of the smallest scripts you could ever write : 

```
1+1; [[ Note that this ';' is optional, as 1+1 is the last expression ]]
```

Every operator you declared previously might be used, but any other operator will induce a syntax error :

```
1-1; [[ This will crash with "Unknown operator '-'." ]]
```

The operators _arity_ requirements must also be met. When declaring `+`, for example, we told that its arity was 2, which means it takes in 2 arguments. If more or less than two arguments are passed to it (we'll see the syntax for that later), the compiler will crash with a syntax error.

Whitespaces and line returns are ignored, so this is perfectly valid :

```
1
+    1;  [[ Works ! ]]
```

### Expressions

Let's talk a little bit more about what an expression is. It consists of things (called operands), bound together by operators. An expression may also be a single operand. 

Operands can be one of the following :
* A literal. This can be a string (characters enclosed inside quotes, such as `"Hello world !"`), a number (`1` or `2.0` for instance), or a name (a letter or underscore, followed by letters, underscores and digits : `x`, `true` and `_list2` are names). Literals are the "elementary blocks" of your script, in that that they can't be split into smaller components.

* Another expression.

* A macro (the syntax of which I will tell you about later)


> `8`, `1+"hi"`, `x*y*z` and `1+(2*"hello")` are all valid expressions.


There's one very, very important thing you need to have in mind before going any further. We distinguish three concepts :  
* An _operation_ is the result of applying an operator to some other things, called operands. `1+1` is an operation, with the operator being `+` and the operands being two `1`.  
* An _expression_, on the other hand, can contain several operations. `1+1*1`, for example.
* A _statement_ is the designation given to a top-level expression, that is an expression that's followed by a semicolon and stands for itself. Syntactically speaking, statements and expressions are exactly the same thing, so we'll stick with "expressions" for both here.

An operation is pretty straightforward to understand when we see it : `1+1` is applying `+` to 1 and 1, period. However, for an expression, the whole thing is thougher. Take `1+1*1`, for example. Here, it's `+` and `*`, so we know, because we're used to them, that `*` comes before `+` and so the expression must be interpreted as `1+(1*1)`. But what if the operators had weird names like `@` and `||` ?

That's why you **always** have to tell yourself the compiler how expressions should be interpreted. Remember our [operator declarations](#a-small-compiler-to-start-with), and how the `*` operator was declared before `+` ? It means that whenever there is an ambiguity, `*` has a higher _precedence_ (since it stands before) than `+` (we also say `*` _binds tighter_ than `+`) and should be executed first. So, when the compiler encounters `1+1*1`, it knows that it means `1+(1*1)`.

> To declare two operators with the same precedence, you would wrap them side by side in a list, like so :

```js
[
    { name: "*", arity: 2 },
    [{ name: "+", arity: 2 }, { name: "-", arity: 2 }]
]
```

> Here `*` binds tighter than `+`, which binds as tight as `-`.

If several operators with the same precedence are encoutered in a single expression, the one at the left executes first, then the second, and so on. `1+1+1+1` therefore executes as `((1+1)+1)+1`.

> If you need to override precedence and execute `+` before `*` in an expression, you need to enclose it in parentheses : `(1+1)*1`.

### The long-awaited list of keywords

Now for the the list of all keywords of the language : 

No keywords, actually ! Instead, we got hash operators. These are operators that start with `#` (hence the name), followed by one or more letters : `#hello` or `#world` for instance.  These are used just like normal, symbol-based operators, and allow for nicer syntax when needed : `elem >?> myList` is pretty obfuscated, while `elem #in myList` is crystal clear !

> If you wonder, here are the symbols that can be used in symbolic operators : `&|~^@=+'-$*%!/:.,?!<>`. Any combination of such symbols is a valid operator.

I repeat one more time, because it's quite important and easy to miss : **hash operators do not differ from standard operators like `+`, and it's only a matter of stylistic preference.**

### Macros

I said macros are a type of operand, but I didn't explain how to write one. Let's fix that.

For now, you can view a macro as some sort of block of code, that groups together several expressions.
The general syntax for a macro is the following (plus several tweaks we'll talk about later) :

```
macroName(arg1; arg2; ...; argn;) {
    statement1;
    ...
    statementn;
}
```

Which is much more complicated than anything we've seen until there, but bear with me.  
The thing between parentheses is called the argument list (or head), the thing between curly brackets is known as the body.

`arg1` (etc) as well as `statement1` (etc) should be valid expressions. For both lists, the final semicolon is optional.
If the head is empty, parentheses can be dropped, so this is valid syntax :

```
macroName { 
    [[ Some expressions here ]]
}
```

However, even if there are no expressions inside the body, the curly brackets aren't optional.
Note that the formatting (the line return after the first curly bracket, the indent, etc) is purely a question of style, and that this is fine as well :

```
macroName(
    arg1; 
    arg2;
    arg3;
) { statement1; statement2 }
```

`macroName` must have been declared through [the second argument of the compiler](#a-small-compiler-to-start-with). For example, if you want to declare a macro called `doSomething` that takes in one argument, you can replace the empty list with : 

```js
[{ name: "doSomething", arity: 1 }]
```

With `arity: 1` meaning the same as for operators : our newly created macro takes one single argument. An arity check is automatically performed everytime this macro is encountered, so this would crash :

```
[[ Fails with "Macro doSomething expects between 1 and 1 arguments, got 2." ]]
doSomething (x, y) {

};
```

You can add some flexibility to the number of arguments accepted by passing in a two-sized list instead : `arity: [a,b]` means the macro accepts between a and b arguments, inclusive at both endpoints.


Always remember that macros are still expressions, and, as such, this is valid syntax :

```
[[ Multiplying 1 by the result returned by myMacro ]]
1 * myMacro(...) { ... };

[[ And to remind you hash operators exist : ]]
hello #is mySecondMacro(...) { ... };
```

## Syntax versus semantics

You may have noticed that until now I only talked about what you could _write_, not about what it actually _does_.  The first thing is usually called the syntax of a program, the second its semantics.

This is actually what makes Micro stands apart from other languages. It's only a syntax. No semantics. If you compile a script, it actually does... nothing !

```js
compiler.compile("1+1*1;")
// Prints nothing, returns void.
```

> "But... wait ! I do want my code to do something, somehow !".   

Me too ! That's where it gets interesting. Remember the two other arguments we passed to the compiler ? It's time to actually understand what they do.

### Go touch some grass

First, let's talk about trees. 
In a typical programming language, before doing anything with a script, said script is usually translated in a much more practical form, called an _Abstract Syntactic Tree_ (or AST), and Micro is no exception to that.

Behind this daunting name hides a rather simple thing. Here's what it means. When you write `1+(2*3);`, for example, it's translated to something that roughly looks like `{ op: '+', args: [1, { op: '*', args: [2, 3] }] }`. It appears uglier, but it's now standard javascript and, as such, it's much easier for a program to manipulate it.

> The name comes from the fact you can represent it like so :

```
    +
   / \
  1   *
     / \
    2   3
```

> Which, if flipped, looks (somewhat, don't expect an computer scientist to know what a tree is) like a tree.
                     
In standard languages, it's, more often than not, **not** visible by the user, and directly used by the compiler to perform actions. In Micro, however, once the ASTs for the whole script are obtained (we say the script has been _parsed_), it's passed down to **you**, so you can do whatever you want with it.

> You don't need to understand how ASTs are implemented for now, but we'll talk about it later.

### Macros, again

And that's where macros kick in ! Conceptually, macros are functions that are meant to evaluate parsed code, and return a standard JS value of type `T`. A macro is made up of two things : a `MacroDeclaration`, which tell what the macro looks like, and something called a `MacroReducer<T>`, which tells what the macro does.

A `MacroReducer<T>` is a function that takes in four arguments :
* An context, of type `Context<T>`,
* The body of the macro as a list of `AST`s. Each AST of the list is one expression inside your script. So if your macro contained `1+1; 2+2; 3+3;`, it will have length of 3. 
* The arguments to the macro as a list of `AST`s too ; same here, each argument is an `AST` on its own.
* An empty object called `limbs`. Don't worry about that last one for now.

When you write and compile a script, what actually happens is that it gets passed to the [script macro](#a-small-compiler-to-start-with) just like it was its body, with no arguments, so that it gets called like this : `scriptMacro(context, script, [], {})`. And that script macro gets to do whatever it wants with it.

The most important thing here is `context`. It has three members : `ev`, `operators` and `macros`, but we'll solely focus on `ev` for now.

The `ev` function pretty much does all the job and allows, as its name suggests, to _evaluate_ `AST`s.
Suppose we just want to evaluate sequentially every expression inside your script. We would write :

```js
let scriptMacro = ({ ev }, body, args, limbs) => {
    // See below for what `ops` and `macros` are here
    let ops = []
    let macros = []

    // We iterate throught each expression
    for (let expr of body) {
        // And evaluate it !
        ev(expr, ops, macros)
    }

    // There's no return value, meaning this macro will return undefined
}
```

And that's done ! Well, almost. Remember how I said that the Micro compiler has absolutely no idea about semantics ? That still holds. So how would it know that `+` adds two numbers and `*` multiplies them ?

By passing operators (of type `MapLike<OperatorReducer<T>>`, which are the implementation counterpart to the `OpDeclaration`s you [passed to the compiler](#a-small-compiler-to-start-with)) as the second argument to `ev`, you implement the actual behavior of such operators.
Let's declare the `+`, for instance :

```js
// The operator reducer. It takes its arguments as a T[], i.e as an array of evaluated (not AST) js objects.
let plusOp = ([a,b]) => a+b 
```

Now let's pass that operator in :


```js
let scriptMacro = ({ ev }, body) => {
    // We can now use '+' !
    let ops = { "+": plusOp }
    let macros = {}

    for (let expr of body) {
        // Here, the body is evaluated, and every time a '+' is encountered, 
        // it will add its operands together !
        ev(expr, ops, macros)
    }
}
```

Notice how operators don't need to evaluate their arguments before adding them ? That's because the compiler automatically `ev`s any argument to an operator. The reducer of an operator therefore takes a `T[]`, and not a `AST[]`.

### More. Macros.

There's still that pesky ```let macros = {}``` line, which is later passed to `ev` along with operators. What is it supposed to mean ?  

Macros are reaaaally powerful, so it would be a shame to just use them to evaluate the script. I gave you the macro syntax :

```
macroName (arg1; ...; argn;) {
    statement1;
    ...;
    statementn;
}
```

When you ask the compiler to evaluate an AST representing a macro, it searches up for the corresponding macro, and evaluates it by calling its reducer with `AST[]`s corresponding to its body and head.

The return type of the macro reducer is used as the return type of the whole macro expression. 

Back to the `macros` passed to `ev`. It is simply meant to give the compiler the implementation of macros that you will use in your script !

For example, let's write a macro that acts like a JS `if` : it should evaluate its single argument, and evaluate its body only if it is truthy.

```js
// We declare the macro :

let compiler = new MicroCompiler(
    [...], // Operator declarations
    [{ name: "if", arity: 1 }], // Macro declarations
    ...
)

let ifMacro = ({ ev }, body, args) => {
    // args here always have a length of 1 because of the arity check.
    let [condition] = args

    // If `condition` (which is an AST, so it needs to be evaluated) evaluates to true
    if (ev(condition)) {
        // Evaluate body
        for (let expr of body) {
            ev(expr)
        }
    }
}
``` 

We need to pass it down just like `+` when evaluating things inside our script macro :


```js
let scriptMacro = ({ ev }, body, args, limbs) => {
    let ops = { "+": plusOp }
    // We can now use 'if' !
    let macros = { "if": ifMacro }

    for (let expr of body) {
        ev(expr, ops, macros)
    }
}
```

We can then use it inside our script like this : 

```
[[ This will evaluate ]]
if (1) {
    1+1;
    2+2;
};

[[ This won't  ! ]]
if (0) {
    0+0;
}
```

And it will behave like a "real" if. Pretty neat, right ?

> If you try to compile the script above, you will get a weird error : `Unknown operator '#number'`. We'll solve that in a minute.

To summarize, the script is handed down to the global script macro, which may, or may not, evaluate its statements using `ev`. When evaluating an expression :
* If it's an operation, then it calls the according operator reducer with its arguments **already evaluated**. 
* If it's a macro, it calls the according macro reducer with its arguments **as ASTs**. It is up to the macro to decide what to do with these.

And there's nothing more to know ! Micro scripts are exclusively constructed with operators and macros, which are then evaluated when needed.

> "But inside the `if` macro, `ev` is called without any operators or macros passed down. Why ?"

This brings us to the next point : scopes.

### Scoping

This one is pretty simple : A macro inherits, inside of its body, any operators and macros that were already defined outside (we call these _ambient_ operators and macros). So that means that `if` doesn't need to pass again `+` and `if` as arguments to `ev`, and that every operator/macro passed down by the script macro is available globally. Every operator/macro passed to `ev` will be available inside the `if` body, and inside it only.

If an operator/macro **that already exist outside** of the `if` is passed to `ev`, that local implementation will override the ambient one (here again, inside the `if` body only).

Suppose now we have an operator, or macro, we defined outside, and that we want to slightly tweak its behavior.
For example, we want to create a `verbose` macro that prints out "Adding a and b !" every time a `+` is encountered.

We would do it like this :

```js
// Declaration
{ name: "verbose", arity: 1 }

let verboseMacro = ({ ev }, body) => {
    // We override the regular behavior of +
    let localPlusOp = ([a,b]) => {
        console.log(`Adding ${a} and ${b} !`)
        return a+b
    }

    for (let expr of body) {
        ev(expr, { "+": localPlusOp })
    }
}
```

That may seem to work, but there is a problem. If we nest a arbitrary number of `verbose` blocks one inside another, we might expect any `+` inside to print out that much "Adding a and b !". In fact, it won't, because the `+` that gets used is the `+` of the innermost `verbose` macro that just prints it once. Luckily for us, the first `context` argument of macros has that `operators` field containing the reducers of the ambient operators. So we can change our implementation to this instead :

```js
let verboseMacro = ({ ev, operators }, body) => {
    let ambientPlusOp = operators["+"]
    let localPlusOp = ([a,b]) => {
        console.log(`Adding ${a} and ${b} !`)
        return ambientPlusOp([a, b])
    }

    for (let expr of body) {
        ev(expr, { "+": localPlusOp })
    }
}
```

And it will work as intended. The `context` argument also has a `macros` field containing the ambient macros if you need them.

Generally speaking, it's a good practice to :
* Pass in every operator at script level, even it's a dummy implementation that always throw
* Delegate operator implementation to the ambient operator if you don't know what to do with its values. That way, nested macros don't interfere with one another. Eventually, if no one knows what to do, it will be passed to the script macro's operator implementation, and, if needed, will crash.

Same applies for macros : pass in every declared macro at script level, then delegate instead of throwing.

### Micro knows nothing about numbers

Let's solve the weird error about the `#number` operator we mentioned above. The title actually speaks for it : knowing what to do with numbers would be semantics, and Micro don't know anything about semantics !

Numbers inside a Micro script are, in fact, syntactic sugar for something called literals. 

> A literal don't have any equivalent in a script, so I'll use backticks (`) to write them, even if it's no actual syntax.

Literals are, like the name suggests, literal excerpts of the source code. They only exist as ASTs and have no direct in-script equivalent.

What happens when you write `1`, for instance, is that it implicitely generates an AST as if you applied operator `#number` to the literal \`1\`. 

> Once again, you don't need to know how literals are actually represented as ASTs for now. All you need to know is that when you type `1`, an AST of this form is generated for you :
```
        #number
           |
          `1`
```


However, the literal \`1\` is, as you can see, an AST, and operators expect **evaluated ASTs** as arguments. It therefore mean we must be able to evaluate the literal before passing it to `#number` for it to actually produce a true number.

This is where the [lift function](#a-small-compiler-to-start-with) comes into play ! Every time a literal must be evaluated, the compiler calls the lift function (which lifts, or promotes, the literal to a true value to be passed around) with a string, representing the literal, as its argument, and the result is understood as evaluating the literal.

Suppose we now want to actually implement numbers.
* We have to replace the dummy `_ => {}` lift function in our compiler by a true lift function. Since lift takes a string  representing a literal as an argument, and that evaluating a literal can, without too much imagination, return that same string, we'll just return the literal itself : `literal => literal`. So passing in \`1\` (as "1") will return "1".
* We have to implement the `#number` operator, which you will then have to pass to `ev` along with `+`. We will juste parse that "1" to 1.0 with the JS `parseFloat`.

```js
// Don't forget to declare '#number' with arity 1 to the 
// compiler, and to pass down 
// { '+': plusOp, '#number': numberOp } inside the script macro
// reducer

// This will for the implementation !
let numberOp = ([s]) => parseFloat(s)
```

And this will work just like you expected it to !

> While this may seem like overcomplicating things, this is a deliberate choice made to strictly enforce syntax and semantics separation. Besides, it actually offers tangible benefits, which we'll study later.

### Micro knows nothing about strings and names either

I think you got it : when Micro comes across a string, it calls the special operator `#string` with a literal representing the string (for "Hello world !", it would be \`Hello world !\`) as its sole operand.

Same goes with names, such as `foo` : the `#name` operator is called with \`foo\` as its only operand.

You can declare and implement the `#string` and `#name` operators depending on your needs ; without them, strings and names respectively are not usable inside your script.


## Let's practice I

With everything we've learnt until now, we can already implement a calculator.
The requirements for it are the following : 
* Numbers are, of course, functionnal
* It will have operators '+', '-', '*', '/', working like you'd expect them to.
* Every statement will print out its result
* There will be an `arity: 1` macro called `chain`.

The `chain` macro should do the following : using its argument as the initial `value`, it evaluates the first expression of its body, then binds the result to `value`. It then evaluates the second expression, binds it to `value`, and so on. At the end, it returns `value`. As an example :

```
3 + chain (7) {
    1+value;         [[ Returns 8, value being 7 ]]
    5*value + 3;     [[ Returns 43, value being 8 ]]
    value-4        [[ Returns 39, value being 43 ]]
};
[[ Final result is thus 42. ]]
```

We will use an `error(msg: string)` function, defined by `let error = msg => { throw msg }`

Let's begin by declaring all these things :

> JS
```js 

let compiler = new MicroCompiler(
    [
        [{ name: "#number", arity: 1 }],
        [{ name: "#name", arity: 1 }], // We'll need it for chain
        [{ name: "*", arity: 2 }, { name: "/", arity: 2 }],
        [{ name: "+", arity: 2 }, { name: "-", arity: 2 }],
    ],
    [
        { name: "chain", arity: 1 },
    ],
    literal => literal,
    scriptMacro, // See below
)
```

Then, let's implement our operators :

```js
let plusOp = ([a,b]) => a+b
let minusOp = ([a,b]) => a-b
let timesOp = ([a,b]) => a*b
let divideOp = ([a,b]) => a/b
let numberOp = ([s]) => parseFloat(s)

// Inside the script, but ouside a `chain` block, names have no meaning, so we throw.
let nameOp = ([name]) => error(`Syntax error : ${name}.`)
```

Let's write the chain macro. It may seem like a complicated one, but it actually isn't at all !

```js
let chainMacro = ({ ev, operators }, body, [value]) => {
    // This is intended to override the global #name 
    // implementation. The #name operator
    // will be called everytime a name is encountered,
    // but the only name that should do something is "value".
    // When passed in any other name, we delegate.
    let nameOp = ([name]) => {
        if (name === "value") return value
        else return operators["#name"]([name])
    }

    let ops = { "#name": nameOp }
    let macros = {}

    for (let expr of body) {
        value = ev(expr, ops, macros)
    }

    return value
}
```

It's time to write the script macro. We want to simply evaluate and print everything : 

```js
let scriptMacro = ({ ev }, body) => {
    let ops = { 
        "+": plusOp,
        "-": minusOp,
        "*": timesOp,
        "/": divideOp,
        "#number": numberOp,
        "#name": nameOp
    }

    let macros = { 
        "chain": chainMacro
    }

    body.forEach(expr => console.log(ev(
        expr, 
        ops, 
        chainMacro
    )))
}

```

Aaaand done ! When using `compiler.compile` to compile the following script :

```
2*8;
3/4;
3 + chain (7) {
    1+value;      
    5*value + 3;  
    value-4
}
```

it should print out 16, 0.75, and 42. You can, if you like, write other macros and operators (`**`, for example) to improve that small calculator of yours !

The whole script, in case you need it :

> JS script

```js
let error = msg => { throw msg }

let plusOp = ([a,b]) => a+b
let minusOp = ([a,b]) => a-b
let timesOp = ([a,b]) => a*b
let divideOp = ([a,b]) => a/b
let numberOp = ([s]) => parseFloat(s)

let nameOp = () => error("Can't use #name here !")

let chainMacro = ({ ev, operators }, body, [value]) => {
    let nameOp = ([name]) => {
        if (name === "value") return value
        else operators["#name"]([name])
    }

    let ops = { "#name": nameOp }
    let macros = {}

    for (let expr of body) {
        value = ev(expr, ops, macros)
    }

    return value
}

let scriptMacro = ({ ev }, body) => {
    let ops = { 
        "+": plusOp,
        "-": minusOp,
        "*": timesOp,
        "/": divideOp,
        "#number": numberOp,
        "#name": nameOp
    }

    let macros = { 
        "chain": chainMacro
    }

    body.forEach(expr => console.log(ev(
        expr, 
        ops, 
        macros,
    )))
}

let compiler = new MicroCompiler(
    [
        [{ name: "#number", arity: 1 }],
        [{ name: "#name", arity: 1 }],
        [{ name: "*", arity: 2 }, { name: "/", arity: 2 }],
        [{ name: "+", arity: 2 }, { name: "-", arity: 2 }],
    ],
    [
        { name: "chain", arity: 1 },
    ],
    literal => literal,
    scriptMacro,
)

```


## Operators

> "Why'd you write another chapter entirely focused on operators ? Haven't we covered most of it already ??"

Nope we don't. Actually, we didn't talk a lot about operators until now. We'll cover many useful syntactic features here, which are very important to code in Micro.

Did you notice how `Operator` takes in an `arity` argument too ? Until there, we exclusively used _binary operators_ (which means "operators with an arity of two"), but these are not the only ones to exist.

> Remember that operators always need to update their arity accordingly to be used in the ways that will follow !

### 2 is good, but 1 is better.

We can use operators in a unary way by prefixing an expression with said operator. Operator precedences still apply, but only at the right of the operator, as unary operators automatically bind tighter than everything on their left side.

So `1 * +1` is parsed as `1*(+1)` since unary operators bind tighter that everything there is on their left, but `+1 * 1` is parsed as `+(1*1)` since precedence on the right side works normally and `*` binds tighter than `+`. Last but not least, things like `#op1 #op2 expr` will always evaluate as `#op1 (#op2 expr)`, independently of precedences. This may seem weird, but it's actually pretty common behavior that is seen in many other languages.

> If this confuses you, don't think too much about it ; it is intended to conform to your intuition.

Notice that the space between `*` and `+` in the first expression is mandatory : `1*+1` would be parsed as the operator `*+` applied to two ones. 

This, combined with hash operators, allows for rather cool syntax reminiscent of python 2.0, with directives-like expressions such as `#print 1*2`. 

> Getting rid of parentheses around `1*2` was achieved by making that `#print` operator low precedence. 

### Ternary (and more) operators

You can't pass more than two operands simultaneously to an operator with the syntax bits we've seen until there. 
I therefore introduce to you the _pack_ syntax.  

The pack syntax is rather simple and comes from the fact it allows to "pack" operands for a single operator. You just need to wrap an expression containing only one type of operators inside brackets. For example :

```
[[ This will apply + to operands 1, 2, 3 simultaneously ]]
[[ Writing 1+2+3 without the brackets would have resulted in + applying to (1+2) and 3 instead. ]]

[1 + 2 + 3];
```

This is called a ternary operator (arity of 3), but you can use the pack syntax to give whatever number of arguments you like, including 2. Also note that an optional trailing operator may be added : 

```
[[ Still valid and does the same thing ]]

[1 + 2 + 3 +]; 
```

This allows, among other things, to use the pack syntax for suffix unary operators : `[1+]` is exactly the same as `+1`.

There's several little catches you might want to be aware of. First, a pair of brackets with only an operand inside or, even worse, nothing inside, are both illegal, since the compiler won't know what operator to use :

```
[]; [[ Whoops ! ]]
[0]; [[ Doesn't work either ]]
```

For the same reason, mixing operators is forbidden : 

```
[ 1 + 2*3 + 4 ]  [[ Syntax error : it's ambiguous whether you want to pack on * or + ]]
[ 1 + (2*3) + 4 ] [[ Works fine. ]]
```

Second, due to the syntax of comments, you must be cautious when nesting one bracket expression inside another. Anytime `[[` is encoutered, the compiler enters comment mode until it finds the matching `]]`. So if you want to write something like this :

```
[[ 1 + 2 + 3 ] + 4 + 5 + 6]
```

You actually need to space out the second bracket :

```
[ [1 + 2 + 3] + 4 + 5 + 6 ]
```

Third and last on our list, you can't make a bracket expression start with an unary operation. Suppose we have declared operators `!`, `&&`, `||` in that order (standard JS boolean operations). If we write :

```
[!false || true]  [[ Wrong but no syntax error, see below ]]
[(!false) || true] [[ Works like you'd expect ]]
```

The first expression would actually yield an AST applying `!` to `(false || true)`, although `!` binds tighter than `||` ! Which brings us to our next point : _prefix pack syntax_ (standard pack syntax being refered to as _infix pack syntax_ when there's an ambiguity).

If the left bracket is immediately (there may be some whitespaces) followed by an operator, it is parsed in prefix mode : every expression (separated by semicolons, last one optional, you know the deal by now) inside the brackets is parsed, then that operator is applied to all of them at once. Writing `[+ 1; 2; 3]` therefore is the exact same as `[1 + 2 + 3]` ! And that's also why `[!false || true]` yields `!(false || true)` : `!` is encountered, the compiler enters prefix mode, `false || true` is parsed, then passed to `!`.

> `[+1]` is by that last rule valid syntax and yields the same as `+1` and `[1+]` ! Down to the bone, choosing which one you want to use mostly is a matter of style, as these three truly are the exact same.

### Nullary operators

Have you ever tried applying an operator to nothing ?

```
#pi;  [[ With #pi an operator taking no operands and returning 3.1415... ]]
```

This will crash, actually, with `Unexpected character ';'` or something similar. This is because it sees `#pi` as an unary operator here, and when it encounters the semicolon, it doesn't understand why there isn't any operand following it.

To call an operator with 0 arguments, you thus need to enclose it in square brackets : `[#pi];` is correct and yields the expected result. 

> While this may seem like somewhat new syntax, this is actually the prefix pack operator doing its job !

### Special operators : `#call` and `#index`

Ever wondered why the `{}` following a macro are not optional ?   
That's actually because they are needed to disambiguate macro syntax from a special operator called... well, `#call`. When an expression of the form `callee(arg1; ...; argn;)` (lAsT sEmIcOLoN OpTiONal) is encountered, `called` as well as `arg1`, ..., `argn` are parsed, then passed together to the special `#call` operator in that order.

> `callee` can be any expression, and, in particular, it can be a name. `myMacro()` would therefore be parsed as applying `#call` to name `myMacro`, which likely isn't what you want here. Adding `{}` at the end forces the compiler to parse that as a macro, since `(myMacro()) {}` doesn't make sense.

This is actually very cool syntactic sugar !

```
myFunction(1; 2; 3);  [[ Same as [#call myFunction; 1; 2; 3] ]]
```

Note that semicolons tell arguments apart from each other, not comas. Comas are regular operators, and `f(1,2,3)` is actually `[#call f; (1,2,3)]`. This might take some time for you to get used to it.

> Using `#call` without that added syntactic sugar is not considered a bad practice, and you can write `[#call f]` or `#call f` instead of `f()` if you prefer. Writing `[f #call 1 #call 2]` for `f(1; 2)`, on the other hand, is very unclear and therefore not recommanded.

Same goes for `x[arg1; ...; argn;]` (last semicolon optional) with the `#index` operator : it will be translated to `[#index x; arg1; ...; argn]`, so that you can write `myArray[0]` to mean `myArray #index 0`. Neat, isn't it ?

> Don't forget that `#call` and `#index` arity is one more than what you would expect, because the thing that's called/indexed is itself an argument to it. Declaring `#index` with an arity of 1 would actually only allow expressions of the form `x[]` !

Such expressions can be nested : `x[0][1]` is `(x #index 0) #index 1`, `f()()` is `[#call [#call f]]`, and `x[0]()` is `#call (x #index 0)`. If you wish to call/index complex expressions, wrapping them inside parentheses works just fine : `(my #complex expression)[index]`

## Let's practice II

We will now upgrade our calculator using what we've just learned.  

First, a word about typing. Until now, we've manipulated values pretty recklessly, assuming every value was of the correct type. And it always was, since we were only passing numbers around.

But suppose we have to handle numbers AND booleans. It would be pretty annoying to test for a value's type every time we have to use it, so instead we prefer manipulating values of the shape `type MicroValue = { type: string, value: any }`, with type telling us what to expect. With that in mind, let's rewrite some of our code. We will add these to help us :

```js
// Our types
const nothing = "nothing"
const name = "name"
const literal = "literal"
const string = "string"
const number = "number"
const bool = "bool"

// Manipulate typed values
let create = (type, value) => ({ type, value })
let expect = (expected, { type, value }) => 
    type === expected 
    ? value 
    : error(`Expected a value of type ${expected}, got ${type}`) 
```

And update [our previous code](#lets-practice-i) :

```js
let error = msg => { throw msg }

// Creates a new function that checks that its operands are numbers, applies f, and returns a typed number.
let numberF = f => (args) => create(
    number,
    f(args.map(x => expect(number, x))),
)

let plusOp = numberFunc(([x,y]) => x+y)
let minusOp = numberFunc(([x,y]) => x-y)
let timesOp = numberFunc(([x,y]) => x*y)
let divideOp = numberFunc(([x,y]) => x/y)
let numberOp = ([s]) => create(number, parseFloat(expect(s, literal)))

let nameOp = () => error("Can't use #name here !")

let chainMacro = ({ ev, operators }, body, [initialValue]) => {
    let nameOp = ([name]) => {
        if (expect(literal, name) === "value") return value
        else operators["#name"]([name])
    }

    let ops = { "#name": nameOp }
    let macros = {}

    let value = ev(initialValue)
    for (let expr of body) {
        value = ev(expr, ops, macros)
    }

    return value
}

let scriptMacro = ({ ev }, body) => {
    let ops = { 
        "+": plusOp,
        "-": minusOp,
        "*": timesOp,
        "/": divideOp,
        "#number": numberOp,
        "#name": nameOp
    }

    let macros = { 
        "chain": chainMacro
    }

    body.forEach(expr => console.log(ev(
        expr, 
        ops, 
        macros,
    )))
}

let compiler = new MicroCompiler(
    [
        [{ name: "#number", arity: 1 }],
        [{ name: "#name", arity: 1 }],
        [{ name: "*", arity: 2 }, { name: "/", arity: 2 }],
        [{ name: "+", arity: 2 }, { name: "-", arity: 2 }],
    ],
    [
        { name: "chain", arity: 1 },
    ],

    // the lift now tells that its value is of type literal
    literalValue => create(literal, literalValue),
    scriptMacro,
)
```

Now, operators check for their operands to be of the correct type before proceeding.

Let's discuss a bit what we want to implement :
* We will remove the old behavior of printing out every expression and replace it with a new `#print` operator.
* Strings
* Unary `+` and `-`,
* A new nullary `#PI` operator that returns `Math.PI`,
* A `<=` operator returning instances of the new bool type : `true` if its operands are well-ordered, else `false`,
* `true` and `false` literals,
* A new ternary `#ifthenelse` operator,
* Functions : `exp`, `cos` and `sin`.

Let's write the `#print`, with a `[0, Infinity]` arity to allow to print any number of argument :

```js
// Declaration to add to the compiler
{ name: "#print", arity: [0, Infinity] }

// We don't care about the types, just log it to the console.
let printOp = args => {
    console.log(...args.map(arg => arg.value))
    return create(nothing, undefined)
}
```

To implement strings is to implement `#string`. Let's do that :

```js
// Declaration
{ name: "#string", arity: 1 }

let stringOp = ([s]) => {
    let source = expect(literal, s)
    return create(string, source)
}
```

We update `+` and `-`'s arity to accept 1 to 2 arguments, and rewrite their definitions :

```js
// These are the new declarations
{ name: "+", arity: [1,2] }
{ name: "-", arity: [1,2] }

let plusOp = numberFunc(([x,y]) => y === undefined ? x : x+y)
let minusOp = numberFunc(([x,y]) => y === undefined ? -x : x-y) 
```

Then we declare `#PI` :

```js
// The declaration
{ name: "#PI", arity: 0 }

let piOp = numberFunc(() => Math.PI)
```
 
`<=` ! We will set its arity to `[2, Infinity]`, to compare any number of operands :

```js
// Declaration
{ name: "<=", arity: [2, Infinity] }

let ltOp = args => {
    let correctArgs = args.map(arg => expect(number, arg))
    for (let i=1; i<correctArgs.length;i++) {
        if (correctArgs[i-1] > correctArgs[i]) {
            return create(bool, false)
        }
    }
    return create(bool, true)
}
```

More interesting : `true` and `false` literals. This is where delegating the actual implementation of names to `#name` really shines, because we just need to modify this operator to achieve that :

```js
let nameOp = ([value]) => {
    let l = expect(literal, value)
    switch (l) {
        case "true":
            return create(boolean, true)
        case "false":
            return create(boolean, false)
        default:
            // There is no other literal, so let's just crash.
            error(`Syntax error : ${l}.`)
    }
}
```

The `#ifthenelse` operator is a breeze. It should evaluate its first argument. If it's `true`, return its second argument, else the third one.

```js
// Declaration
{ name: "#ifthenelse", arity: 3 }

let ifThenElseOp = ([condition, ifValue, elseValue] )=> {
    return expect(boolean, condition)
        ? ifValue
        : elseValue
}
```

And now, functions ! That's the most interesting part.  
We will of course use the `#call` operator, and also the `#name` one. Let's implement it. First, we modify the `#name` operator **again** to return a value of type name in case it's not `true` or `false` : 

```js
let nameOp = ([s]) => {
    ...
        default:
            // Previously used to crash !
            return create(name, l)
    }
}
```

Then, let's declare and implement `#call` :

```js
// Declaration
{ name: "#call", arity: 2 }

let callOp = ([f, arg]) => {
    // Calling a number doesn't make sense, so we tell we 
    // expect the callee (to remind you, the thing that 
    // comes here : >>f<<(arg)) to be a name
    let functionName = expect(name, f)
    let argValue = expect(number, arg)

    let functionImpl = { 
        "sin": Math.sin,
        "cos": Math.cos,
        "exp": Math.exp
    }[functionName] ?? error(`Unknown mathematical function ${functionName}.`)

    return create(number, functionImpl(argValue))
}
```

One last thing. Remember that we defined another `#name` operator inside the `chain` macro that delegates to the ambient one whenever a name is passed in that is not `value` ? This means our `true` and `false` literals still work, even inside a chain macro, and we didn't even need to do anything for that to happen !


Here's operators precedence, but first try to guess for yourself what it will be, to see if you grasped how it affects the syntax, knowing that, by convention, special implicit operators like `#name` and `#index` have the highest precedence, and that operators for which precedence doesn't matter (nullary and n-ary with n greater than 3, since they will always be wrapped inside brackets) have the lowest.
Operators we defined are : `#call`, `#string`, `#number`,`#name`, `+`, `-`, `*`, `/`, `#print`, `#ifthenelse`, `<=`, `#PI`.

Think about it  
Think about iiiit

.  
.  
.  
.  
.  
.  

Here's the solution :  
`#name`, `#call`, `#string`, `#number`  
`*`, `/`  
`+`, `-`  
`<=`  
`#print`  
`#PI`, `#ifthenelse`

Don't forget that you can make multiple operators have same precedence by putting them together in a list.

A small script to demonstrate :

> Micro Script

```
#print 1*2+3; [[ This fortunately still works ]]
2+0; [[ This will silently evaluate ]]

[[ This demonstrates why infix pack notation is cool ]]
[[ Here we test simultaneously 1 <= 2 AND 2 <= 3 ]]
if ([1 <= 2 <= 3]) {
    #print cos([#PI]);
    #print -3*-3 <= 10;
    #print chain(2) { 
        value*value; 
        value*value; 
        value*value;
        [[ Notice how literals still behave correctly inside of chain ]]
        value + [#ifthenelse true; 1; 0]
    }
};

if (false) {
    #print "This will never print.";
};
```

Output should be 5, -1 (because cos(pi) is -1), true and 257.
If you see an error that look like this : `'if' macro has no 'if' limb` (which I did at the time), it probably means you forgot the semicolon at the end of your first `if`. We'll explain what this error means a bit later.

You can play around, and, especially, try to write things you know are wrong, to see if (and how) it fails.

The complete JS file is here :

> JS

```js
const nothing = "nothing"
const name = "name"
const literal = "literal"
const string = "string"
const number = "number"
const bool = "bool"

let create = (type, value) => ({ type, value })
let expect = (expected, { type, value }) => 
    type === expected 
    ? value 
    : error(`Expected a value of type ${expected}, got ${type}`) 

let error = (msg) => { throw msg }
let numberFunc = 
f => (args) => create(
    number,
    f(args.map(x => expect(number, x))),
)

let printOp = args => {
    console.log(...args.map(arg => arg.value))
    return create(nothing, undefined)
}

let stringOp = ([s]) => {
    let source = expect(literal, s)
    return create(string, source)
}

let piOp = numberFunc(() => Math.PI)

let ltOp = args => {
    let correctArgs = args.map(arg => expect(number, arg))
    for (let i=1; i<correctArgs.length;i++) {
        if (correctArgs[i-1] > correctArgs[i]) {
            return create(bool, false)
        }
    }
    return create(bool, true)
}

let ifThenElseOp = ([condition, ifValue, elseValue])=> {
    return expect(boolean, condition)
        ? ifValue
        : elseValue
}

let nameOp = ([value]) => {
    let l = expect(literal, value)
    switch (l) {
        case "true":
            return create(bool, true)
        case "false":
            return create(bool, false)
        default:
            return create(name, l)
    }
}

let callOp = ([f, arg]) => {
    let functionName = expect(name, f)
    let argValue = expect(number, arg)

    let functionImpl = { 
        "sin": Math.sin,
        "cos": Math.cos,
        "exp": Math.exp
    }[functionName as string] ?? error(`Unknown mathematical function ${functionName}.`)

    return create(number, functionImpl(argValue))
}


let plusOp = numberFunc(([x,y]) => y === undefined ? x : x+y)
let minusOp = numberFunc(([x,y]) => y === undefined ? -x : x-y) 
let timesOp = numberFunc(([x,y]) => x*y)
let divideOp = numberFunc(([x,y]) => x/y)
let numberOp = ([s]) => create(number, parseFloat(expect(literal, s)))

let chainMacro = ({ ev, operators }, body, [initialValue]) => {
    let ambientNameOp = operators["#name"]

    let nameOp = ([name]) => {
        if (expect(literal, name) === "value") return value
        else return ambientNameOp([name])
    }

    let ops = { "#name": nameOp }
    let macros = {}
    let value = ev(initialValue)

    for (let expr of body) {
        value = ev(expr, ops, macros)
    }

    return value
}

let scriptMacro = ({ ev }, body) => {
    let ops = { 
        "#print": printOp,
        "+": plusOp,
        "-": minusOp,
        "*": timesOp,
        "/": divideOp,
        "#number": numberOp,
        "#name": nameOp,
        "#string": stringOp,
        "#call": callOp,
        "<=": ltOp,
        "#PI": piOp,
        "#ifthenelse": ifThenElseOp,
    }

    let macros = { 
        "chain": chainMacro,
    }

    body.forEach(expr => ev(expr, ops, macros))

    return create(nothing, undefined)
}

let compiler = new MicroCompiler(
    [
        [
            { name: "#string", arity: 1 },
            { name: "#call", arity: 2 },
            { name: "#number", arity: 1 },
            { name: "#name", arity: 1 },
        ],
        [
            { name: "*", arity: 2 }, 
            { name: "/", arity: 2 }
        ],
        [
            { name: "+", arity: [1,2] },
            { name: "-", arity: [1,2] }
        ],
        { name: "<=", arity: [2, Infinity] },
        { name: "#print", arity: [0, Infinity] },
        [
            { name: "#PI", arity: 0 },
            { name: "#ifthenelse", arity: 3 }
        ]
    ],
    [
        { name: "chain", arity: 1 },
    ],

    literalValue => create(literal, literalValue),
    scriptMacro,
)
```

## Guess what ? We're not done with macros

Macros are the building blocks of Micro, and that's why Micro adds quite a lot of useful syntactic sugar around them : It's time to talk about the [tweaks in macro syntax](#macros) I mentionned when introducing you to macros in the very beginning.

The syntax bits we'll introduce here allows for... say, questionnable scripts :

```
[[ That's valid ! But it's terribly written, and the syntax used is very confusing  ]]
if x(y) {{
    with { someObject } do { 
        [[ Some code here ]]
    }
}}
```

Macro extended syntax allows for pretty wild things, like the ones you see here, but just because you _can_ doesn't mean you _should_. Think twice before implementing a feature, and, when writing scripts, always seek clarity. If at any point your scripts start to look like the above, it means that something went horribly wrong along the way.

### Macro binding

In standard languages, there often are some kind of declarations (classes, functions, etc), which look like this :

> Python
```python
def f(x, y):
    ...

class A(B):
    ...
```

> JS
```js
function f(x, y) {
    ...
}

class A extends B {
    ...
}
```

The pattern is always the same : create a complex object (function or class), then assign it to a name (f and A, respectively).

While we can't obtain the exact same syntax in Micro, we can mimic it pretty well by using _macro binding_ syntactic sugar and the special `#bind` operator. If we write :

```
myMacro bindingName(arg1; ...) {
    statement1;
    ...
}
```

It is translated by the compiler to :

```
[#bind bindingName; myMacro(arg1; ...) {
    statement1;
    ...
}]
```

> `bindingName` must be a "name", that is `x3`, `foo` or `_myFunction`. I quoted name here, because it has the same syntax but is not actually a name : It will be passed to `#bind` as a literal, like \`_myFunction\`

So if we previously declared macros named `func` and `class`, for example, we would be able to write :
```
func f(x; y) {
    [[ Do things ]]
};

class A(B) {
    [[ Class body ]]
};
```

Translated to :

```
[#bind `f`; func(x; y) { ... }];
[#bind `A`; class(B) { ... }];
```

Cool, isn't it ?

> That's where [that `if x(y) {{ ... }}`](#guess-what--were-not-done-with-macros) came from (why there was two curly brackets will follow later) : it's actually bind syntax translating to ```[#bind `x`; if (y) { ... }]```. Pretty confusing when you expect `x(y)` to be a function call, not even mentioning the fact that binding an `if` that way doesn't particularly make sense.

### Head, body, and limbs

Behold : we're finally about to uncover the truth about that [mysterious `limbs` argument](#macros-again) that is passed to macro reducers !

We earlier wrote an `if` macro that worked pretty well : 

```
if (condition) {
    [[ code here ]]
}
```

While it's all nice and stuff, it lacks something : an else branch. Sure, we could do something like this :

```

if (expr) {
    [[ code here ]]
}

if (#not expr) {
    [[ code here ]]
}
```

But that's really awkward : what if `expr` is super long and we don't want to write it out twice ? And it requires to implement `#not` beforehand.

Fortunately, there is support for something called limbs. Take that beautiful comparison between macros and the human body, which has no goal but to convince you the appelation `limbs` makes sense :

```
if (expr) [[ <-- That's the head ]] {
    [[ Here's the body ]]
} else { 
    [[ And here's an 'else' limb ! ]]
    [[ Some code here ]]
}
```

Limbs are additionnal blocks of code following a macro and begining with an identifier (here, "else").

What happens is that when encoutering such an expression, the compiler calls the `if` macro reducer with the good old first three arguments and a fourth one, `{ "else": AST[] }`. 
In the `else` field of that object are all `AST`s corresponding to statements written in the else limb.

If we were to write an `if` macro with an `else` limb : 

```js
// The declaration is slightly different from our previous if
// Here limbs is used to tell that `if` accepts an else { ... } block as a limb
{ name: "if", arity: 1, limbs: ["else"] }

let ifMacro = 
    ({ ev }, body, [ condition ], { "else": elseLimb }) => {
        let [condition] = args
        
        if (expect(bool, ev(condition))) {
            for (let expr of body) {
                ev(expr)
            }
        } else {
            for (let expr of elseLimb) {
                ev(expr)
            }
        }
        
        return create(nothing, undefined)
    }
```

> It is common to refer to macros with limbs by joining together their name and limbs, separed by slashes : `if/else`, for example, would refer to the above macro. It doesn't reflect in the declarations and code however, as the macro is still known as just `if` to the compiler.

Our `if` now evaluates its condition, and executes the according block.

A single macro can declare as many limbs as it wants. 
If `limbs: ...` is not explicitely set inside the macro declaration, it defaults to `[]`. Limbs are pretty flexible and can be used without much restrictions. For example, with a declaration for a macro `try/catch/finally` of `{ name: "try", arity: 0, limbs: ["catch", "finally"]}`, these are all correct :

```
[[ All two limbs in the right order ]]
try {} catch {} finally {};

[[ All two limbs in the wrong order ]]
try {} finally {} catch {};

[[ Missing "finally" limb ]]
try {} catch {};

[[ No limb at all ]]
try {}; 

[[ These by the way demonstrate that limbs can be used to mimic really well control flow structures syntax. ]]
```

If a limb is missing, its corresponding field in the `limbs` argument to the macro reducer is defaulted to `undefined`, so these two are **not** the same :

```
[[ limbs["else"] is undefined ]]
if { [[ Blah blah ]] };  

[[ limbs["else"] is [] ]]
if { [[ Blah blah ]] } else {}; 
```


Don't overestimate limbs : a thing such as python's `elif` cannot be implemented with them (nor implemented at all) in Micro :

```
[[ Not valid Micro syntax ]]

if (cond) {

} elif (cond2) {

} elif (cond3) {

} else {

}
```

First because limbs don't take arguments, so `elif (cond2) { ... }` would make the compiler crash, then because limbs can only appear once, so the second `elif` limb would trigger an error. That doesn't mean limbs aren't powerful ; rather that they're not powerful enough to mimic everything and anything.

> That's the explanation for the [`with { someObject } do { ... }`](#guess-what--were-not-done-with-macros) : It's a `with/do` macro. While it may seem like a creative use of limbs to pass in arguments to macros... well, there already is a clearer way to do so : 

```
with (someObject) {
    ...
}
```

> By using limbs instead of arguments, we loose the arity check, only gaining a little bit of... english-ness ? Thus it's considered very bad practice.

### Silence !

You may have wondered what curly brackets do when used alone : 

```
[[ What is that ? ]]
{}
```

The answer is : it's in fact a (although very inconspicuous) macro call to a special macro, called the _silent macro_. It's a stylish name for a macro that is actually called "" (empty string).

So a pair of curly bracket, eventually with statements inside, just calls the silent macro with said statements as its body, no arguments and no limbs. 
There is no way to pass in arguments or limbs to the silent macro, and its declaration must therefore always read so : 

```js
{ name: "", arity: 0 } // limbs: [] is optional
```

You could, for example, use it to implement JS-like object notation : 

```
{
    a: "foo";
    b: "bar";
    c: {
        hello: "World !";
        list: [1, 2, 3]
    }
}
```

By the way, could you tell the "list literal" is actually the coma operator used in an infix pack ?

> That's what the [`if x(y) {{ ... }}`](#guess-what--were-not-done-with-macros) was : a silent macro nested inside an `if` macro. The silent macro is an expression by itself, and it's better to explicitely space it out from other constructs. 