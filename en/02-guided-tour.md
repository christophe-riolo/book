# A Guided Tour

We'll start our introduction to OCaml with the OCaml toplevel, an
interactive shell that lets you type in expressions and then evaluates
them immediately.  When you get to the point of running real programs,
you'll want to leave the toplevel behind, but it's a great tool for
getting to know the language.

You should have a working toplevel as you go through this chapter, so
you can try out the examples as you go.  There is a zero-configuration
browser-based toplevel that you can use for this, which you can find here:

     http://realworldocaml.org/core-top

Or you can install OCaml and Core on your computer directly.
Instructions for this are found in Appendix {???}.

## OCaml as a calculator

Let's spin up the toplevel and open the `Core.Std` module, which gives
us access to Core's libraries, and then try out a few simple numerical
calculations.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
        Objective Caml version 3.12.1

# open Core.Std;;
# 3 + 4;;
- : int = 7
# 8 / 3;;
- : int = 2
# 3.5 +. 6.;;
- : float = 9.5
# sqrt 9.;;
- : float = 3.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This looks a lot what you'd expect from any language, but there are a
few differences that jump right out at you.

- We needed to type `;;` in order to tell the toplevel that it should
  evaluate an expression.  This is a pecularity of the toplevel that
  is not required in compiled code.
- After evaluating an expression, the toplevel spits out both the type
  of the result and a the result itself.
- Function application in OCaml is syntactically unusual, in that
  function arguments are written out separated by spaces, rather than
  being demarcated by parens and commas.
- OCaml carefully distinguishes between `float`, the type for floating
  point numbers and `int`.  The types have different literals (_e.g._,
  `6.` instead of `6`) and different infix operators (_e.g._, `+.`
  instead of `+`).  This can be a bit of a nuisance, but it has its
  benefits, since it makes it prevents some classes of bugs that arise
  from confusion between the semantics of `int` and `float`.

We can also create variables to name the value of a given expression,
using the `let` syntax.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let x = 3 + 4;;
val x : int = 7
# let y = x + x;;
val y : int = 14
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  
After a new variable is created, the toplevel tells us the name of the
variable, in addition to its type and value.

## Functions and Type Inference

The `let` syntax can also be used for creating functions:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let square x = x * x ;;
val square : int -> int = <fun>
# square (square 2);;
- : int = 16
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that we're creating more interesting values, the types have gotten
more interesting too.  `int -> int` is a function type, in this case
indicating a function that takes an `int` and returns an `int`.  We
can also write functions that take multiple arguments:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let abs_diff x y =
    abs (x - y) ;;
val abs_diff : int -> int -> int = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

and even functions that take other functions as arguments:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let abs_change f x =
    abs_diff (f x) x ;;
val abs_change : (int -> int) -> int -> int = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This notation for multi-argument functions may be a little surprising
at first, but we'll explain where it comes from when we get to
function currying in Chapter {???}.  For the moment, think of the
arrows as separating different arguments of the function, with the
type after the final arrow being the return value of the function.
Thus, 

~~~~~~~~~~~~~~~~~~~~~~~~~
int -> int -> int
~~~~~~~~~~~~~~~~~~~~~~~~~

describes a function that takes two `int` arguments and returns an
`int`, while 

~~~~~~~~~~~~~~~~~~~~~~~~~
(int -> int) -> int -> int
~~~~~~~~~~~~~~~~~~~~~~~~~

describes a function of two arguments where the first argument is
itself a function.

The types are quickly getting more complicated, and you might at this
point ask yourself how OCaml determines these types in the first
place.  Roughly speaking, OCaml infers the type of an expression from
what it already knows about the types of the elements of that
expression.  This process is called _type-inference_.  As an example,
in `abs_change`, the fact that `abs_diff` takes two integer arguments
lets the compiler infer that `x` is an `int` and that `f` returns an
`int`.

Sometimes, the type-inference system doesn't have enough information
to fully determine the concrete type of a given value.  Consider this
example.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let first_if_true test x y =
    if test x then x else y;;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This function takes a function called `test`, and two values, `x` and
