# Project 3: SmallC Interpreter

CMSC 330, Spring 2019

Due November 14th at 11:59pm (Late: November 15th at 11:59pm)

P/R/S: 50/50/0

Ground Rules and Extra Info
---------------------------
This is **NOT** a pair project. You must work on this project alone as with most other CS projects. See the Academic Integrity section for more information.

In your code, you may use **any** non-imperative standard library functions (with the exception of printing), but the ones that will be useful to you will be found in the [`Pervasives` module][pervasives doc] and the [`List` module][list doc]. Note that the `List` module has been disallowed in previous projects, but in the case of this project and projects going forward it will be allowed.

You still may not use any imperative structures of OCaml such as references or the `Hashtbl` module.

Introduction
------------
In this project, you will implement an interpreter for SmallC, a small C-like language. The language contains variables, `int` and `bool` types, equality and comparison operations, math and boolean operations, simple control flow, and printing, but no input statements. Here is an example program:
<!--EDIT Line 19 - Made it explicit SmallC has no input statements. -->
```
int x;
x = 2 * 3 ^ 5 + 4;
printf(x > 100);
```
When executed, it will produce the result `true`. 

As discussed in class a language implementation has two basic steps. First, a *parser* takes text files and converts them into a data structure called an *abstract syntax tree* (AST). Then, the AST is either directly interpreted or compiled so it can be executed. <!--Line 28 - EDIT changed "parts" to "steps" since the second wasn't a part (noun), but an action.  --> 

In this project, you are implementing an interpreter for the second step. The first step, the parser, has been provided for now but, you will be asked to implement it in a later project. You are writing key parts of an interpreter that behaves like the Ruby command to take a SmallC program file name, run the code, and produce the output. <!--Line 30 - EDIT Added the last sentence to make clear the context of their work.  --> 

More specifically for these key parts you will implement functions `eval_expr` and `eval_stmt` in the file `eval.ml`. These functions, respectively, evaluate *expressions*, which are ASTs of type `expr` and execute *statements*, which are ASTs of type `stmt`. Expression and statement AST types `expr` and `stmt` are defined as variant types in the file `smallCTypes.ml`. This file should be a constant reference while you are working on the project. Functions that carry out language interpretation are frequently called `eval`, and later we'll call your code **Evaluator** instead of interpreter. 

To assist you in your work, we are providing code that will do parsing for you. This code will take in a text file and produce an AST of type `stmt`. For our example above, the parser we provide will produce a the following value of type `stmt`:
```
Seq(Declare(Int_Type, "x"),
Seq(Assign("x",
  Add(
    Mult(
      Int(2),
      Pow(
        Int(3),
        Int(5))),
    Int(4))),
Seq(Print(Greater(ID("x"), Int(100))), NoOp)))
```
This `stmt` value will then be passed to the `eval_stmt` function you implement to be interpreted. Since we're doing the parsing, you are not responsible for poorly formed input - if there's a parse error, your code should not be called. <!--Line 47 - FIXME added that last sentence to better explain I/O behavior. Assuming it's true ... -->

<!-- EDIT. Added the below to better explain the whole thing. -->
The raw input SmallC files will have a single main() function, so the example above would appear as below. But, you only have to interpret the sequence of statements in the body of the program, not the function header. The parser we provide will strip the header and supply you with the AST that represents the body as one compound statement.
```
int main() {
int x;
x = 2 * 3 ^ 5 + 4;
printf(x > 100);
}
```
A regular interpreter, like the Ruby example above, would take this program and print `true`. But, for testing purposes in this assignment, you will not directly print the output. You will print through a special set of routines explained later that allow us to capture and test your output. Read the **Print** section carefully. And, for testing purposes we'll look at the internal state you maintain while your code interprets and executes SmallC. You'll keep and update an environment as a list of (variable,value) pairs. In the example above you'd start with the empty environment `[]`, update to `[(x,0)]` when x is declared, and end with the environment`[(x,490)]` after the assignment. 

