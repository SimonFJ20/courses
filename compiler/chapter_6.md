
# 6 Summary theory

What's the next step?

To answer that question, we'll have to understand a bit of theory, to know where we are and how we got here.

## 6.1 Parsing source code into AST

We started in chapter 2 and 3 by making a parser consisting of the `Lexer` and `Parser` classes. The parser takes source code and translates, or *parses*, it into a structured representation called AST. AST is short for __A__bstract __s__yntax __t__ree, which means that the original program, or code, is represented as a tree-structure.

We defined the *structured* of the AST (*code as tree-structure*), by consisting of the `Stmt`, `Expr`, ... types.

We converted the source code from text to AST, because AST is easier to work with in the step that followed.

## 6.2 Evaluating AST

In chapter 4 we made an evaluator consisting primarily of the `Value` type, `Syms` class and `Evaluator` class. The evaluator, or AST evaluator, takes a program represented in AST (*code as tree-structure*), goes through the tree-structure and calculates the resulting values.

Execution using the evaluater is a top-down, outside-in proceess, where we start with the root node, and then call `evalStmt` and `evalExpr` for each child node recursively. We then use the value optained by evaluating the child nodes to calculate the result for a node.

The only upside for implementing an AST evaluator, is that it is simple to implement and think about. An AST evaluator operates on an AST which is a comparatively close representation of the original source code. Humans understand programs from the point of the source code. Therefore, an AST evaluator executes the code, or *thinks about* the program, the same way we humans think about program. The human brain functions efficiently using layers of abstraction.

Take the math expression `2 * (3 + 4)`. We start by examining the entire expression. Then we split up the expression into its components, that being a multiplication of `2` by the addition of `3` and `4`. We then, to calculate the result of the outer expression, calculate the result of the inner expression: `3 + 4 = 7`. Then we use that result and calculate the multiplicat: `2 * 7 = 14`. The evaluator functions exactly this way.

There are multiple downsides to AST evaluation.

One downside is that some features of the source languages are *ugly* to implement. While expression evaluation is conceptually simple to evaluate using function calls, other features are not. Control flow related features such as break and return cannot be evaluated using only function calls. This is because function calls follow the (control) flow, while break and return *breaks* the control flow.

But the primary downside of AST evaluation is performance. While humans are most efficient when using layers of abstractions, computers are not. For various reasons, calling functions recursively repeatedly and jumping through tree-structures is very inefficient for computers. Both in terms of memory footprint and execution time. Computers are much more efficient with sequential execution.

Take the expression defined above. Now imagine we're describing the instructions for how to get the result. We would of course look at the while expression, then break it down. We could then formulate instructions such as *add `3` to `4`, then multiply by `2`*. Now if we *execute* the instructions, we don't start by examening the entire expression, we just execute the instructions in order. We have now translated the expression into linear execution, which computers are very good at running.

## 6.3 Instructions


