# Introduction

Today we're going to be talking to about Typed Racket. Typed Racket adds an optional static type system to Racket, formerly known as PLT Scheme. Unlike untyped Racket, Typed Racket can allow you to ensure that certain aspects of your programs' behavior are correct before they run.

Typed Racket is designed to allow you to write programs in a style similar to how you would write programs in untyped Racket. The idea is that you should be able to modify your previously untyped Racket programs as little as possible, and reuse them within Typed Racket wherever possible, having Typed Racket infer types wherever it can. Typed Racket achieves this by equipping its type system with a number of features, notably: 

 * Occurrence typing: derive types from predicate tests inside the body of a definition.
 * Subtyping: e.g. ```Any``` is the top type, ```Nothing``` is the bottom type and ```Real``` is a subtype of ```Number```.
 * All values are types: e.g. ```3``` is a subtype of ```Integer```.
 * Untagged union types: e.g. ```(U String Number)```.
 
We'll have a closer look at these particular features of Typed Racket's type system later on in this workshop - and look at examples of how previously untyped Racket code can often type check with minimal modification.

Let's first get you using Typed Racket. We'll look at some more familiar features of Typed Racket's type system, having you add type annotations to examples of previously untyped Racket code.

# Type Annotations

In this section, we will explore the basic features of Typed Racket's type system, type-annotating examples of previously untyped Racket code.

