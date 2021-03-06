# Getting Started with Fennel

If you know Lua and a lisp already, you'll feel right at home in
Fennel. Even if not, Lua is one of the simplest programming languages
in existence, so if you've programmed before you should be able to
pick it up without too much trouble, especially if you've used another
dynamic imperative language with closures. The
[Lua reference manual](https://www.lua.org/manual/5.1/) is a fine place to
look for details.

A programming language is made up of **syntax** and **semantics**. The
semantics of Fennel vary only in small ways from Lua (all noted
below). The syntax of Fennel comes from the lisp family of
languages. Lisps have syntax which is very uniform and predictable,
which makes it easier to
[write code that operates on code](http://www.defmacro.org/ramblings/lisp.html)
as well as
[structured editing](http://danmidwood.com/content/2014/11/21/animated-paredit.html).

## OK, so how do you do things?

Variables are set with `set`, and functions are created with `fn`,
using square brackets for argument lists:

```lisp
(set add (fn [x y] (+ x y)))
(add 32 12) ; -> 44
```

Functions defined with `fn` are fast; they have no runtime overhead
compared to Lua. However, they also have no arity checking. For safer
code you can use `lambda`:

```lisp
(set print-calculation
     (lambda [x ?y z] (print (- x (* (or ?y 1) z)))))
(print-calculation 5) ; -> error: Missing argument z
```

Note that the second argument `?y` is allowed to be omitted, but `z` is not.

Local variables are introduced using `let` with the names and values
wrapped in a single set of square brackets:

```lisp
(let [x (+ 89 5.2)
      f (fn [abc] (print (* 2 abc)))]
  (f x))
```

### Numbers and strings

Of course, all our standard arithmetic operators like `+`, `-`, `*`,
and `/` work here in prefix form. Note that numbers are
double-precision floats in all but Lua 5.3, which optionally supports
integers.

```lisp
(let [x (+ 1 99)
      y (- x 12)]
  (/ y 10))
```

Strings are essentially immutable byte arrays. UTF-8 support is
provided from a
[3rd-party library](https://github.com/Stepets/utf8.lua). Strings are
concatenated with `..`:

```lisp
(.. "hello" " world")
```

### Tables

In Lua (and thus in Fennel), tables are the only data structure. The
main syntax for tables uses curly braces with key/value pairs in them:

```lisp
{"key" value 
 "number" 531
 "f" (fn [x] (+ x 2))}
```

Some tables are used to store data that's used sequentially; the keys
in this case are just numbers starting with 1 and going up. Fennel
provides alternate syntax for these tables with square brackets:

```lisp
["abc" "def" "xyz"] ; equivalent to {1 "abc" 2 "def" 3 "xyz"}
```

You can use `.` to get values out of tables:

```lisp
(let [tbl (function-which-returns-a-table)
      key "a certain key"]
  (. tbl key))
```

And `tset` to put them in:

```lisp
(let [tbl {}
      key1 "a long string"
      key2 12]
  (tset tbl key1 "the first value")
  (tset tbl key2 "the second one")
  tbl) ; -> {"a long string" "the first value" 12 "the second one"}
```

The `#` function returns the length of sequential tables and strings:

```lisp
(let [tbl ["abc" "def" "xyz"]]
  (+ (# tbl) 
     (# (. tbl 1)))) ; -> 6
```

### Iteration

Looping over table elements is done with `each` and an iterator like
`pairs` (used for general tables) or `ipairs` (for sequential tables):

```lisp
(each [key value (pairs {:key1 52 :key2 99})]
  (print key value))

(each [index value (ipairs ["abc" "def" "xyz"])]
  (print index value))
```

Note that whether a table is sequential or not is not an inherent
property of the table but depends on which iterator is used with it.
You can call `ipairs` on any table, and it will only iterate
over numeric keys starting with 1 until it hits a `nil`.

You can use any Lua iterator with `each`, but these are by far the most common.

The other iteration construct is `for` which iterates numerically from
the provided start value to the inclusive finish value:

```lisp
(for [i 1 10]
  (print i))
```

You can specify an optional step value; this loop will only print odd
numbers under ten:

```lisp
(for [i 1 10 2]
  (print i))
```

### Conditionals

Finally we have conditionals. The `if` form in Fennel can be used the
same way as in other lisp languages, but it can also be used as `cond`
for multiple conditions compiling into `elseif` branches:

```lisp
(let [x (math.random 64)]
  (if (= 0 (% x 2))
      "even"
      (= 0 (% x 10))
      "multiple of ten"
      :else
      "I dunno, something else"))
```

Being a lisp, Fennel has no statements, so `if` returns a value as an
expression. Lua programmers will be glad to know there is no need to
construct precarious chains of `and`/`or` just to get a value!

The other conditional is `when`, which is used for an arbitrary number
of side-effects, has no else clause, and always returns `nil`:

```lisp
(when (currently-raining?)
  (wear "boots")
  (deploy-umbrella))
```

## Back to tables just for a bit

Strings that don't have spaces in them can use the `:keyword` syntax
instead, which is often used for table keys:

```lisp
{:key value :number 531}
```

If a table has string keys like this, you can pull values out of it easily:

```lisp
(let [tbl {:x 52 :y 91}]
  (+ tbl.x tbl.y)) ; -> 143
```

You can also use this syntax with `set`:

```lisp
(let [tbl {}]
  (set tbl.one 1)
  (set tbl.two 2)
  tbl) ; -> {:one 1 :two 2}
```

Finally, `let` can destructure a sequential table into multiple locals:

```lisp
(let [f (fn [] ["abc" "def" "xyz"])
      [a d x] (f)]
  (print a)
  (print d)
  (print x))
```

Note that unlike many languages, `nil` in Lua actually represents the
absence of a value, and thus tables cannot contain `nil`. It is an
error to try to use `nil` as a key, and using `nil` as a value removes
whatever entry was at that key before.

## Error handling

Error handling in Lua has two forms. Functions in Lua can return any
number of values, and most functions will indicate failure by using
multiple return values: `nil` followed by a failure message
string. You can interact with this style of function in Fennel by
destructuring with parens instead of square brackets:

```lisp
(let [(f msg) (io.open "file" "rb")]
  ;; when io.open succeeds, f will be a file, but if it fails f will be
  ;; nil and msg will be the failure string
  (if f
      (do (use-file-contents (f.read f "*all"))
          (f.close f))
      (print "Could not open file: " .. msg)))
```

You can write your own function which returns multiple values with `values`:

```lisp
(set use-file
     (fn [filename]
       (if (valid-file-name? filename)
           (open-file filename)
           (values nil (.. "Invalid filename: " filename))))
```

If you detect a serious error that needs to be signaled beyond just
the calling function, you can use `error` for that. This will
terminate the whole process unless it's within a protected call,
similar to the way throwing an exception works in many languages. You
can make a protected call with `pcall`:

```lisp
(let [(ok val-or-msg) (pcall potentially-disastrous-call filename)]
  (if ok
      (print "Got value" val-or-msg)
      (print "Could not get value:" val-or-msg)))
```

The `pcall` invocation there is equivalent to running
`(potentially-disastrous-call filename)` in protected mode. It takes
an arbitrary number of arguments which are passed on to the
function. You can see that `pcall` returns a boolean (`ok` here) to
let you know if the call succeeded or not, and a second value
(`val-or-msg`) which is the actual value if it succeeded or an error
message if it didn't.

The `assert` function takes a value and an error message; it calls
`error` if the value is `nil` and returns it otherwise. This can be
used to turn failures into errors:

```lisp
(let [f (assert (io.open filename))
      contents (f.read f "*all")]
  (f.close f)
  contents)
```

In this example because `io.open` returns `nil` and an error message
upon failure, a failure will trigger an `error` and halt execution.

## Gotchas

There are a few surprises that might bite seasoned lispers. Most of
these result necessarily from Fennel's insistence upon imposing zero
runtime overhead over Lua.

* The arithmetic and comparison operators are not first-class functions.

* There is no `apply` function; use `unpack` instead: `(f 1 3 (unpack [4 9])`.

* Tables are compared for identity, not based on the value of their
  contents, as per [Baker](http://home.pipeline.com/%7Ehbaker1/ObjectIdentity.html).

* Printing a table will not show you the values inside the table. It's
  recommended to use a 3rd-party library like
  [serpent](https://github.com/pkulchenko/serpent) or
  [inspect.lua](https://github.com/kikito/inspect.lua) to address
  this, but it is technically optional.

* Lua's standard library is quite small, and so common functions like
  `map`, `reduce`, and `filter` are absent. It's recommended to pull
  in something like [Lume](https://github.com/rxi/lume) for those. The
  `lume.serialize` function can be used as a pretty-printer in a
  pinch, but `serpent` and `inspect.lua` produce output that is more
  readable.

* Line numbers represent the lines in the compiled Lua output, not in
  the Fennel source. Luckily the compiled output is usually fairly
  readable, but this is the biggest drawback of Fennel by far.

## Other stuff just works

Note that built-in functions in
[Lua's standard library](https://www.lua.org/manual/5.1/manual.html#5)
like `math.random` above can be called with no fuss and no overhead.

This includes features like coroutines, which are usually implemented
using special syntax in other languages. Coroutines
[let you express non-blocking operations without callbacks](http://leafo.net/posts/itchio-and-coroutines.html).

You can use the `require` function to load code from Lua files.

```lisp
(let [lume (require "lume")
      tbl [52 99 412 654]
      plus (fn [x y] (+ x y))]
  (lume.map tbl (partial plus 2))) ; -> [54 101 414 656]
```

Modules in Lua are simply tables which contain functions and other values.

Tables in Lua may seem a bit limited, but
[metatables](http://nova-fusion.com/2011/06/30/lua-metatables-tutorial/)
allow a great deal more flexibility. All the features of metatables
are accessible from Fennel code just the same as they would be from Lua.

## Embedding

Lua is most commonly used to embed inside other applications, and
Fennel is no different. The simplest thing to do is include Fennel
compilation as part of your overall application's build process.
However, the Fennel compiler is very small, and including it into your
codebase means that you can embed a Fennel repl inside your
application or support reloading from disk, allowing a much more
pleasant interactive development cycle.

Here is an example of embedding the Fennel compiler inside a
[LÖVE](https://love2d.org) game written in Lua to allow live reloads:

```lua
local fennel = require("fennel")
local f = io.open("mycode.fnl", "rb")
;; mycode.fnl ends in a line like this:
;; {:draw (fn [] ...) :update (fn [dt] ...)}
local mycode = fennel.eval(f:read("*all"), {filename = "mycode.fnl"})
f:close()

love.update = function(dt)
  mycode.update(dt)
  -- other updates
end

love.draw = function()
  mycode.draw()
  -- other drawing
end

love.keypressed = function(key)
  if(key == "f5") then -- support reloading
    local f = io.open("mycode.fnl", "rb")
    local mycode_new = fennel.eval(f:read("*all"), {filename = "mycode.fnl"})
    f:close()
    for k,v in pairs(mycode_new) do
      mycode[k] = v
    end
  else
    -- other key handling
  end
end

```
