
# 6 Summary theory

What's the next step?

To answer that question, we'll have to understand a bit of theory, to know where we are and how we got here.

## 6.1 Parsing source code into AST

We started in chapter 2 and 3 by making a parser consisting of the `Lexer` and `Parser` classes. The parser takes source code and translates, or *parses*, it into a structured representation called AST. AST is short for **A**bstract **s**yntax **t**ree, which means that the original program, or code, is represented as a tree-structure.

We defined the *structure* of the AST (*code as tree-structure*), by consisting of the `Stmt`, `Expr`, ... types.

We converted the source code from text to AST, because AST is easier to work with in the step that followed.

## 6.2 Evaluating AST

In chapter 4 we made an evaluator consisting primarily of the `Value` type, `Syms` class and `Evaluator` class. The evaluator, or AST evaluator, takes a program represented in AST (*code as tree-structure*), goes through the tree-structure and calculates the resulting values.

Execution using the evaluator is a top-down, outside-in process, where we start with the root node, and then call `evalStmt` and `evalExpr` for each child node recursively. We then use the value obtained by evaluating the child nodes to calculate the result for a node.

The only upside for implementing an AST evaluator, is that it is simple to implement and think about. An AST evaluator operates on an AST which is a comparatively close representation of the original source code. Humans understand programs from the point of the source code. Therefore, an AST evaluator executes the code, or *thinks about* the program, the same way we humans think about program. The human brain functions efficiently using layers of abstraction.

Take the math expression `2 * (3 + 4)`. We start by examining the entire expression. Then we split up the expression into its components, that being a multiplication of `2` by the addition of `3` and `4`. We then, to calculate the result of the outer expression, calculate the result of the inner expression: `3 + 4 = 7`. Then we use that result and calculate the multiplication: `2 * 7 = 14`. The evaluator functions exactly this way.

There are multiple downsides to AST evaluation.

One downside is that some features of the source languages are *ugly* to implement. While expression evaluation is conceptually simple to evaluate using function calls, other features are not. Control flow related features such as break and return cannot be evaluated using only function calls. This is because function calls follow the (control) flow, while break and return *breaks* the control flow.

But the primary downside of AST evaluation is performance. While humans are most efficient when using layers of abstractions, computers are not. For various reasons, calling functions recursively repeatedly and jumping through tree-structures is very inefficient for computers. Both in terms of memory footprint and execution time. Computers are much more efficient with sequential execution.

Take the expression defined above. Now imagine we're describing the instructions for how to get the result. We would of course look at the while expression, then break it down. We could then formulate *instructions* such as *add 3 to 4, then multiply by 2*. Now if we execute the instructions, we don't start by examining the entire expression, we just execute the instructions in order. We have now translated the expression into linear execution, which computers are very good at running.

## 6.3 Instructions

The 2 main pros of using instructions counter the cons of AST evaluation.

A lesser point pertaining to implementing control flow, is that everything is done using the sequential instructions. This means special control flow such as break, are handled in the same manner as regular control flow like loops.

The primary upside compared to AST evaluation is performance. Running instructions is a sequential operation, which could for example look like an array of instructions, where a location of the current instruction is stored, and then execution loops over the array with a for loop. This is A LOT faster compared to AST evaluation with a tree-structure and recursive function calls. The technical details of why the performance is faster this way is hard to explain both simply and accurately, so I'll spare the explanation. You can think about it like this: the parser generates a tree, to evalute the program, we need to do tree-traversal, tree-traversal is slow, therefore we should minimize the amount of tree-traversal. In the AST evaluator, tree-traversal is used on an expression everytime it is run, in a loop or function call for instance. Instead we'd rather want to do tree-traversal once for the entire program, and then generate these instructions, which does not require tree-traversal to run, ie. we do the costly tree-traversal once instead of multiple times.

The primary downside of this approach compared to AST evaluation is the effort required. AST evalution is both conceptually simple and relatively simple to implement, as it executes the code in just the form the parser outputs, which is also close to the source language. To run instructions instead, we need to translate the program from the AST into runnable sequential instructions. To evalute using instructions, instead of AST evalution, we need to do the following conceptual steps (implementationally seperate in our case):

- Parse source code into AST.
- ~~Evaluate AST.~~
- Resolve symbols. Like how we used the symbol table in the evaluator.
- Check semantic code structure. We can often parse source code, that cannot be run. In the evaluator we had checks different places, that would throw errors. This is the same concept. This will also include type checking in our case.
- Translate (or compile) resolved and checked AST into instructions.
- Execute instructions.

As can be seen, some of the needed steps are steps which are combined in the evaluator. Symbol resolution (resolving) is comparable to how we resolved symbol values (variables) in the evaluator. Semantic code checking is comparative to how we check the AST at different places in the evaluator, such as checking that the `+`-operator is only applied to either 2 ints or 2 strings.

Translation into instructions and seperate executions are new steps. These are also conceptually more advanced than AST evaluation in the sense, that AST evaluation operates on a high level representation of the code meaning it's close to the original source code, whereas instructions are further away, meaning more low level. This means we need to make some information needed for executing instructions explicit, which may be implicit in AST representation because the tree-structure in-an-of-itself contains some of the information.

We also need to design the instructions, meaning we need to choose which instructions should be available (instruction set) and some details of how they should be run. The design decisions in this step is essentially arbitrary, meaning there often is not a *correct* decision, whereas the evaluation is designed exactly to evaluate the AST which in some sense is designed to exactly represent the source code. These design decisions require trade-offs, eg. of perfomance, simplicity, ease of implementation, portability, versatility.

## 6.4 Virtual machine

The component executing the program is called a virtual machine, VM for short, when we're working with instructions. We'll make the distinction like this: an evaluator is synonymous with an AST evaluation, whereas a virtual machine runs instructions.

The design decisions of a virtual machine, ie. how it runs and how the program should be, is called the architecture.

When running a program, we often need features which operate *outside of* the program. An example is the builtin functions in chapter 4. These functionalities are called the *runtime*. The runtime is typically implemented to a certian degree in the virtual machine. In that sense the VM also functions as an interface between the program and the host computer.

## 6.5 Types

All values in a program, are of certain types (value types). The type determines how the value is handled, eg. `+` operator on two integers is addition, while the same operator on two strings is concatenation. Types in a program are either known in advance *at compiletime* or is first known when the program runs *at runtime*. The evaluator implemented in chapter 4 would figure out the types as the evaluation went along, this was an example of determining types at runtime, also called *dynamic typing*. When all types are determined in compiletime, this is called *static typing*. The language implemented thus far offer the programmer no way of explicitly specifying types for variables or functions.

There are three important factors in the decision of using either *static* or *dynamic* typing.

The first is language design and developer experience. Types may both help the programmer in writing clear and correct programs, but may also hinder them, because the amount of work to be done is larger, when everything requires a determined type.

The second factor pertains to target, runtime and performance. A program can only be compiled down to and run at a certain level without types being determined. With types predetermined, we can determine the size of values and which operations to execute, without we need to run the program to figure these things out. The lower we can compile a program before execution, the less the theoretical overhead. The higher level the program is at runtime, the bigger the *runtime* need to be theoretically.

The third factor is about effort. Static types require more tooling, eg. parsing types and type checking.

We could also delve into type annotations in dynamically typed languages and runtime compilation and optimizations in JIT compilers to explain languages like Typescript, but that's beyond this course.