### Type Basic Value Definitions
  
   We'll start looking at some basic types in Typed Racket. First, let's type-annotate a definition for a person's     full name. 

   ```racket
   (define full-name "John Smith")
   ```
   
   We can type-annotate this definition by saying that `full-name` is a `String`.

   ```racket
   (: full-name String)
   (define full-name "John Smith")
   ```

   Exercise: type-annotate the age of the person. 
  
   ```racket
   (define age 28)
   ```

   You might say that ```age``` is an ```Integer```.
  
   ```racket
   (: age Integer)
   (define age 28)
   ```

   However, you could be more precise by saying that ```age``` is
   a ```Positive-Integer```.
  
   ```racket
   (: age Positive-Integer)
   (define age 28)
   ```
   
   There's a diagram of Type Racket's numeric type hierarchy in the paper titled [Typing the Numeric Tower](http://www.ccs.neu.edu/home/stamourv/papers/numeric-tower.pdf), which may be of use to you.
   
### Type Basic Function Definitions
  
  We'll now look at function types in Typed Racket. Let's type a function that takes the `gcd` of two integers.

  ```racket
  (define (gcd a b)
    (cond [(= b 0) a]
          [(gcd b (abs (modulo a b)))]))
  ```
  
  This function can be given the type `(-> Integer Integer Integer)` since it works for negative integers as well, e.g. `(gcd -3 4) => 1`.
  
  ```racket
  (: gcd (-> Integer Integer Integer))
  (define (gcd a b)
    (cond [(= b 0) a]
          [(gcd b (abs (modulo a b)))]))
  ```
  
  However, it is also possible to implement a `gcd` function using only substraction; in this case the function won't terminate if given a negative integer as one of its inputs. 
  
  ```racket
  (define (gcd m n)
    (cond [(< m n) 
           (gcd m (- n m))]
          [(> m n)
           (gcd (- m n) n)]
          [else m]))
  ```
  
  In untyped Racket, we could impose the constraint that this function has to be given non-negative integers as input at run-time with contracts. 
  
  ```racket
  (define/contract (gcd m n)
    (-> exact-nonnegative-integer?
        exact-nonnegative-integer?
        exact-nonnegative-integer?)
    (cond [(< m n) 
           (gcd m (- n m))]
          [(> m n)
           (gcd (- m n) n)]
          [else m]))
  ```
  
  In Typed Racket, we'd like to be able to impose this constraint before our programs that use this function run.    However, we are unable to guarantee with types before our program runs that our function will take two non-negative integers and give as a non-negative integer as a result: we have to cast the result of subtracting our two non-negative integers to a non-negative integer at run-time in order for our programs that use this function to type check. 
  
  
  ```racket
  (: gcd 
     (-> Exact-Nonnegative-Integer
         Exact-Nonnegative-Integer 
         Exact-Nonnegative-Integer))
  (define (gcd m n)
    (cond [(< m n) 
           (gcd m 
                (cast (- n m) Exact-Nonnegative-Integer))]
          [(> m n)
           (gcd (cast (- m n) Exact-Nonnegative-Integer) n)]
          [else m]))
  ```
  
  Exercise: type-annotate the definition of a function that computes the n-th fibonacci number.
  
  ```racket
  (define (fib n)
    (cond [(<= n 2) 1]
          [else (+ (fib (- n 1))
                   (fib (- n 2)))]))
  ```
  
  You could say that the type-signature of this function should be `(-> Integer Integer)`. 
  
  ```racket
  (: fib 
     (-> Integer 
         Integer))
    (define (fib n)
    (cond [(<= n 2) 1]
          [else (+ (fib (- n 1))
                   (fib (- n 2)))]))
   ```
   
   However, we want this function to take a positive integer and give us a positive integer: i.e. `(fib 1)` should give us the first fibonacci number. Though, we suffer from the same problem that we saw with our `gcd` function: we have to cast the result of subtracting numbers at run-time to a positive integer in order for our programs that use `(fib n)` to type check. 
  
   ```racket
   (: fib 
     (-> Positive-Integer
         Positive-Integer))
   (define (fib n)
     (cond [(<= n 2) 1]
           [else (+ (fib (cast (- n 1) Positive-Integer))
                    (fib (cast (- n 2) Positive-Integer)))]))
   ```
  
  Though that's the best we can currently do in Typed Racket. 
  
### Record Types
  
  Now we'll look at record types in Typed Racket; record types are like product types where each field has a name.
  In Racket and Typed Racket, record types are called ```struct```s. Here's an example of a `struct` in untyped    Racket: a ```struct``` for student's, where each student has field for their name, age, faculty and term. Untyped Racket supports ```struct```s, though it only knows about these ```struct```s at run-time. 
  
  ```racket
  (struct student (id name age faculty term))
  ```
  ```
  > (student "00000999" "John Smith" 20 'mathematics '2A)
  #<student>
  ```
  
  Suppose that we'd want to impose the constraints that each student's:
  
  * `id` is a non-negative integer between 0 and 99999999.
  * `name` is a ```String```.
  * `age` is a positive integer between 16 and 80.
  * `faculty` is either `Mathematics`, `Science`, `Engineering`, `Arts` or `Applied Health Sciences`.
  * `term` is either `1A`, `1B`, `2A`, `2B`, `3A`, `3B`, `4A` or `4B`. 
  
We don't want to be able to create student instances that don't satisfy these constraints; 
  student instances who don't satisfy these constraints would be invalid.

  In untyped Racket, we could impose these constraints at run-time with contracts.
    
  In Typed Racket, we can impose some of these constraints at compile-time with
  types.
  
  It's tricky to compose the contrainst that each student's `id` is a non-negative integer between 0 and 99999999 with types in Typed Racket. It's also tricky to impose the constraint that their age is a positive integer between 16 and 80. Though, 
  recall that in Typed Racket, all values are types: e.g. `16` is a subtype of `Positive-Integer`.
  We also have untagged union types in Type Racket: e.g. `(U Number String)`, meaning that 
  the inhabitants of this type can either be a `Number` or `String`, such as `3.0` or `"foo"`.
  
  We can quite easily certify that each student's term and faculty are valid at compile-time,
  using untagged union types and the fact that all values are types in Typed Racket: 
  
  ```racket
  (define-type Term (U '1A '1B '2A '2B '3A '3B '4A '4B))
  ```
    
  ```racket
  (define-type Faculty (U 'mathematics 'science 'engineering 'arts 'applied-health-sciences))
  ```
  
  However, it's trickier to certify that each student's `id` and `age` are meet our specification
  for valid `id`s and `ages` at compile-time; to do this, we'd have to enumerate all possible 
  valid `ids` and `ages`, storing these possibilities in a untagged union types that depend on 
  values like `Term` and `Faculty`. 
  
  For the sake of this example, let's simplify our specification for valid `id`s and `ages`:
  
  We'll say that a valid `age` is a positive integer between 16 and 25: 
  
  ```
  (define-type Age (U 16 17 18 19 20 21 22 23 24 25))
  ```
  
  We'll say that a valid `id` is a 1-digit string containing only numeric characters:
  
  ```
  (define-type Id (U 0 1 2 3 4 5 6 7 8 9))
  ```
  
  Then we can define our student record type in Typed Racket as follows, and impose these
  constraints at compile-time:
  
  ```racket
  (struct: Student 
           ([id : Id] 
            [name : String] 
            [age : Age]
            [faculty : Faculty]
            [term : Term]))
  ```
    
  ```racket
  > (Student 9 "John Smith" 20 'mathematics '2A)
  - : Student
  #<Student>
  ```
  
  This concludes are example of record types in Typed Racket. Now let's look at some more exercises.
  
  Exercise: represent the Peano numbers in Typed Racket using record types.
  
  Hint: you'll need to define three `struct`s - one for `Nat`, one for `Z` and one for successors `S` of some    natural number. You should also impose the constraint that `Z` and `S` are subtypes of `Nat`. If you happen to be familiar with Scala and Scala's type system, think about how you would represent the Peano numbers using case classes to answer this question. 
  
  ```racket
  (struct: Nat ())
  (struct: Z Nat ())
  (struct: S Nat ((n : Nat)))
  ```
  
  Subtyping gets in the way with type-inference in Typed Racket: you have to use ```ann``` to explicitly annotate 
  that certain naturals are indeed instances of `Nat` and not just instances of `Z` or `S` in Typed Racket. 
  
  For instance: 
  
  ```racket
  > (Z)
  - : Z
  #<Z>
  ```
  
  ```racket
  > (S (Z))
  - : S
  #<S>
  ```
  
  Might often have to explicitly annotated as `Nat`, using the `ann` form: 
  
  ```racket
  > (ann (Z) Nat)
  - : Nat
  #<Z>
  ```
  
  ```racket
  > (ann (S (Z)) Nat)
  - : Nat
  #<S>
  ```
  
  This concludes our exercise for defining the Peano numbers in Typed Racket using record types.
  
  Remark: if you were working in a programming language with algebraic data types, such as Haskell 
  or Idris, you might define a representation of the Peano numbers as follows: 
  
  ```haskell
  data Nat = Z | S Nat
  ```
  
  Typed Racket currently lacks support for algebraic data types, and it's pattern-matching construct ```match```
  does not provide compile-time case coverage for all variants of a data type. We'll look at adding algebraic data 
  types and a ```type-case``` construct that provides compile-case coverage later on this workshop.
  
  I simplified the specification for valid ```id```s and ```age```s of student's in my previous example of record types in Typed Racket. I said that you'd have to enumerate
  all possible valid `id`s and `age`s, storing them in untagged union types that depend on values in 
  order to make types that allow to certain that the `id`s and `age`s of students are correct before 
  our programs run in Typed Racket. It's possible to write functions that enumerate all possible `id`s 
  and `age`s at run-time - you could even write programs to do this and then wrap the lists of results,
  pasting them into types for `Id` and `Age` in Dr. Racket.
  
  However, in Racket we have hygienic macros. Macro expansion takes place before type checking occurs 
  in Typed Racket. We can e.g. create a `define-range-type` macro in Typed Racket that
  takes a type name, a lower-bound integer & an upper-bound integer as input, and then defines 
  an untagged union type with all integer values in our range as its inhabitants.
  
  Exercise: define such a macro. 
  
  ```racket
 (define-syntax
   (define-range-type stx)
   (syntax-case stx ()
     [(_ type-name lower-bound upper-bound)
      (letrec [(range (λ (a b)
                         (if (<= a b)
                           (cons a (range (+ 1 a) b))
                           '())))]
        (quasisyntax
          (define-type type-name
            (U (unsyntax-splicing
                 (range (syntax->datum (syntax lower-bound))
                        (syntax->datum (syntax upper-bound))))))))]))
 ```
 
 Check that your ```define-range-type``` macro correct defines our `Age` type correctly.
 
 ```racket
 (define-range-type Age 16 80)
 ```
 
 ```racket
 > (:type Age)
 (U 16 17 ... 80)
 ```
 
### Parametric Polymorphism
 
  Typed Racket supports parametric polymorphism. You can define types (and record types) that
  take types as parameters to construct new types. For instance, you can define an ```Option```
  type, like you might have seen in Scala, that takes a type and gives you a type; option
  types allow you to e.g. create functions where you might get a result of a particular
  type if your input to the function is valid at run-time (```(Some a)```),  or get an empty result (`(None)`)
  otherwise.

  ```racket
  (struct: None ())
  (struct: (a) Some ([v : a]))
 
  (define-type (Option a) (U None (Some a)))
  ```
  
  Remark: you'll have to use ```ann``` in many cases, just like what we saw in our ```Nat```,
  to convince Typed Racket that instances of ```(None)``` or ```(Some a)``` are ```Option```s.
  
  Exercise: define a type for ```Either``` that takes two type parameters ```a b```, and is either a ```(Left a)```
  or a ```(Right b)```.
  
  ```racket
  (struct: (a) Left ([v : a]))
  (struct: (b) Right ([v : a]))
  
  (define-type (Either a b) (U (Left a) (Right b)))
  ```
  
  Remark: in programming languages that support algebraic data types, like Haskell or Idris, you might define 
  polymorphic ```Option``` and ```Either``` data types as follows: 
  
  ```haskell
  data Option a = None | Some a
  ```
  
  ```haskell
  data Either a b = Left a | Right b
  ```
  
  We could make it more convenient to define ```Option``` or ```Either``` data types in Typed Racket by adding  support for algebraic data types to Typed Racket. As mentioned, we'll look at how we might add algebraic data types to Typed Racket later on in this workshop.
  
 Exercise: type a polymorphic `chunk` function. 

 ```racket
 (define (chunk xs n)
   (cond [(empty? xs) empty]
         [(< (length xs) n)
          (cons xs empty)]
         [else (cons (take xs n)
                     (chunk (drop xs n) n))]))
 ```
   
 ```racket
 (All (a) (-> (Listof a) Integer (Listof (Listof a)))))
 (define (chunk xs n)
   (cond [(empty? xs) empty]
         [(< (length xs) n)
             (cons xs empty)]
         [else (cons (take xs n)
                     (chunk (drop xs n) n))]))
 ```
  
### Recursive Type Constructors
  
  Typed Racket also supports recursive type constructors. This meanings that you can define 
  types that refer to themselves, e.g. a binary tree is either a leaf of some type or it 
  branches off into two binary trees.
  
  We can define a recursive polymorphic type constructor for a ```BinaryTree``` in Typed Racket as follows:

  ```racket 
  (define-type (BinaryTree A) (Rec BT (U A (Vector BT BT))))
  ```
  
  And, as another example, we can define a recursive polymorphic type constructor for a ```QuadTree``` in 
  Typed Racket as follows: 
  
  ```racket
  (define-type (QuadTree A) (Rec QT (U A (Vector QT QT QT QT))))
  ```
  
  Exercise: define a recursive polymorphic type constructor for an ```OctTree``` in Typed Racket.
  
  ```
  (define-type (OctTree A) (Rec QT (U A (Vector QT QT QT QT QT QT QT QT))))
  ```
  
  Exercise: we can also define a type for the Peano numbers recursively in Typed Racket; though you
  still need to define record types for ```Z``` and a polymorphic ```S``` because Typed Racket 
  doesn't really support tagged union types. Try this.
  
  ```racket
  (struct: Z ())
  (struct: (a) S ((n : a)))
  
  (define-type Nat (Rec Nat (U Z (S Nat))))
  ```
  
  Remark: I think it's possible to emulate recursive types with only record types and subtyping in Typed Racket.   Luckily, `Z` and `S` in this case are still treated as subtypes of `Nat`: we can still explicitly type annotate
  `Z` or `(S n)` as `Nat` when we need to.
  
  Exercise: define a function that converts a `Nat` to a `Number` in Typed Racket. 
  
  Hint 1: use Racket's `match` construct.
  
  Hint 2: type-annotate this function.
  
  ```racket
  (define (nat->number nat)
    (match nat
      [(S n)
       (+ (nat->number n) 1)]
      [(Z) 0]))
  ```
  
  Answer:

  ```racket
  (: nat->number (-> Nat Number))
  (define (nat->number nat)
    (match nat
      [(S n)
       (+ (nat->number n) 1)]
      [(Z) 0]))
  ```
  
### Occurrence Types

Occurrence types allow us to derive types from predicate tests inside the body of a definition. This feature of Typed Racket's type system is useful for allowing us to use previously untyped Racket code in Typed Racket with minimal modification. 

```racket
(: foo 
   (-> (U String Any)
       (U Integer String)))
(define (foo str-or-any)
  (cond [(string? str-or-any) 
         (string-length str-or-any)]
        [else "error"]))
```

We'll see in the next section how Typed Racket's type system allows us to use previously untyped Racket code in Typed Racket with minimal modification.

# Break

This is an opportunity to grab a coffee or ask questions.

# No Type-Annotations Needed

As mentioned, Typed Racket aims to allow you to write programs in a style similar to how you would write programs in untyped Racket. Several of the example programs on the Racket homepage type check in Typed Racket without modification.

* Program 1:

  ```racket
  #lang racket
  ;; Finds Racket sources in all subdirs
  (for ([path (in-directory)]
       #:when (regexp-match? #rx"[.]rkt$" path))
    (printf "source file: ~a\n" path))
  ```
* Program 2:

  ```racket
  #lang racket
  
  (define listener (tcp-listen 12345))
  
  (let echo-server ()
    (define-values (in out) (tcp-accept listener))
    (thread (lambda () (copy-port in out)
                       (close-output-port out)))
    (echo-server))
  ```

  ```racket
  #lang typed/racket

  (define listener (tcp-listen 12345))
  
  (let echo-server ()
    (define-values (in out) (tcp-accept listener))
    (thread (lambda () (copy-port in out)
                       (close-output-port out)))
    (echo-server))
  ```
  
* Program 3:
  
  ```racket
  #lang racket

  ;; Report each unique line from stdin
  (define seen (make-hash empty))
  
  (for ([line (in-lines)])
    (unless (hash-ref seen line #f)
    (displayln line))
  (hash-set! seen line #t))
  ```

  ```racket
  #lang typed/racket

  ;; Report each unique line from stdin
  (define seen (make-hash empty))
  
  (for ([line (in-lines)])
    (unless (hash-ref seen line #f)
    (displayln line))
  (hash-set! seen line #t))
  ```
  
* Exercise: find another program on the Racket homepage that type checks without modification.

# Adding Algebraic Data Types to Typed Racket

We saw in the previous section how Typed Racket type system let's use previous untyped Racket code in Typed Racket with minimal modification. Though, it felt rather cumbersome to define `Nat`, `Option` and `Either` using record types and subtyping in Typed Racket. We also couldn't provide compile-type case coverage of variants of a data type using `match`. Moreever, the style in which we may prefer to program in a typed functional programming language like Typed Racket may not reflect the style in which we have tended to program in untyped Racket; e.g. dispatching by predicates test inside the body of definitions at run-time. 

It would be better if we could make it easier to define data types like `Nat`, `Option` and `Either`, as well as provide compile-time case coverage for when we have definitions that dispatch by pattern-matching on these data types. In fact, Andrew Kent, a graduate student at Indiana University, has been working on a solution to this problem: he has been working on a module that adds ML-like algebraic data types to Typed Racket.

Let's look back at how we defined `Nat`, `Option` & `Either` using record types and subtyping in the previous sections. We want to make it easier to define these data types; though we want these data types to be functionally the same, even if we make it easier to define them. We'll create a `define-datatype` macro, that makes it easier to define these data types:
 
  ```racket 
  (require datatype)
  ```
 
  ```racket
  (define-datatype Nat
    [S (Nat)]
    [Z ()])
  ```
  
  Exercise: check that this definition macro-expands to the same definition 
  that we made for `Nat` using record types and subtyping in the previous sections.
  Hint: use the macro-expand in Dr. Racket.
  
  Now, recall the `nat->number` function that we made at one point. It 
  works here too for the definition of `Nat` that we just made using
  `define-datatype`.
  
  However, suppose that we forgot to provide case-coverage
  for `Z`:
 
  ```racket
  (: nat->number (-> Nat Number))
  (define (nat->number nat)
    (match nat
      [(S n)
       (+ (nat->number n) 1)]))
  ```
  
  If we don't provide case-coverage for `Z`, then our program will fail at run-time:
 
  ```racket
  > (nat->number (Z))
  - : Number
  match: no matching clause for (Z)
  ```
  
  We can create a `type-case` construct that ensures that we provide full case-coverage
  for all variants of `Nat`, or else or program won't compile:
  
  ```racket
  (: nat->number (-> Nat Number))
  (define (nat->number nat)
    (type-case Nat nat
      [(S n) => (+ (nat->number n) 1)])
  ```
  
  ```racket
  type-case: missing case(s) for the following: (Z)
 in: (syntax (type-case Nat nat ((S n) => (+ (nat->number n) 1))))
  ```
  
  Exercise: fix our `nat->number` function so that our program compiles.
  
  ```racket
  (: nat->number (-> Nat Number))
  (define (nat->number nat)
    (type-case Nat nat
      [(S n) => (+ (nat->number n) 1)]
      [(Z) => 0]))
  ```
  
  Test that it works:
  
  ```racket
  > (nat->number (Z))
  - : Number
  0
  ```
  
  ```racket
  > (nat->number (S (S (S (Z)))))
  - : Number
  3
  ```
  
  Exercise: define an algebraic data type for `Option`.
  
  ```racket
  (define-datatype (Option a)
    [Some (a)]
    [None ()])
  ```
  
  Exercise: define an algebraic data type for `Either`.
  
  
  ```racket
  (define-datatype (Either a b)
    [Left (a)]
    [Right (b)])
  ```
  
  Exercise: check that these definitions macro-expand to the definitions
  we made for `Option` and `Either` using record types in the previous 
  section. Hint: press the macro-expander button in Dr. Racket.
  
# What Next?

We've talked about:

* The features of Typed Racket's type system.
* Typed-annotated previously untyped Racket code.
* Showed how Typed Racket's type system is designed to allow us to use our previously untyped Racket code in Typed Racket with as little modification as possible.
* Added features to Typed Racket's type system to make it easier for us to ensure that our program's satisfy certain constraints before they run:
  *  Range types made it easier for us to make sure that only valid students would be created before our programs run.
  *  Algebraic data types and the type-case construct made it easier for us to define types for `Nat`, `Option` and `Either`. Our type-case construct allowed us to ensure that we provided full case coverage for all variants of our algebraic data types when pattern matching on them before our programs run.

But, how might we improve Typed Racket?

* There are several Racket libraries that still need explicit type-annotations added to them, e.g. [How do to Design Programs](https://github.com/lexi-lambda/racket-2htdp-typed).
* There are currently soundness bugs in Typed Racket, where the compile-time type of a term differs from its run-time type.
  * Example: [call/cc + letrec + occurrence typing can be unsound](https://github.com/racket/typed-racket/issues/128)

  ```racket
  (define-type klk [(List klk Boolean) -> Nothing])
  
  (: foo : [-> False])
  (define (foo)
    (letrec ([x (let/cc k : (List klk Boolean)
                (list k #f))])
    (if (false? (second x))
        (begin
          (let/cc k : (List klk Boolean)
            ((first x) (list k #t)))
          (second x))
        ((first x) x))))
  ```

  ```
  > (foo)
  #t
  ```
* In the future, I'd like to try adding an experimental dependently typed functional programming language to Racket,
  e.g. ```#lang dependent/racket```. I wouldn't want this language to try to accodomate the style that untyped Racket
  programmers program in, nor would I strive for untyped Racket code. I'm not a fan of gradual typing: I find too
  much typing to be bad, in the sense that many incorrect untyped Racket programs type check in Typed Racket
  without having to be corrected first. Typed Racket is designed to accodomate and make the most out of sloppy thinking,
  not necessary correct or disallow it.
* I'd be interested in trying to add Haskell-style type classes and adhoc-polymorphism to Typed Racket.
* I'd be interested in trying to add higher-kinded types to Typed Racket.
* I'd be happy if you were willing to join me in pursuing these endeavours!

# Resources

* [(Somewhat) Algebraic Data Types for Typed Racket](https://github.com/andmkent/datatype)
* [The Typed Racket Guide](http://docs.racket-lang.org/ts-guide/index.html)
* [Racketcon 2012: Sam Tobin-Hochstadt - Tutorial: Typed Racket](https://www.youtube.com/watch?v=w-fVHOxeEpM)
* [The Design and Implementation of Typed Scheme](http://www.ccs.neu.edu/racket/pubs/popl08-thf.pdf)
* [Typing the Numeric Tower](http://www.ccs.neu.edu/home/stamourv/papers/numeric-tower.pdf)
* [Making a Dual Typed/Untyped Racket Library](http://unitscale.com/mb/technique/dual-typed-untyped-library.html)