`y`, where `x` is to be returned if `test x` is `true`, and `y`
otherwise.  So what's the type of `first_if_true`?  There are no
obvious clues such as arithmetic operators to tell you what the type
of `x` and `y` are.  Indeed, it seems like one could use this
`first_if_true` on values of any type, as long as `test` was able to
take that type as an input.  Indeed, if we look at the type returned
by the toplevel:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
val first_if_true : ('a -> bool) -> 'a -> 'a -> 'a = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

we see that rather than choose a particular type for the value being
tested, OCaml has introduced a _type variable_ `'a`.  Type variables
are used to express that a type is generic.  So, a type containing a
type variable `'a` can be used in a context where `'a` is replaced
with any concrete type.  So, we can write:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let long_string s = String.length s > 3;;
val long_string : string -> bool = <fun>
# first_if_true long_string "foo" "bar";;
- : string = "bar"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

And we can also write:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let big_number x = x > 3;;
val big_number : int -> bool = <fun>
# first_if_true big_number 4 3;;
- : int = 4
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

But we can't mix and match two different concrete types for `'a` in
the same use of `first_if_true`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# first_if_true big_number "foo" "bar";;
Characters 25-30:
  first_if_true big_number "foo" "bar";;
                           ^^^^^
Error: This expression has type string but
    an expression was expected of type int
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

While the `'a` in the type of `first_if_true` can be instantiated as
any concrete type, it has to be the same concrete type in all of the
different places it appears.  This kind of genericity is called
_parametric polymorphism_, and is very similar to generics in C# and
Java.

## Tuples, Options, Lists and Pattern-matching

### Tuples

So far we've encountered a handful of basic types like `int`, `float`
and `string` as well as function types like `string -> int`.  But we
haven't yet talked about any datastructures.  We'll start by looking
at a particularly simple datastructure, the tuple.  You can create a
tuple by joining values together with a comma:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let tup = (3,"three")
val tup : int * string = (3, "three")
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The type, `int * string` corresponds to set of pairs of `int`s and
`string`s.  (For the mathematically inclined, the `*` character is
used because the space of all 2-tuples of type `t * s` effectively
corresponds to the Cartesian product of `t` and `s`.)

You can extract the components of a tuple using OCaml's
pattern-matching syntax Here's a function for computing the distance
between two points on the plane, where each point is represented as a
pair of `float`s.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let distance p1 p2 =
    let (x1,y1) = p1 in
    let (x2,y2) = p2 in
    sqrt ((x1 -. x2) ** 2. +. (y1 -. y2) ** 2)
;;
val distance : float * float -> float * float -> float = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can make this code more concise by doing the pattern matching on
the arguments to the function directly.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let distance (x1,y1) (x2,y2) =
    sqrt ((x1 -. x2) ** 2. +. sqr (y1 -. y2) ** 2.)
;;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is just a first taste of pattern matching.  Pattern matching
shows up in many contexts, and turns out to be a surprisingly powerful
tool.

### Options

Another common datastructure in OCaml is the `option`.  An `option` is
used to express that a value that might or might not be present.  For
example,

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let divide x y =
    if y = 0 then None else Some (x/y)
val divide : int -> int -> int option = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here, `Some` and `None` are explicit tags that are used to construct
an optional value.  

Options are important because they are the standard way in OCaml to
encode a value that might not be there.  By default, values in OCaml
are non-nullable, so if you have a function that takes an argument of
type `string`, it's guaranteed to actually get a well-defined value of
type `string` when it is invoked.  This is different from most other
languages, including Java and C#, where objects are by default
nullable, and as a result, the type system does little to defend you
from null pointer exceptions at runtime.

Given that in OCaml ordinary values are not nullable, you need some
other way of representing values that might not be there, and the
`option` type is the most common solution.

To get a value out of an option, we use pattern matching, as we did
with tuples.  Consider the following simple function for printing a
log entry given an optional time and a message.  If no time is
provided (_i.e._, if the time is `None`), the current time is computed
and used in its place.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let print_log_entry maybe_time message =
    let time =
      match maybe_time with
      | Some x -> x
      | None -> Time.now ()
    in
    printf "%s: %s\n" (Time.to_string time) message
val print_log_entry : Time.t option -> string -> unit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here, we use a new piece of syntax, the `match` statement, to do the
pattern matching.  A `match` statement lets you do a case analysis
driven by the shape of a datastructure, and it can be used for many
different datastructres in OCaml.

This is the basic shape of a match statement.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
match <expr> with
| <pattern1> -> <expr1>
| <pattern2> -> <expr2>
| ...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first pattern that matches the structure of the expression between
the `match` and the `with` is chosen, and the right-hand side of the
`->` is evaluated, and is the result of evaluating the entire
expression.  As with `print_log_entry`, the pattern can also create
new variables, giving a name to sub-components of the datastructure
being matched.

But we don't necessarily need to use the `match` statement in this
case.  Core has a whole module full of useful functions for dealing
with options.  For example, we could rewrite `print_log_entry` using
`Option.value`, which returns the content of an option, or a default
value if the option is `None`.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let print_log_entry maybe_time message =
    let time = Option.value ~default:(Time.now ()) maybe_time in
    printf "%s: %s\n" (Time.to_string time) message
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Lists

Tuples let you combine a fixed number of items, potentially of
different types, together in one datastructure.  Lists let you hold
any number of items of the same type in one datastructure.  For
example:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let languages = ["OCaml";"Perl";"French";"C"];;
val languages : string list = ["Perl"; "OCaml"; "French"; "C"]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can access the elements of a list using pattern-matching.  List
patterns have two key components: `[]`, which represents the
empty-list, and `::`, which connects an element at the head of a list
to the remainder of the list.  Using these along with a recursive
function call, we can do things like define a function for summing the
elements of a list.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let rec sum l =
    match l with
    | [] -> 0
    | hd :: tl -> hd + sum tl
  ;;
val sum : int list -> int
# sum [1;2;3;4;5];;
- : int = 15
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We had to add the `rec` keyword in the definition of `sum` to allow
for `sum` to refer to itself.  We can introduce more complicated list
patterns as well.  Here's a function for destuttering a list, _i.e._,
for removing sequential duplicates.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let rec destutter list =
    match list with
    | [] -> []
    | hd1 :: (hd2 :: tl) ->
      if hd1 = hd2 then destutter (hd2 :: tl)
      else hd1 :: destutter (hd2 :: tl)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Actually, the code above has a problem.  If you type it into
the top-level, you'll see this error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Warning 8: this pattern-matching is not exhaustive.
Here is an example of a value that is not matched:
_::[]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is warning you that we've missed something, in particular that
our code doesn't handle one-element lists.  That's easy enough to fix
by adding another case to the match:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let rec destutter list =
    match list with
    | [] -> []
    | [hd] -> [hd]
    | hd1 :: (hd2 :: tl) ->
      if hd1 = hd2 then destutter (hd2 :: tl)
      else hd1 :: destutter (hd2 :: tl)
val destutter : 'a list -> 'a list = <fun>
# destutter ["hey";"hey";"hey";"man!"];;
- : string list = ["hey"; "man!"]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Note that in the above, we used another variant of the list pattern,
`[hd]`, to match a list with a single element.  We can do this to
match a list with any fixed number of elements, _e.g._, `[x;y;z]` will
match any list with exactly three elements, and will bind those
elements to the variables `x`, `y` and `z`.

So far, we've built up all of our list functions using pattern
matching and recursion.  But in practice, this isn't usually
necessary.  Just like there's an `Option` module with useful functions
for dealing with options, there's a `List` module with useful
functions for dealing with lists.  For example:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# List.map ~f:String.length languages;;
- : int list = [5; 4; 6; 1]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`List.map` is a function that takes a list and a function for
transforming elements of that list, and returns to us a new list with
the transformed elements.

There's another new piece of syntax to learn here: labeled arguments.
`String.length` is passed with the label, `~f`.  Labeled arguments are
arguments that are specified by name rather than position, which means
they can be passed in any order.  Thus, we could have written
`List.map ~f:String.length languages` instead of `List.map languages
~f:String.length`.  We'll see why labels are important in Chapter
_{??Functions??}_.

## Records and Variants

So far, we've only looked at datastructures that were pre-defined in
the language, like lists and tuples.  But OCaml also allows us to
define new datatypes.  Here's a toy example of a datatype representing
a point in 2-dimensional space:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# type vec2d = { x : float; y : float };;
type vec2d = { x : float; y : float; }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`vec2d` is a _record_ type, which you can think of as a tuple where
the individual fields are named, rather than being defined
positionally.  Record types are easy enough to construct:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let v = { x = 3.; y = -4. };;
val v : vec2d = {x = 3.; y = -4.}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

And we can get access to the contents of these types using pattern
matching:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
# let magnitude { x = x; y = y } = sqrt (x ** 2. +. y ** 2.);;
val magnitude : vec2d -> float = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

In the case where we want to name the value in a record field after
the name of that field, we can write the pattern match even more
tersely.  Instead of writing `{ x = x }` to name a variable `x` for
the value of field `x`, we can write `{ x }`.  Using this, we can
rewrite the magnitude function as follows.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
# let magnitude { x; y } = sqrt (x ** 2. +. y ** 2.);;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

We can also use dot-syntax for accessing record fields:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
# let distance v1 v2 =
     magnitude { x = v1.x -. v2.x; y = v1.y -. v2.y };;
val distance : vec2d -> vec2d -> float = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 

And we can of course include our newly defined types as components in
larger types, as in the following types, each of which representing a
different geometric object.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# type circle = { center: vec2d; radius: float } ;;
# type rect = { lower_left: vec2d; width: float; height: float } ;;
# type segment = { endpoint1: vec2d; endpoint2: vec2d } ;;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, imagine that you want to combine multiple of these scene objects
together, say as a description scene containing multiple objects.  You
need some unified way of representing these objects together in a
single type.  One way of doing this is using a _variant_ type:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# type shape = | Circle of circle
               | Rect of rect
               | Segment of segment;;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can think of a variant as a way of combining different types as
different possibilities.  The `|` character separates the different
cases of the variant, and each case has a tag (like `Circle`, `Rect`
and `Segment`) to distinguish each case from the other.  Here's how we
might write a function for testing whether a point is in the interior
of one of a list of `shape`s.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# let is_inside_shape vec shape =
     match shape with
     | Circle { center; radius } ->
       distance center vec < radius
     | Rect { lower_left; width; height } ->
       vec.x > lower_left.x && vec.x < lower_left.x +. width
       && vec.y > lower_left.y && vec.y < lower_left.y +. height
     | Segment _ -> false
     ;;
val is_inside_shape : vec2d -> shape -> bool = <fun>
# let is_inside_shapes vec shapes =
     List.for_all shapes ~f:(fun shape -> is_inside_shape vec shape)
val is_inside_shapes : vec2d -> shape list -> bool = <fun>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You might at this point notice that the use of `match` here is
reminiscent of how we used `match` with `option` and `list`.  This is
no accident: `option` and `list` are really just examples of variant
types that happen to be important enough to be defined in the standard
library (and in the case of lists, to have some special syntax).