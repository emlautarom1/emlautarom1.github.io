---
title: "Mandatory Semicolons Are Terrible"
date: 2023-05-11T18:24:51-03:00
summary: A rant on why I think semicolons (;) are terrible for programming
draft: true
---

## Introduction

After working with Haskell for over two years, and before that with Typescript for around a year, I kind of forgot about mandatory semicolons, but now I'm working with C# and Rust and, oh boy, they're extremely annoying after being *on the other side*.

> I'm explicitly talking about **mandatory** semicolons, those that you need at the end of each statement otherwise the code does not compile.

Let's take some examples of languages where semicolons are not mandatory. For example, in Haskell, semicolons exist and can be used, and sometimes are quite useful in `do` blocks. For example:

```haskell
func = do { name <- getLine; putStrLn ("Hello, " ++ name) }
```

In practice though, the community will prefer to not use semicolons or braces, and several autoformatters will make the change automatically:

```haskell
func = do
    name <- getLine              -- no `;` at the end
    putStrLn ("Hello, " ++ name) -- no `;` at the end either
```

The important part here is that the semicolon is **optional**. You don't need to write it, and the language manages just fine.

Another language with a similar behavior is Javascript (and by extent, Typescript). Here, braces are **mandatory** but semicolons are not. For example

```javascript
let name = "Lautaro"   // no `;` at the end
console.log("Lautaro") // no `;` at the end 
```

In Javascript though this can **rarely** cause some issues, like in the following function:

```javascript{linenos=true}
function test() {
    let x = 2
    return
        x * x
}
```

Here, the expression on line 4 (`x * x`) is ignored due to the `return` statement in the previous line, and it's probably not what the developer intended. Still, if you're using an editor more advanced than plain Notepad then you should be getting an `unreachable statement` warning, and if you're using Typescript then even better.

All of this is to say that semicolons are not mandatory for pretty much 99% of the code, and we could easily make them optional like in Javascript, Haskell and Kotlin, to give some examples of real languages where this is the case.

## Reasons to not have mandatory semicolons

I would like to list now why I think that mandatory semicolons have no place in a modern language, given that we've already mentioned a scenario where they could have place.

### Hard to type

Maybe it's just me, but I miss the semicolon key very often, making me type multiple `l`s and `'` by accident. Lack of practice could be the reason, but still, they make me slower than I already am.

### Don't work well with autocomplete

Moder dev environments heavily rely on autocomplete (`Ctrl+Space`, and `Ctrl+Tab`), where the editor will complete the method/function name and even place placeholder paramenters in some cases.

Mandatory semicolons make them a bit akward to use. Let's take this Rust code for example:

```rust
// `❚` stands for the caret
let x = 5.cm❚
// Press tab to autocomplete `cmp`
let x = 5.cmp(❚)
// Write a paramenter
let y = 5.cmp(&3❚)
```

As you can see, now you need to manually go to the end of the line (just a single keystroke generally) to place the semicolon which provides no value at all, since...

### Provide no value

As we already mentioned, semicolons can be optional in pretty much all scenarios, so they provide no real value to the programmer. In my experience, semicolons end up being treated as noise that you can safely ignore, except when they cause a compilation error, which leads me to...

### Obfuscate errors

For several years I've used some form of error lens plugin in my IDEs to show errors (for [VSCode](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens) and for [IntelliJ](https://plugins.jetbrains.com/plugin/19678-inspection-lens)), warnings and suggestions next to my code. In practice, it looks something like this:

![Error lens](/screens/error-lens.png)

When you have mandatory semicolons, if there is an error in a particular line **and you're also missing the semicolon at the end**, then the error message becomes useless since it will point out first that a semicolon is missing, instead of the actual error. For example:

![Missing semicolon](/screens/error-lens-semicolon.png)

Here, the actual error is comparing an `number` with a `string` and it has nothing to do with the fact that a semicolon is missing.

Consider the common autocomplete workflow with statically typed languages:

1. Write part of a method/function name
2. Autocomplete the name
3. Adjust the parameters
4. There is a compilation error, but you don't exactly know until you add the semicolon at the end
5. Now you can figure out the error

Step 4 is the productivity killer in my experience, since I need to do a manual, repetitive and worthless action just to continue with my work. It might not seem like a lot, but in my experience it adds up and quickly.

## Conclusions

Mandatory semicolons are useful in extremely rare occasions and in some particular languages. For the most part, they provide little to no value and result in noise that developers often completely ignore while writing the code. However, they still need to be considered to keep the code compiling.

Several programming languages have shown that semicolons can be optional, making writing code easier, less repetitive, and better suited for modern tools like code autocomplete.