If the SmallC program being interpreted is valid, we will test if you produce the right sequence of print statements and final environment. If the program is invalid, with a type error or numeric error, we will test if you raise the right exception at the right time. 

While type name `stmt` is singular, and looks like one statement, `stmt` is a recursive type with `stmt => stmt; stmt`. Think of `stmt` as one atomic statement, or a sequence of statements as one compound statement. The `eval_stmt` function properly takes one or a sequence of statements and evaluates each in turn.

To review, you are to write and test the implementations for two Ocaml functions found in the file **src/eval.ml**:

 `eval_expr : environment -> expr -> value`
 `eval_stmt : environment -> stmt -> environment`
 
The rest of this readme will explain these functions in detail. We suggest before coding you look at the SmallC examples in **test/public_input** to understand SmallC, read carefully the specifications for `eval_expr` and `eval_stmt` below, study the unit cases, and write out for yourself all the cases for these functions.  

<!-- End added section -->

Project Files
-------------
To begin this project, you will need to commit any uncommitted changes to your local branch and pull updates from the git repository. [Click here for directions on working with the Git repository.][git instructions] The following are the relevant files:

-  OCaml Files
    - **src/**: This directory contains all of the code you should be working with directly.
    - **src/eval.ml**: All of your code will all be written in the file `eval.ml`. _None of the files other than this one should be changed._
    - **src/eval.mli**: This is the _interface_ for `eval.ml`. It defines what types and functions are visible to modules outside of `eval.ml`.
    - **src/smallCTypes.ml**: This file contains all type definitions used in this project.
    - **src/evalUtils.ml**: The small part of this file that concerns you in implementing this project is called out very explicitly when it is needed later in the document, otherwise you should not need to look at this file.
    - **bin/**: This directory contains code for a SmallC interface you are given.
    - **bin/interface.ml**: We have provided this interface to aid you in testing your program. You should not modify this file. The following section in the document details how you can use the interface.
    - **test/**: This directory contains unit tests and related code.
    - **test/public.ml** and **public_inputs/**: The public test driver file and the SmallC input files to go with it, respectively.
    - **test/testUtils.ml**: Functions written to aid in tests. These functions are used by our test suite and you are encouraged to use them in your own tests.
- Compiled OCaml Files
    - **deps/parser/**: This directory contains a precompiled implementation for a SmallC lexer and parser used for turning plain files into OCaml datatypes. They are precompiled because you will implement  your own parser in a later project.
- Submission Scripts and Other Files
  - **test.sh** and **interface.sh**: Shell script wrappers around dune commands to help you run and test the interpreter.
  - **submit.rb**: Execute this script to submit your project to the submit server.
  - **submit.jar** and **.submit**: Don't worry about these files, but make sure you have them.
  
Compilation, Tests, and Running
-------------------------------

Public and student tests can be run using the same `dune` command that you used in the previous projects but, this time you need to set the environment variable `OCAMLPATH` before running the command. The full command is now `OCAMLPATH=dep dune runtest -f`. Setting `OCAMLPATH` tells `dune` where it can find the lexer and parser that we have provided. You will need to provide this environment variable for every `dune` command so you may want to add it to your environment once by running `OCAMLPATH=dep` as separate command before using `dune`. We have also provided a shell script `test.sh` that runs the command given above.

As an alternative to running tests, you can run the SmallC interpreter directly on a SmallC program by running `OCAMLPATH=dep dune exec bin/interface.bc -- <filename>`. This driver, provided by us, reads in a program from a file and evaluates the code, outputing the results of any print statements present in the source file. As we mentioned before, this command is a lot like the `ruby` command, but instead of running the ruby interpreter, it runs the SmallC interpreter that you wrote yourself! This depends on the lexer and parser that we wrote for you.

If you would like more detailed information about what your code produced, running `OCAMLPATH=dep dune exec bin/interface.bc -- <filename> -R` provides a report on the resultant variable bindings as reported by your evaluator. If you would like to see data structure that is being generated by the parser and being fed into your interpreter, run `OCAMLPATH=dep exec bin/interface.bc <filename> -U` and our `Utils` module will translate the data structure into a string and print it out for you - this part does not require any of your code, so feel free to try it on the public tests before you even start! Use the `interface` executable to your advantage when testing; that's why we're providing it! Note that you don't need to touch `interface.ml` yourself, as it only functions as an entry point for the interpreter and is independent of your implementation.

As with running tests, we have provided a shell script `interface.sh` that executes the command given above. <!-- EDIT Original did say which version, With or without the -R option. Was not clear that "the command" refers to two paragraphs above. -->

The Evaluator
-------------
*Your evaluator must be implemented in `eval.ml` in accordance with the signatures for `eval_expr` and `eval_stmt` found in `eval.mli`. `eval.ml` is the only file you will write code in. The functions should be left in the order they are provided, as your implementation of `eval_stmt` will rely on `eval_expr`.*

The heart of SmallC is your evaluator. We have already implemented `Lexer` and `Parser` modules that deal with constructing tokens and creating the AST out of a program. Where your code picks up is with a representation of SmallC programs as OCaml datatypes, which you are then responsible for evaluating according the rules below. A program is made up of a series of statements and expressions:  <!-- FIXME Have we ever defined Lexer and tokens? We haven'

- Statements represent the structure of a program - declarations, assignments, control flow, and prints in the case of SmallC. - Expressions represent operations on data - variable lookups, mathematical and boolean operations, and comparison. Expressions can't affect the environment, and as a result only return a `value` containing the value of the expression.

You are responsible for implementing two functions, `eval_expr` and `eval_stmt` in that order. Each of these takes as an argument an `environment` (given in `smallCTypes.ml`) that is an association list. An association list is a list of pairs (2-tuples) where the key is the first element of the pair and the value is the second element. Elements earlier in the list override elements later in the list. In our case the association list is of type `(string * value) list` where the string is the id name and the value is the current value of that id. In general an id in a program can be variable or something else like a function or class name, but in SmallC all ids are variables. **The List module has many useful functions for working with association lists**. Statements may change the values of variables, so `eval_stmt` returns a possibly updated environment. <!-- FIXME This refers to an association list of ids, but the only ids in SmallC are variables and the only values are ints and bools, right? So added a sentence to say that --> 

A formal operational semantics of SmallC can be found in [`semantics.pdf`][semantics document]. We highly recommend you read this document as you work to clear up any possible ambiguity introduced by the English explanations below. The formal semantics do not, however, define error cases such as addition between a boolean and an integer and therefore represent a stuck reduction. The expected behavior for these irreducable error cases are defined only in this document and can be boiled down to the following rules:
- Any expression containing division by zero should raise a DivByZero error when evaluated.
- Any expression or statement that is applied to the wrong types should raise a `TypeError` exception when evaluated, for example, the expression `1 + true` would result in `TypeError`.
- An expression or statement that redefines an already defined variable, assigns to an undefined variable, or reads from an undefined variable should raise a `DeclareError` when evaluated.

We do not enforce what messages you use when raising the `TypeError` or `DeclareError` exceptions; that's up to you. Evaluation of subexpressions should be done from left to right, as specified by the semantics, in order to ensure that lines with multiple possible errors match up with our expected errors.

### Function 1: eval_expr <!-- EDIT Changed it from Part 1 to Function 1. Remember, part 1 is the Parser, and part 2 is the Interpreter. Elegant writing is consistent.  -->

`eval_expr` takes an environment `env` and an expression `e` and produces the result of _evaluating_ `e`, which is something of type `value` (`Int_Val` or `Bool_Val`). Cases of the expression are:

#### Int

Integer literals evaluate to a `Int_Val` of the same value.

#### Bool

Boolean literals evaluate to a `Bool_Val` of the same value.

#### ID

An identifier evaluates to whatever value it is mapped to by the environment. Should raise a `DeclareError` if the identifier has no binding.

#### Add, Sub, Mult, Div, and Pow

*These rules are jointly classified as BinOp-Int in the formal semantics.*

These mathematical operations operate only on integers and produce a `Int_Val` containing the result of the operation. All operators must work for all possible integer values, positive or negative, except for division, which will raise a `DivByZeroError` exception on an attempt to divide by zero. If either argument to one of these operators evaluates to a non-integer, a `TypeError` should be raised.

#### Or and And

*These rules are jointly classified as BinOp-Bool in the formal semantics.*

These logical operations operate only on booleans and produce a `Bool_Val` containing the result of the operation. If either argument to one of these operators evaluates to a non-boolean, a `TypeError` should be raised.

#### Not

The unary not operator operates only on booleans and produces a `Bool_Val` containing the negated value of the contained expression. If the expression in the `Not` is not a boolean (and does not evaluate to a boolean), a `TypeError` should be raised.

#### Greater, Less, GreaterEqual, LessEqual

*These rules are jointly classified as BinOp-Int in the formal semantics*

These relational operators operate only on integers and produce a `Bool_Val` containing the result of the operation. If either argument to one of these operators evaluates to a non-integer, a `TypeError` should be raised.

#### Equal and NotEqual

These equality operators operate both on integers and booleans, but both subexpressions must be of the same type. The operators produce a `Bool_Val` containing the result of the operation. If the two arguments to these operators do not evaluate to the same type (i.e. one boolean and one integer), a `TypeError` should be raised.

### Function 2: eval_stmt <!-- EDIT Changed it from Part 2 to Function 1.  -->

`eval_stmt` takes an environment `env` and a statement `s` and produces an updated `environment` (defined in Types) as a result. This environment is represented as `a` in the formal semantics, but will be referred to as the environment in this document.

#### NoOp

`NoOp` is short for "no operation" and should do just that - nothing at all. It is used to terminate a chain of sequence statements, and is much like the empty list in OCaml in that way. The environment should be returned unchanged when evaluating a `NoOp`.

#### Seq

The sequencing statement is used to compose whole programs as a series of statements. When evaluating `Seq`,  evaluate the first substatement under the environment `env` to create an updated environment `env'`. Then, evaluate the second substatement under `env'`, returning the resulting environment.

#### Declare

The declaration statement is used to create new variables in the environment. If a variable of the same name has already been declared, a `DeclareError` should be raised. Otherwise, if the type being declared is `Int_Type`, a new binding to the value `Int_Val(0)` should be made in the environment. If the type being declared is `Bool_Type`, a new binding to the value `Bool_Val(false)` should be made in the environment. The updated environment should be returned.

#### Assign

The assignment statement assigns new values to already-declared variables. If the variable hasn't been declared before assignment, a `DeclareError` should be raised. If the variable has been declared to a different type than the one being assigned into it, a `TypeError` should be raised. Otherwise, the environment should be updated to reflect the new value of the given variable, and an updated environment should be returned.

#### If

The `if` statement consists of three components - a guard expression, an if-body statement and an else-body statement. The guard expression must evaluate to a boolean - if it does not, a `TypeError` should be raised. If it evaluates to true, the if-body should be evaluated. Otherwise, the else-body should be evaluated instead. The environment resulting from evaluating the correct body should be returned.

#### While

The while statement consists of two components - a guard expression and a body statement. The guard expression must evaluate to a boolean - if it does not, a `TypeError` should be raised. If it evaluates to `true`, the body should be evaluated to produce a new environment and the entire loop should then be evaluated again under this new environment, returning the environment produced by the reevaluation. If the guard evaluates to `false`, the current environment should simply be returned.

*The formal semantics rule for while loops is particularly helpful!*

#### DoWhile

The do-while statement consists of two components - a body statement and a guard expression. The guard expression must evaluate to a boolean - if it does not, a `TypeError` should be raised. The body should be evaluated to produce a new environment, and if the guard expression evaluates to `true` in this new environment, the entire loop should then be evaluated again in the new environment, returning the environment produced by the reevaluation. If the guard evaluates to `false`, the new environment should be returned.

*The formal semantics rule for do-while loops is particularly helpful!*


#### Print

The print statement is your project's access to standard output. First, the expression to `print` should be evaluated. Print supports both integers and booleans. Integers should print in their natural forms (i.e. printing `Int_Val(10)` should print "10". Booleans should print in plaintext (i.e. printing `Bool_Val(true)` should print "true" and likewise for "false"). Whatever is printed should always be followed by a newline.

**VERY IMPORTANT NOTE ON PRINTING**: If you attempt to print with `print_string`, `print_int`, etc... you will fail any test that checks your output. This is because we do not directly intercept your output, but rather have implemented some wrapper functions in the `EvalUtils` for program printing (do not remove `open EvalUtils` from the top of `eval.ml`!). As a result, your code may have any amount of `print_string` statements and it **WILL NOT** affect the tests. Only printing performed through the following wrapper functions will be evaluated by the test system, and as a result should be used exclusively with `Print`. The supplied print wrappers which you *must use in place of the built-in print functions for any and all output resulting from `Print` statements and for nothing else* are as follows:
- `print_output_string : string -> unit`
- `print_output_int : int -> unit`
- `print_output_bool : bool -> unit`
- `print_output_newline : unit -> unit`

Again, you may use the normal prints defined in the standard library `Pervasives` module, but they will not affect your test grading in any way.

Project Submission
------------------
You should submit the file `eval.ml` containing your solution. You may submit other files, but they will be ignored during grading. We will run your solution as individual OUnit tests just as in the provided public test file.

Be sure to follow the project description exactly! Your solution will be graded automatically, so any deviation from the specification will result in lost points.

You can submit your project in two ways:
- Submit your files directly to the [submit server][submit server] as a zip file by clicking on the submit link in the column "web submission".
Then, use the submit dialog to submit your zip file containing `eval.ml` directly.
Select your file using the "Browse" button, then press the "Submit project!" button. You will need to put it in a zip file since there are two component files.
- Submit directly by executing a the submission script on a computer with Java and network access. Included in this project are the submission scripts and related files listed under **Project Files**. These files should be in the directory containing your project. From there you can either execute submit.rb or run the command `java -jar submit.jar` directly (this is all submit.rb does).

No matter how you choose to submit your project, make sure that your submission is received by checking the [submit server][submit server] after submitting.

Academic Integrity
------------------
Please **carefully read** the academic honesty section of the course syllabus. **Any evidence** of impermissible cooperation on projects, use of disallowed materials or resources, or unauthorized use of computer accounts, **will be** submitted to the Student Honor Council, which could result in an XF for the course, or suspension or expulsion from the University. Be sure you understand what you are and what you are not permitted to do in regards to academic integrity when it comes to project assignments. These policies apply to all students, and the Student Honor Council does not consider lack of knowledge of the policies to be a defense for violating them. Full information is found in the course syllabus, which you should review before starting.

<!-- Link References -->
[list doc]: https://caml.inria.fr/pub/docs/manual-ocaml/libref/List.html
[semantics document]: semantics.pdf

<!-- These should always be left alone or at most updated -->
[pervasives doc]: https://caml.inria.fr/pub/docs/manual-ocaml/libref/Pervasives.html
[git instructions]: ../git_cheatsheet.md
[submit server]: https://submit.cs.umd.edu
[web submit link]: image-resources/web_submit.jpg
[web upload example]: image-resources/web_upload.jpg
