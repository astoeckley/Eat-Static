## Eat Static

#### Easy run-time type-checking and validation of Clojure maps and functions

##### A single vector DSL that joins many features in one place to *quickly*:
* Make type checks on whole maps, specific k/v pairs, or function arguments easily
* Specify and validate function return types
* Build intricate map constructors & validators with one line of code
* Construct complex functional predicates on the fly
* Build custom types on demand
* Compose existing types into new composite types easily
* Generate default maps based on simple inheritance 
* Write functions that accept named, unordered parameters
* Specify function arguments as required or optional
* Provide default values for optional arguments
* Group :pre/:post style conditions
* Leverage trait-like behavior from basic Clojure maps

... and doing these tasks with *very* minimal syntax.

***

#### The Goal

Automate several excellent tools in Clojure for writing safer code -- tools you may not use frequently because of the extra syntax overhead. Apply these tools to easy map validation for getting the most out of basic data structures.

***

**First**, if you want a good static type system that runs at compile time, look no further than [Core.Typed](https://github.com/clojure/core.typed).  Alternately, there is the interesting [core.contracts](https://github.com/clojure/core.contracts) library. Both these options add a fair amount of verbosity and additional logic to your work. Instead, Eat Static prioritizes streamlined syntax to guard how you call functions and ensure that arguments and data structures are always what you expect -- without writing much extra code. If you desire this flexibility and power without syntax bloat, Eat Static might be for you.

**Second**, Eat Static was an excellent techno group from the U.K.

**Third**, a couple good pre-requisites for using Eat Static are Sierra's [Thinking in Data](http://www.infoq.com/presentations/Thinking-in-Data) and Hickey's [Simplicity Matters](https://www.youtube.com/watch?v=rI8tNMsozo0), both of which expound the virtues of consolidating function arguments as an associative structure, and using run-time checks via assertions to make your code safer in a dynamic environment. (These checks are easily disabled when you compile for production.)

### Includes

* clojars
* :require

## Examples

Clojure is a dynamic language, and we like it that way. These run-time checks are opt-in and ad-hoc. Mix these function definitions with normal def and defn to sprinkle safety in places you want it.

### Quick Start

A normal ```df``` function definition without any validations and default values simply offers a named, unordered arg list with ease:

```clojure
(df my-function [a b c] ...)

;; instead of:

(defn my-function [{:keys [a b c] :as my-function-input}]...)

;; Additionally, the df version makes the keys a,b,c required, 
;; unlike the simple defn version which sets them to nil if they are omitted.

;; Read on to see how easy it is to change these requirements, as well as
;; how to offer custom names to items such as the :as map
```
The ```df``` family of tools simply expand to a ```defn```, ```defn-```, or ```fn``` that takes a single map argument. All the destructuring and assertions are automatically handled.

Functions created this way with or without these are called like this:

```clojure
(my-function {:a 1 :b 2 :c 4})
```
This library also adds the option to call it like this, using the ```c``` (for "call") function:

```clojure
(c my-function :a 1 :b 2 :c 4)
```

Of course, the key/value arguments may be presented in any order:

```clojure
(c my-function :c 4 :a 1 :b 2) ; same as above
```

And of course, keys not specified are simply ignored by the run-time assertions, but still available on the overall ```:as``` map:

```clojure
(c my-function :c 4 :a 1 :b 2 :e 5 :f 6)

;; this provides interesting uses for trait-like behavior, 
;; described further below
```
The syntax savings grows with argument complexity. As soon as you decide that all args are optional with a default value of 0 for each, it becomes:

```clojure
(df my-function [[a b c 0]] ...)

;; instead of:

(defn my-function [{:keys [a b c] :or {a 0 b 0 c 0}
                                  :as my-function-input}] ...)
```
Any symbol inside a vector is an optional argument, and you can assign the same default to many optionals at once. (*optional* here does not mean the same thing as in a language like Swift.)

More and more syntax is saved as argument requirements increase. 

#### Type Checks
Next, you want to ensure that ```a``` ```b``` and ```c``` are only integers:

```clojure
(df my-function [:int [a b c 0]] ...)

;; Several shortcuts are offered as well:

(df my-function [:i [a b c 0]] ...)

;; This causes an exception if you pass in non-integer values for a,b,c.

;; Equivalent:

(defn my-function [{:keys [a b c] :or {a 0 b 0 c 0}
                    :as my-function-input}]
                   {:pre [(integer? a) (integer? b) (integer? c)]}
                    ...)

;; In reality, df is implemented as assert calls, 
;; not a :pre/:post map, to offer specific error messages
;; and to handle other features described below.
;; You can freely add your own :pre/:post map to a df if desired

```
Here are a few more basic examples:
```clojure
(df my-function [a [b c 0]] ...) 
;; a is required, but b,c are optional, with same default value

(df my-function [a [b c]] ...) 
;; b,c are optional, no default value (remain nil if omitted)

(df my-function [:i a [b 5 c 6 d 7] e :n f] ...)
;; a,e,f are required and b,c,d each have a different default value
;; a,b,c,d,e must be integers, but f can be any number
;; (more details below)
```
More complex argument requirements are equally handled with ease:
```clojure
;; Preview of the validation expressions described below:

(df two-ints [(> b) a :i b (< 10) a b] ...)

;; b must be an integer and a must be greater than b, 
;; but both a and b must be less than 10.
;; if any of these fail, the whole function call fails
```

#### Doc strings and custom names for the whole input map

You have some flexibility in what you supply to the function definition:
```clojure
;; By default, the whole map is available as a symbol derived from
;; the function name. If the function is my-function, the map can 
;; be accessed in the body as my-function-input.

;; An additional parameter before the arg vector provides a custom name 
;; for the whole map that was passed in, rather than using the default 
;; name implicitly provided:

(df my-function in
 [a b c]
 ;; do something with "in" ...
 )

;; This allows you to access any parameter in the function call, 
;; regardless of whether it appears in the argument list. You could
;; use (:a in), rather than listing "a" explicitly. 

;; One particularly interesting feature is that you can reference this
;; whole input map in the arg list as well, and treat it with overall validation.
;; This has useful consequences when composing predicates and working with custom
;; types and traits. (see further below).
;; However, you cannot specify the whole input map as optional, of course.

;; Support for doc strings, of course:

(df my-function
 "A fantastic doc string"
 [a b c]
 ...
 )

;; Both a doc string and a custom name for the entire input map:

(df my-function 
 "A really great doc string"
 input-map 
 [a b c]
 ...
 )

;; The optional parameters of a custom map name and a doc string
;; can actually appear in any order:

(df my-function 
 input-map 
 "Still a really great doc string"
 [a b c]
 ...
 )
```
#### Validating a function's output

The above examples have introduced basic validation for fn inputs, and the flexibility available for much more specific and complex validation is described in the rest of this document. All forms of validation available for the argument vector are also available for asserting the output of a function's return value as well, with two caveats:

* You cannot assign a default value to any of the output data; if you try, it is simply ignored
* You do not provide symbol names of locals in the output validation, as all expressions operate directly on its return value.

Examples:
```clojure
(df validate-output
    (:i)
    [a]
    a)

;; In this example, the return of validate-output must be an integer, 
;; or a run-time assertion triggers.

;; A list that appears anywhere before the argument vector 
;; follows the same syntax as the arg vector, but in list form.

(df another
    "Output must be an integer less than 10"
    custom-input-map-name
    (:i (< 10)) ; validation expressions like this are explained below
    [:i a b]
    (+ a b))

;; The doc string, custom input map name and output validation list
;; are all optional, and may appear in any order:

(df another
    (:i (< 10))
    custom-input-map-name
    "Output must be an integer less than 10"
    [:i a b]
    (+ a b))

;; If your return value was a map, you could use the validation tools explained
;; below to validate the individual elements of the output map as well. 

;; For example, this expression requires the output of the function to be 
;; a map containing non-nil values for keys :foo and :bar --

(df has-keys-foo-bar
   ((:foo)(:bar))
   [a] ...)

;; same as this use of the pred> form, explained further below:

(df has-keys-foo-bar
   ((pred> [foo bar]))
   [a] ...)

;; Requires :foo and :bar in the output map to be integers:

(df has-keys-foo-bar
   ((pred> [:i foo bar]))
   [a] ...)

;; This requires the output to be either 0 or 1, and nothing else:

(df zero-or-one
    ((or> #(= 1 %) #(= 0 %)))
    [x] ...)

;; pred> and or> and other useful validation helpers and described further below
```
Comparison with normal Clojure code:
```clojure
;; This requires the output to be an integer greater than 10
;; OR less than -5:

(df out (:i (or> #(> % 10) #(< % -5)))
    [:i x y z] ...)

;; For comparison's sake, here is the normal Clojure way
;; to do almost the same thing:

(defn out
      [{:keys [x y z] :as out-input}]
      {:pre [(integer? x) (integer? y) (integer? z)]
       :post [(integer? %)
              (or (> % 10) (< % -5))]}
              ...)

;; (Eat Static does not use the :pre/:post map internally)
```

### Going Further

This example leverages basic primitive types and simple validation expressions. To compose these easily into complex re-usable custom types with very little code, see further below.

You want a function that creates a colored circle with a specified radius, centered at X, Y, with an optional color passed by a keyword, but a default color otherwise. The specified radius must be at least 1, and any passed keyword color must exist in a color map you have somewhere. You need X and Y to only be integers. You want to ensure this function is always called properly based on these requirements. You don't want to remember which order these arguments are passed in. And, you want the option to *quickly* add new args or default values later without breaking existing calls to the function, and without the verbosity and time of writing new multi-arity functions or editing assertions and :pre checks.

You can get halfway there by using a combination of destructuring, ```:or``` maps and ```:pre``` conditions. But doing this frequently is tedious.

The above function could be written as:

```clojure
(defn circle
  [{:keys [radius x y color] :or {color :blue}
    :as circle-input}]
  {:pre [(>= radius 1) (integer? x) (integer? y)
         (#{:blue :white} color)]}
  ...
  )
```

This library creates the equivalent fn like this:

```clojure
(df circle
  [(>= 1) radius :i x y (#{:blue :white}) [color :blue]]
  ...
  )
```
Later, you decide to take this function into 3 dimensions, but keep it working for 2 dimensions. So you want to add an optional ```z``` value. It must also be passed as an integer and will have no default value and thus should remain nil if it is excluded.

You could amend your function in Clojure by doing two things:

```clojure
(defn circle
  [{:keys [radius x y z color] :or {color :blue} ;;added z
  :as circle-input}]
  {:pre [(>= radius 1) (integer? x) (integer? y)
         (#{:blue :white} color)
         (if z (integer? z) true)]} ;;added this line
  ...
  )
```

  This library makes the change with two characters:

```clojure
(df circle
   [(>= 1) radius :i x y -z (#{:valid :colors}) [color :blue]]
  ...
  )

;; added -z
```
Validation checks like ```(>= 1)``` or ```:i``` affect only those symbols that follow until the next validator. Thus ```x``` ```y``` and ```z``` must be integers, but are not affected by ```(>= 1)```, and ```z``` must only be an integer if it is actually provided. 

Basic keyword validator syntax:

  * ```:i -z``` specifies ```z``` is an optional integer arg
  * ```:i [z]``` means the same thing as ```:i -z```
  * ```:i [z 5]``` would add a default value to the optional ```z```
  * ```:i [z w 5]``` would make both ```z``` ```w``` optional integers with a default of 5
  * ```:i [z 4 w 5]``` would make both ```z``` ```w``` optional integers with separate default values
  * ```:i z``` would mean ```z``` is a required integer argument
  * ```:n z``` would mean ```z``` is a required number, including any number type (i.e. floats)
  * Instead of the ```:n``` or ```:i``` shortcuts, all types have more verbose synonyms, like ```:num``` and ```:number``` or ```:int``` and ```:integer```

Just as ```:n``` specified ```z``` as a number, Eat Static provides easy built-in run-time type checks for all Clojure primitives:

    * number
    * integer
    * float
    * ratio
    * boolean
    * function
    * vector
    * map
    * set
    * list
    * symbol
    * string
    * character
    * keyword
    * "anything" type -- allows any value for an arg

    The anything type is also the default if an arg 
    is not preceded by any validation criteria. 

    Keyword specifiers for these types are listed at the top of this
    distribution's validations.clj file, in the map called type-checks.

Creating specific validation expressions for :pre-style checks uses syntax similar to the ```->``` macro, and allows one validation to check multiple args:

```clojure
[(>= 1) radius (neg?) x y]

;; in this example, it is not necessary to
;; specify that these are numbers since the
;; validation checks that for us

;; The same arg can be used in multiple checks:

[(>= 1) radius :i radius x y]

;; x y and radius must be integers
;; and radius must be at least 1

;; Note: the keyword validators for primitives
;; such as :i, :n and many others are only available
;; in the top level of an argument vector, and not
;; inside validation forms like (>= 1) which may
;; use keywords for other purposes.
```
Combine validation expressions with ```++```
```clojure
[(++ :i (> 1)) x y z]

;; x, y, z must all be integers greater than 1

;; Note: You may have noticed earlier that output validation lists
;; automatically require all expressions to pass for the single return
;; value, therefore they operate like ++. Thus, ++ is unnecessary and
;; not available for output validation lists.

```
Writing and providing normal functions:

The above validation expressions like ```(> 1)``` are efficient syntax similar to the thread-first macro expressions, but sometimes you want a bit more flexibility. The ```or>```, ```and>``` and ```ep>``` validation helpers let you do just that by passing any function:
```clojure
[(or> #(< 2 % 10) #(> -2 % -10)) x y]

;; x and y must both be between 2 and 10 or between -2 and -10

;; and and> would simply require that all functions are true.
;; ep> uses Clojure's every-pred to combine the predicate functions
;; into a single predicate. In nearly all cases, ep> and and> should
;; be synonmous, only the implementation is different.

;; Requires x to be an integer or a keyword:

[(or> integer? keyword?) x]

;; use the "t" function (for "type") to use the same keyword 
;; type specifier available in the arg vector:

[(or> (t :i) (t :k)) x]

;; "t" merely returns the predicate associated with the keyword

;; Any validiation expression can be combined with ++ :

(df int-stuff
    (:i (< 1))
    [(++ :i (or> #(< 2 % 10) #(> -2 % -10))) [x y 5]]
    ... )

;; x and y must also be integers between one of these two ranges

;; ep>, and> and or> mean the same thing if you just use a single
;; function within

;; standard Clojure equivalent of int-stuff:

(defn int-stuff
      [{:keys [x y] :or {x 5 y 5}
        :as int-stuff-input}]
      {:pre [(integer? x) (integer? y)
             (or (< 2 x 10) (> -2 x -10))
             (or (< 2 y 10) (> -2 y -10))]
       :post [(integer? %) (< % 1)]}
      ... )
```

Attempting to call functions with improper arguments yields useful exceptions indicating the exact argument name and the validation it failed.

Additionally, your own custom :pre/:post map can be used in the function normally, if desired.

#### Simple enums

Since sets are functions, this simple technique can be re-used everywhere you need a value to be only one of a particular set of options:

```clojure
(def sexes #{:male :female})

(df person [:str name (sexes) sex] ...)

;; The function will only accept a value for "sex" that is :male or :female
```

### Custom types, traits, and the pred form

We can think of a custom "type" as restrictions on the contents of an entire map, including the types of its elements, and a custom "trait" as specific requirements for part of a map's contents. But technically, this is merely a conceptual difference as they are tested the same way and refer here to nothing more than easy analysis and categorization of a map's composite data.

Terminology like type, class, trait, and object tends to vary in the industry, and here we do not refer to them in an OO context; we do not discuss mutable state that is married to functions that act on it (though you could certainly store a function in a map -- that's up to you). We use these terms loosely and sometimes interchangeably below.

Complex predicates built as separate functions are an easy way to declare custom "types" and "traits" in a functional way that you can re-use in your argument lists anywhere. 

#### pred

You could write a normal function that returns a truthy value to use as a predicate, but since the ```df``` forms build these tests automatically, one very quick way to build a trait predicate would be:

```clojure
;; Create a simple fn that returns true if its args pass:

(df is-senior? [:string name (> 65) age]
  true)

;; Use it as your trait check:

(df process-seniors
  [(is-senior?) person1 person2]
  ...)

;; Error if you call this fn with maps that don't exhibit the required trait
```
**However**, since the purpose of the df forms is to build assertive functions, you can't use is-senior? everywhere you might want a true or false return value, since it throws exceptions instead of false (...and what if you have assertions turned off in your environment?). Additionally, there is the inelegant extraneous presence of the dangling "true". Fixing both of these problems is the **pred** form, which creates a non-assertive function using the same syntax as df. You do not pass a function body or a :pre/:post map to pred:

``` clojure

;; Much better way:

(pred is-senior? [:string name (> 65) age])

;; The fn is-senior? always returns truthiness or false, and performs no asserts.

;; You can still use this as a trait check, but you can also use it in
;; other places where a false return value is a valid possibility.

;; Predicates created with pred are not allowed to have a function body,
;; but they do accept doc strings and custom names for the input map.
```
Finally, for completeness, you can create an anonymous predicate function with ```predfn```, which builds a fn rather than a defn:
```clojure
((predfn [a b]) {:a 1})  ;this ad-hoc test returns false for the map {:a 1}

;; predfn accepts the arg list only -- no fn name, no doc string, no body.
;; You also cannot access the full input map with predfn.

;; You could inline it as a quick custom check 
;; for a function argument that is a map:

(df foo [((predfn [a b])) mymap] ...)

;; When using predfn as an ad-hoc test inside a validation expression like this,
;; you can simplify with pred> which removes a layer of parens:

(df foo [(pred> [a b]) mymap] ...)

;; these two expressions are the same, and return false:

((predfn [a b]) {:a 1})

(pred> {:a 1} [a b])

;; predfn and pred> introduce a new layer of the special arg vector, 
;; and you can nest these as deep as you need for complex object
;; hierarchies, if necessary

;; But if you merely want to see if a single key exists in a map supplied as
;; an argument, this is easier:

(df foo [(:a) mymap])

;; expands to a check of (:a mymap)

;; pred and predfn are not assertive, but this foo call is assertive
;; as all df forms enforce their arguments to the specified criteria.
;; foo fails if mymap does not have :a and :b keys

;; the *> helpers are laid out to simplify validation expressions, 
;; and several are introduced below.
```
In this way, the pred/predfn tools may be useful even if you have no desire to introduce assertions in your work. Use them to quickly build predicates for testing the contents of a map anywhere in your code.

#### Testing homogenous collections

Sometimes you need to ensure that all items in a sequence, such as a vector, exhibit the same traits.

```clojure
;; If you want to pass in a vector of persons and make sure it is a homogenous
;; collection that meets a trait requirement, you could write:

(df process-seniors
   [(#(every? is-senior? %)) persons]
   ...)

;; That's not too bad, but Eat Static provides another validation helper, epcoll>, 
;; which combines the Clojure core fns every-pred and coll? and also reverses
;; the order of arguments to make it more idiomatic for these checks.
;; It implicitly requires the argument to pass the coll? test *first* so you can
;; still get a false value rather than an exception in cases where you need that:

(pred is-awesome? [(= :super-cool) cool-factor])

(df process-awesome-seniors
    [(epcoll> is-senior? is-awesome?) persons]
    ...)

;; will fail unless persons is a collection and 
;; each item is is-senior? *and* is-awesome?

;; Or you could test that each map in a vector has the keys a and b:

(df all-have-a-b
    [(epcoll> (predfn [a b])) maps maps2]
    ...)

(c all-have-a-b :maps [{:a 1 :b 2} {:a 5 :b 6 :c 7}]
                :maps2 [{:a "hi" :b 11.1} {:b 6 :a :yo}])

;; these arguments pass validation

;; this tests that the vector passed in contains only integers:

(df intvec [(epcoll> integer?) v]
    ... )

;; enforces that is is indeed a vector also:

(df intvec [(++ :v (epcoll> integer?)) v]
  ... )

;; For cases like this, more specific versions of epcoll> automatically
;; use a more restrictive collection type in the check, so you don't have to
;; explicitly add the :v --

;; epv> will test that it is a vector before running the args on each item
;; epl> requires a list
;; eps> tests for a set
;; epm> tests for a map, though the other map validation functions described
;; in this document are likely to be better choices for a map collection

;; Note that keyword type checks like :int and thread-first-style 
;; validation expressions are only available in the top level of
;; the arg list, and not inside an expressions like epcoll>, and>, or> and ep>. 
;; Thus use integer? rather than :i or :int and use #(< % 1), not (< 1) --
;; anything you could pass normally to Clojure's every-pred function.

;; Alternately, use the simple "t" function (for "type") inside epcoll>, 
;; which will retrieve the actual function predicate associated with
;; keyword:

;; same as above:

(df intvec [(epv> (t :i)) v]
  ... )

;; enforces that the collection passed in contains only integers
;; or keywords:

(df vari [(or> #(epcoll> % (t :i))       
               #(epcoll> % (t :k)))
               v] 
  ... )

```
##### Simple functional types
If you want to create and re-use a particular "type" such as the vector of integers demonstrated above, just lift it out into its own function:
```clojure
(defn vec-of-ints [x] (epv> x (t :i))) 
```
Now you can just use this as the specifier for a type of function argument:
```clojure
(df many-vecs [(vec-of-ints) a b c] ...)

;; many-vecs is a function whose parameters of a,b,c
;; must all be vectors of integers.
```
Here is a way to describe a kitty cat, and a bunch of them, and then a function to feed them, making sure you don't accidentally try to feed a dog, and ensuring your pet shop doesn't have a monster instead of a kitty lurking somewhere:
```clojure
(pred kitty [:k color :i age :b likes-milk])

(defn bunch-o-kitties [ks] (epv> ks kitty))

(df feed-kitties [(bunch-o-kitties) cats] ... )
```
#### Custom aggregate types

In addition to the simple techniques above for defining a custom "type", Eat Static provides a couple of useful tools for rounding out the picture of a specification for an aggregate map.

Map constructors and validators using ```describe```:

```clojure
;; You could make a robust "person" object using a constructor built
;; with df that simply returns the map if the parameters passed all checks:

(df make-person
    [:bool eats-meat :str name [spouse] country :i age :k education]
    make-person-input)

(make-person {:name "Ludwig" :spouse "Angela" :education :college
              :country "USA" :age 39 :eats-meat false})

(make-person {:name "Bobby" :country "USA" :age 4
              :eats-meat true :education :pre-school})

;; Bobby doesn't have a spouse (yet)

;; Your map will fail if it doesn't meet the types and parameters
;; required by make-person

;; Worth noting that this same function can also be used as an assertive
;; predicate on maps already constructed, to verify they are the 
;; "make-person" type. But, typically, for true/false verification, it is better
;; to just build a pred:

;; Verifies a map is a "person" object:

(pred person? [:bool eats-meat :str name [spouse] country :i age :k education])

;; Note that the df form and the pred form simply have the exact same
;; vector-formatted arg list. 
;; You can accomplish both at once using "describe":

;; Two birds, one stone:

(describe child [:str name fav-toy :i age :k education])

;; This one-liner generates an assertive constructer called make-child
;; and a true/false non-assertive validator called child? 

(def alex
   (make-child {:name "alex" :fav-toy "legos"
                :age 8 :education :primary}))

(child? alex) ;; returns a truthy value

;; Function bodies, custom input map names and docstrings
;; are not passed to describe.

;; In both functions, the input map is also named child-input should you
;; wish to use it in the argument vector to (describe ...), i.e. for 
;; creating new descriptions based on existing ones:

(describe baby-child [(child?) baby-child-input (< 2) age])

;; Additionally, the make-child function automatically adds default values
;; to your map, if they were supplied in the original arg list:

(describe defaults [:i a [b 8 c 9] -z])

(make-defaults {:a 1})

;; returns {:a 1 :b 8 :c 9}

(make-defaults {:a 1 :b 2})

;; returns {:a 1 :b 2 :c 9}

;; You can optionally provide the prefix and suffix used for the make function
;; and predicate, respectively; either you provide both, or neither:

(describe baby-child [(child?) baby-child-input (< 2) age] new- "")

;; This generates new-baby-child as the make function, and baby-child as the
;; predicate function.

;; A nearly identical form of "describe" is "desc". It is shorter to type
;; and does not accept optional names for the two functions it creates, using the
;; standard make- and ..? function names.
;; Using "desc" forces consistency on the naming conventions, which is important
;; if you use "blend" described further below.

;; Additionally, describe- and desc- create private versions 
;; of the constructor and validator
```
##### A note on function names

```describe```, ```desc``` and ```blend``` (explained below) generate 2 functions, a "factory" with ```df``` that validates a map before usage, using assertions, and a predicate that provides true/false verification that a map meets the described criteria. These are separate bonafide functions that have their own def'd name. As shown above, ```describe``` gives you the option of controlling how they are named, while ```desc``` and ```blend``` do not. Several of the features of Eat Static rely on a consistent naming scheme behind the scenes, and you can override the ```make-``` and ```...?``` prefix and suffix used throughout the library if you wish, which will change how function names are defined, and update other tools that need to peek at these generated functions:

```set-describe-names! [prefix suffix]```

This will accept two strings used in place of the ```make-``` prefix and the ```?``` suffix used in factory and predicate functions.

```slim-describe-names! []```

This sets the make prefix to ```+``` and leaves an empty predicate suffix, such that a type named ```child``` is created with ```(+child ...)``` and checked as ```(child ...)```.

```default-describe-names! []```

This sets the prefix and suffix back to the defaults of ```make-``` and ```?```.

Additionally, instead of calling the def'd functions directly, you can use the ```make``` and ```is?``` tools to do the same tasks you'd normally do by calling the functions directly, regardless of how those functions' prefix and suffix are named:

```clojure
(set-describe-names! "front" "back")

(desc fb [:k q w e [r :whatever]])

(make fb {:q :ui :w :w :e :e})
;; allows you to forget the naming scheme that is set

(is? fb *1)

(d fb)
;; returns {:r :whatever}
;; getting defaults is described further below

;; these two functions return unresolved symbol errors, 
;; since the naming scheme has changed:

make-fb
fb?

;; however these two now exist instead:

frontfb
fbback
```
Or, you can simply never change the describe names, never provide naming options to ```describe``` or just always use ```desc``` and ```blend```, and then none of these tools are necessary. But the flexibility exists to control exactly how functions are ultimately named in your namespace.

#### A few more trait examples

Based on what has been shown thus far, here are a few more examples of quickly generating custom types and traits that are easy to build and validate using ```desc```:
```clojure
(desc person [:str name :i age :k sex :n height])
(desc tall [(> 2) height])
(desc tall-person [(ep> person? tall?) tall-person-input])
(desc short-person [(person?) short-person-input (< 1) height])
(desc tall-person-bobby [(tall-person?) tall-person-bobby-input (= "bobby") name])
(desc child [(person?) child-input (< 27) age])
(desc short-child [(child?) short-child-input (< 0.8) height])

;; fails:

(make-short-child {:name "andrew" :age 12 :height 1.5 :sex :m})
```
Predicates are great for building trait validation, where a data structure must meet different criteria at different times, depending on the traits of interest:

```clojure

(df invite-to-amsterdam-elder-poker-party
    [(is-senior?) person (is-dutch?) person] ...)

;; If instead of explicitly trusting these args and operating on them, 
;; you simply want to verify a combination of traits:

(pred is-dutch-senior? [(is-senior?) person (is-dutch?) person])

;; call it:

(c is-dutch-senior? :person hank)

;; You get truth or false: Either Hank is a senior citizen who lives in Holland
;; or isn't. 
```
##### Using the full input-map 

In the above example, it is unnecessarily verbose to pass hank with a key of ```:person``` rather than just letting the guy pass on his own -- he is a map, after all, and our functions all accept maps.     

When composing predicates, accessing the full single input map in the arg list (as shown further above) can be quite helpful to avoid unnecessary nesting:

```clojure
(describe person [:str name country :i age :k sex :n height :b eats-meat])

(pred american-meat-eating-child? [(< 10) age (= true) eats-meat (= "USA") country])

(american-meat-eating-child? {:age 50 :eats-meat true :country "USA"}) ;; false

(def jimmy (make-person {:age 5 :eats-meat true :country "USA"
                         :height 1 :sex :m :name "jimmy"}))

(american-meat-eating-child? jimmy) ;; truth

;; To make sure a young american cat doesn't pass the test:

(pred american-child-likes-meat? 
   [(person?) kid (american-meat-eating-child?) kid])

(c american-child-likes-meat? :kid jimmy)

;; Note the difference in the way these two are called:

(american-meat-eating-child? jimmy)

(american-child-likes-meat? {:kid jimmy})

;; This is because the second function used two predicates
;; on a single map passed as an arg, while the first tested 
;; the contents of the map directly. Accessing the input map
;; directly in the arg list helps us here:

(pred american-child-likes-meat? kid ;; added kid to name input map
   [(person?) kid (american-meat-eating-child?) kid])

;; All we did is make "kid" the name of the full input map instead.
;; Now you can pass the map directly.
```
And a few more examples:
```clojure
(pred is-dutch? [(= "netherlands") country])

(pred is-senior? [(> 65) age])

(pred is-vegetarian? [(= false) eats-meat])

(pred elder-dutch-vegetarian? person
      [(person?) person
       (is-senior?) person
       (is-vegetarian?) person
       (is-dutch?) person])

;; With Clojure's every-pred, you can compose the predicates and thus
;; avoid repeating parameter names:

(pred elder-dutch-vegetarian? person
      [((every-pred person? is-senior? is-vegetarian? is-dutch?)) person])

;; ep> simplifies this:

(pred elder-dutch-vegetarian? person
      [(ep> person? is-senior? is-vegetarian? is-dutch?) person])

;; This is just like epcoll> except it doesn't take a collection.

;; If all your checks are on the same input map and none of its
;; individal elements, the above are more easily composed as follows:

(def old-dutch-veggie-eater? (every-pred person? is-senior? is-vegetarian? is-dutch?))

(def veggie-eater? (every-pred person? is-vegetarian?))

(def young-veggie-eater? (every-pred person? (complement is-senior?) is-vegetarian?))
;; or
(def young-veggie-eater? (every-pred person? (predfn [(< 20) age]) is-vegetarian?))
;; or
(pred young-veggie-eater? m [(ep> person? is-vegetarian?) m (< 20) age])

;; (pred ...) is typically easier when you combine
;; full-map validation with individual element checks
```
#### Blend

Now that you have seen ```describe```, ```desc``` and general predicate composition, the final tool for building new type/traits out of existing ones is ```blend``` which is a special version of ```describe```. 

A ```make-...``` function built with ```describe``` or ```desc``` automatically adds any default values specified in the definition, as noted previously. And while you can manually build a combined predicate expression that ensures your new type meets several criteria, it doesn't provide you automatic default values for all the composite types.

```blend``` combines the predicate composition with an overall combined defaults listing behind the scenes, all you must do is specify symbols previously defined with ```describe``` or ```desc```:

```clojure
;; a and b have default values, and q is required:
(describe ab [:i q [a 1 b 2]])
;; all must be integers

;; c and d have default values and w is required
(desc cd [:f w -r :any [c 3 d 4]])
;; w must be a float, while c and d can by anything

;; desc and describe are synonymous in the above definitions

;; blend can use any functions defined using the standard "make-..." and "...?"
;; naming convention outlined above

;; e is a required keyword, and ab-cd is both a ab and a cd type
(blend ab-cd [:k e] ab cd)
```
The ```ab-cd``` type (or trait) has all the properties and requirements of the ```ab``` and ```cd``` types, and if you build an ```ab-cd``` type, you will automatically get all of their default values, as well as enforcement on the others, as if you had included ```(ep> ab? cd?)```:
```clojure
;;these will fail because required values of the composite types are omitted:

(make-ab-cd {:e :hi})
(make-ab-cd {:e :hi :w 1.1})

;; but this passes:

(make-ab-cd {:e :hi :w 1.1 :q 55})

;; returned:
;; {:e :hi, :w 1.1, :q 55, :a 1, :c 3, :b 2, :d 4}

```
If you are blending "types" that all have the same named parameter as a default, the order you pass to ```blend``` matters, and precedence is given to those at the end of the list, as if you were defining an inheritance tree:
```clojure
(desc red-rectangle [:n [width 5 height 3] :k [color :red]])

(desc square [:n [width 5 height 5]])

;; makes a red square from a red rectangle
(blend red-square [] red-rectangle square)

(make-red-square {})

;;returns {:width 5, :height 5, :color :red}

;; however, if the order to blend is reversed:
(blend red-square [] square red-rectangle)

(make-red-square {})

;; returns {:color :red, :width 5, :height 3}
;; not a square! because the final blended item was a rectangle,
;; which gets priority.

;; of course, you can make a blue square too:
(blend blue-square [[color :blue]] red-rectangle square)

(make-blue-square {})

;; returns {:width 5, :height 5, :color :blue}
```
If you stored functions as defaults, then this would let you decide which of the blended items gets the proper implementation for the same named function, much like inheritance in standard OO.

##### Quickly generating default maps

If all you want is a map of all the default values for a described or blended type, you can use ```d``` (for "defaults") and ```dv``` to get a vector of those defaults:

```clojure
(desc baby-white-kitty [:k [color :white] :i [age 0] :b [likes-milk true]])

(d baby-white-kitty)

;; returns {:color :white, :age 0, :likes-milk true}

(def feline-litter (dv baby-white-kitty 5))

;; contains 5 baby white kitties

[{:color :white, :age 0, :likes-milk true}
 {:color :white, :age 0, :likes-milk true}
 {:color :white, :age 0, :likes-milk true}
 {:color :white, :age 0, :likes-milk true}
 {:color :white, :age 0, :likes-milk true}]
 ```
This is particularly useful if your type descriptions contain only default values for all parameters; then you can quickly generate baseline objects that will pass all predicates, if that is appropriate.

If you want to store "objects" as map parameters, this is also useful:
```clojure
(desc car [:i [age 0] :str [make "GM"]])

(desc new-car-purchase [:str store (car?) [new-car (d car)]])

(c make-new-car-purchase :store "Al's Car Shop")

;; returns {:store "Al's Car Shop", :new-car {:age 0, :make "GM"}}

(desc factory-output [(epv> car?) [cars (dv car 5)]])

(make-factory-output {})

;; returns: 

{:cars
 [{:age 0, :make "GM"}
  {:age 0, :make "GM"}
  {:age 0, :make "GM"}
  {:age 0, :make "GM"}
  {:age 0, :make "GM"}]}
```
If you want to tweak the defaults, just use the ordinary ```make-``` version as shown above with any additional or overridden parameters, or use ```vmake``` to make a vector of makes with the same tweaks:
```clojure
(def new-white-cars (vmake car {:color :white} 5))

;;returns:

[{:color :white, :age 0, :make "GM"}
 {:color :white, :age 0, :make "GM"}
 {:color :white, :age 0, :make "GM"}
 {:color :white, :age 0, :make "GM"}
 {:color :white, :age 0, :make "GM"}]
```
#### Relationships between objects:
```clojure
;; Analyze interactions between maps:

(defn married? 
   [husband wife]
   (and (person? husband) (person? wife)
        (= (:name husband) (:spouse wife))))  

;; If you want to make sure you never call married? with numbers or 
;; pets instead of people, it just becomes:

(df married? 
    [(person?) husband wife]
    (= (:name husband) (:spouse wife)))

;; Calling the df version is different than calling the defn version, of course,
;; since the parameters are now named by the caller, their ordering no longer matters,
;; and passing a person type is required.

;; For completeness' sake, here is the pred version, demonstrating that you can
;; access other arguments in a validator:

(pred married?
      [(person?) husband wife
       (#(= (:name husband) (:spouse %))) wife])    

;; You could technically write entire complex functions inside an
;; argument list, but that's not really the idea. And remember for pred, 
;; it only ever returns a truthy or false value.

;; Using married? and other quick predicates we built above:

(df old-dutch-vegetarian-spouses?
    [(person?) husband wife]
    (and (c married? :husband husband :wife wife)
         (elder-dutch-vegetarian? husband)
         (elder-dutch-vegetarian? wife)))

;; And a slightly more succinct version, for example's sake:

(df old-dutch-vegetarian-spouses? in
    [(person?) husband wife]
    (and (married? in)
     ((predfn [(elder-dutch-vegetarian?) husband wife]) in)))

;; calling it:
         
(old-dutch-vegetarian-spouses? {:husband ludwig :wife allison})

;; or

(c old-dutch-vegetarian-spouses? :wife alice :husband jason)

;; This function returns true or false as long as we pass in person
;; objects. If we lifted the elder-dutch-vegetarian? checks into the
;; arg list as well, it would *require* them to be true, which is not
;; the obvious goal of this function. Alternately, we could use pred,
;; but that would not enforce that you called this predicate with actual
;; person types, so your choice depends on the type of safety you want.
```
##### **c>** 
Just as ```c``` lets you pass named parameters as individual arguments, ```c>``` does the same, but re-arranges the arg order slighly to allow for idiomatic usage inside one of the validation expressions of an argument list:
```clojure
;; Alternative:

(pred old-dutch-vegetarian-spouses?
      [(elder-dutch-vegetarian?) husband wife 
       (c> married? :husband :wife wife) husband])

;; The value of husband is associated with the name :husband

;; This version is non-assertive regardless of inputs. It removes the person?
;; check since elder-dutch-vegetarian? already checks that. 

;; Of course, unordered named parameters let you do it this way also:

(pred old-dutch-vegetarian-spouses?
      [(elder-dutch-vegetarian?) husband wife 
       (c> married? :wife :husband husband) wife])

;; In this particular example, just using the input map is better:

(pred old-dutch-vegetarian-spouses? in
      [(elder-dutch-vegetarian?) husband wife (married?) in])

;; You call these the same as the above df version:

(c old-dutch-vegetarian-spouses? :wife alice :husband jason)

```

#### The family:

##### Expand to defn or defn- or fn:

 * **df** for a defn
 * **df-** for a defn-
 * **dfn** for a fn. unlike fn where name is optional, here it is required
 * **pred** for a bodyless, non-assertive defn predicate
 * **pred-** for a defn- equiv of pred
 * **predfn** for an anonymous predicate fn
 * **describe/desc** to generate both **make-...** and **...?** df and pred forms, respectively
 * **describe-/desc-** creates private defn- versions same as description
 * **blend** is a special ```describe``` that blends default values and automatically creates a composite predicate based on other types defined with ```describe```

##### Validation helpers:

 * **++** combines validation expressions in the arg vector so all must pass

*> helpers are typically used inside a validation expression of an arg list, though could be used stand-alone for convenience. They all share the quality that the item you are validating is their first argument, as with ```->``` macro expressions:

 * **c>** is like c, but designed for use in an argument validation expression
 * **pred>** is simplified predfn syntax for a validation expression
 * **and>** accepts predicate or anonymous functions, all of which much pass on the argument; fundamentally same use as ep>
 * **or>** same as and> but only one of the supplied functions must pass
 * **ep>** builds and calls an every-pred expression; supply one or more functions
 * **epcoll>** tests coll? on argument, and runs ep> on each element
 * **epv>** like epcoll> but uses vector? before running the args on each item
 * **epl>** like epcoll> but uses list?
 * **eps>** like epcoll> but uses set?
 * **epm>** like epcoll> but users map?

##### Other conveniences

 * **c** to call fns without squiggly braces
 * **t** to get a primitive type check associated with a keyword, i.e. :i for an integer? check
 * **d** returns a map of all default values available on a symbol created with ```desc``` or ```blend```
 * **dv** like ```d``` but creates a vector of default maps
 * **vmake** like ```dv``` but accepts additional args to merge onto the default map repeated in the vector
 * **make** and **is?** allow you to access the factory and predicate versions of symbols described by ```desc``` and ```blend``` without knowing the naming scheme

Hopefully, they help you take advantage of Clojure's excellent tools for making your code safer and easier.

### Turning it Off

While the ```pred``` functions are non-assertive, the functions you build with ```df``` and the factory parts of ```describe```, ```desc``` and ```blend``` ("make-"... ) throw run-time assertions if arguments are not provided as expected. That's the idea, but eventually you may wish to disable these assertions in production code.

In Clojure, all asserts can be ignored by setting the global dynamic variable ```*assert*``` to ```false```. You must recompile your code for this to take affect. Any assertive function written with Eat Static, as well as any other asserts you use in your code, including :pre/:post maps, will no longer then throw assertions.

Eat Static offers another library-specific mechanism to remove all the expanded ```assert``` expressions entirely (but not the non-assertive expressions used in functions built with the ```pred``` family). Simply call ```(off)``` and then recompile, and the expanded `defn` forms will not have any asserts. Turn it back on by calling ```(on)```, and recompiling. This is independent of the ```*assert*``` binding.

### Tests and Code Examples

All the code on this page is available in the examples.clj file of the test folder, so you can download and play around with the functions.

Also, if you are familiar with the basic tools offered by clojure.test, including just the ```deftest``` and ```is``` macros, you can see many examples of this library's features in the test folder, including scenarios when assertions are thrown.

### Gotchas

#### A few things to remember and watch out for

Just as you can do things in a Clojure pre/post map that make no sense, you could also do the same with Eat Static. For example, requiring conflicting validations to all pass for a local. Here are a few other additional things to look out for.

##### General tidbits

* Some of the concepts here, especially ```blend```, share some similarities to object-oriented programming. But the predicates created by these functions are not like a type check in other languages. There is a difference between a minimum-acceptable combination of parameters and a full-fledged "object" with all desired parameters. When you specify parameters as optional with default values, the predicate allows them to be excluded entirely. Therefore, it is often preferable to use the ```make-``` version of a blended description on a map first, before using it, rather than assuming that a map that passes a predicate test has all the data you need; or just pass your maps to functions by using the desired make- tool to ensure any defaults are included on maps that are themselves parameters.
* Remember, all the functions created with this library take a single argument, a map, and the parameters are named. The ```c``` tool helps make this idiomatic. The benefits are huge, but it might take a bit to remember this as you start using the library.
* Remember, these are run-time checks, not static compilation checks. We like the dynamic nature of Clojure and have no intention of interfering with that. Reach for this run-time safety when you need it.

##### Default and optional values

* Default value vectors can assign multiple symbols to the same default, which appears at the end of a sequence of symbols. Be mindful that in ```[[a b 9]]```, ```a``` has a default value of ```9```, not ```nil```. If you want to mix optional and default values and make ```a``` have no default, use ```[-a [b 9]]``` or ```[b 9 a]```.
* It is entirely possible to specify a default value for a symbol that would not itself pass another validation specified in the argument vector, just as you could write conflicting ```:or``` and ```:pre``` maps. 
* If you use a local in your function body, but specify that local as optional with no default value, then it is entirely possible that you will get a null pointer exception if you haven't first checked that it could be nil and act appropriately. i.e. ```(df mult [-b] (* b 1))``` does exactly this with ```(mult {})```, since you cannot multiply nil as a number.
* The library will let you try to assign multiple default values to the same symbol in different default value vectors. Only one will be used, of course, and it is not defined which one it will be. You would not likely ever do this except by accident.
* The libary will not let you specify a symbol as both required and optional.
* If you wish to specify all defaults in the same default vector for beauty's sake, but the args have different types, use ```:any``` before the vector to mix default types: ```(describe aa [:i -a :f -b :any [a 1 b 4.4]])``` If ```:any``` is omitted, then you'd be trailing the ```:f``` type validator for these defaults, requiring ```a``` to be both an integer and a float! Don't worry -- one validation like ```:any``` does not cancel another like ```:f```, so the type of ```a``` and ```b``` is still enforced.

##### Validation

* A validation item in the argument vector affects only those symbols up until the next validation item. If you have ```[:i :n a b c]``` the ```:i``` has no purpose. This is good since often these would confict, i.e. ```[:i :f a b]```. If you need to combine them, use the ```++``` expression.
* Validation expressions are not functions. They are list expressions, similar to the thread-first ```->``` macro. Don't use functions like ```neg?``` in isolation, use ```(neg?)``` instead.
* While the vast majority of use cases will provide a meaningful assertion message if a validation fails, occasionally you can write an expression that attempts to do something with nil that instead leads to a null pointer exception instead, which carries no useful message. The assertion is valid based on your specified validation criteria, but the message is not clear.
* Beware of ```false``` and ```nil```. If you expect to pass nil as a legitimate value to your function, do not make that argument required, as it will enforce it as non-nil. If you expect to pass ```false``` to your function, it is wise to specify the argument as a boolean type, as in some cases a ```false``` value may not pass validation if it is a more general type, such as ```:any```.


### Community

Issues, feedback, pull requests all welcome. Many thanks.



Copyright 2015 Balcony Studio, the Netherlands