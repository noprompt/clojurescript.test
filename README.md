# clojurescript.test [![Travis CI status](https://secure.travis-ci.org/cemerick/clojurescript.test.png)](http://travis-ci.org/#!/cemerick/clojurescript.test/builds)

A maximal port of `clojure.test` to ClojureScript.

## Why?

I want to be able to write portable tests to go along with my portable
Clojure[Script], and `clojure.test`'s model is Good Enough™ (it's better than
that, actually).  Combine with something like
[cljx](https://github.com/lynaghk/cljx) to make your ClojureScripting a whole
lot more pleasant.

## Installation

clojurescript.test is available in Maven Central. Add it to your **`:plugins`**
in your Leiningen `project.clj`:

```clojure
[com.cemerick/clojurescript.test "0.2.2"]
```

(clojurescript.test is actually a project dependency _and_ a Leiningen plugin;
adding it as the latter just helps simplify test configuration, as you see below.)

Or, add this to your Maven project's `pom.xml`:

```xml
<dependency>
  <groupId>com.cemerick</groupId>
  <artifactId>clojurescript.test</artifactId>
  <version>0.2.2</version>
</dependency>
```

## Usage

clojurescript.test provides roughly the same API as `clojure.test`, thus making
writing portable tests possible.

(Note that `clojurescript.test` doesn't take any responsibility for any hosty or
otherwise-unportable things you do in your tests, e.g. `js/...` or naming JVM
types or Clojure- or ClojureScript-only functions; either don't do that, or use
something like cljx to include both Clojure and ClojureScript code in the same
file.)

Here's a simple ClojureScript namespace that uses clojurescript.test:

```clojure
(ns cemerick.cljs.test.example
  (:require-macros [cemerick.cljs.test
                    :refer (is deftest with-test run-tests testing test-var)])
  (:require [cemerick.cljs.test :as t]))

(deftest somewhat-less-wat
  (is (= "{}[]" (+ {} []))))

(deftest javascript-allows-div0
  (is (= js/Infinity (/ 1 0) (/ (int 1) (int 0)))))

(with-test
  (defn pennies->dollar-string
    [pennies]
    {:pre [(integer? pennies)]}
    (str "$" (int (/ pennies 100)) "." (mod pennies 100)))
  (testing "assertions are nice"
    (is (thrown-with-msg? js/Error #"integer?" (pennies->dollar-string 564.2)))))
```

**Note**: each test namespace in your project must `(:require
cemerick.cljs.test)` even if you only use macros. Otherwise, the ClojureScript
compilation process won't include clojurescript.test in its output, resulting
in an error similar to "`ReferenceError: Can't find variable: cemerick`".

You can load this into a ClojureScript REPL, and run its tests using familiar functions:

```clojure
=> (t/test-ns 'cemerick.cljs.test.example)

Testing cemerick.cljs.test.example
{:fail 0, :pass 3, :test 3, :error 0}

=> (test-var #'cemerick.cljs.test.example/somewhat-less-wat)
{:fail 0, :pass 1, :test 1, :error 0}
```

All of the test-definition macros (`deftest` and `with-test`, as well as the
`set-test` utility) add to a global registry of available tests (necessary given
ClojureScript's lack of namespaces), so you can also define, redefine, and run
tests interactively:

```clojure
=> (deftest dumb-test
     (is (empty? (filter even? (range 20)))))
#<[object Object]>
nil
=> (t/test-ns 'cemerick.cljs.test.example)

Testing cemerick.cljs.test.example

FAIL in (dumb-test) (:0)
expected: (empty? (filter even? (range 20)))
  actual: (not (empty? (0 2 4 6 8 10 12 14 16 18)))
{:fail 1, :pass 3, :test 4, :error 0}
```

### Writing portable tests

Because clojurescript.test has (approximately) the same API as `clojure.test`,
writing portable tests with it is easy.  For example, the test namespace above
can be made portable using cljx like so:

```clojure
(ns cemerick.cljs.test.example
  #+clj (:require [clojure.test :as t
                   :refer (is deftest with-test run-tests testing)])
  #+cljs (:require-macros [cemerick.cljs.test
                           :refer (is deftest with-test run-tests testing test-var)])
  #+cljs (:require [cemerick.cljs.test :as t]))

#+cljs
(deftest somewhat-less-wat
  (is (= "{}[]" (+ {} []))))

#+cljs
(deftest javascript-allows-div0
  (is (= js/Infinity (/ 1 0) (/ (int 1) (int 0)))))

(with-test
  (defn pennies->dollar-string
    [pennies]
    {:pre [(integer? pennies)]}
    (str "$" (int (/ pennies 100)) "." (mod pennies 100)))
  (testing "assertions are nice"
    (is (thrown-with-msg? #+cljs js/Error #+clj Error #"integer?"
          (pennies->dollar-string 564.2)))))
```

Note that `test-var` is a macro in clojurescript.test; this allows you to
portably write code like `(test-var #'name-of-test)`, even though ClojureScript
doesn't support `#'` or the `(var ...)` special form.  `test-var` forms
macroexpand to calls to `cemerick.cljs.test/test-function`, which is the
corollary to clojure.test's `test-var`.

### Using with lein-cljsbuild

Most people use [lein-cljsbuild](https://github.com/emezeske/lein-cljsbuild) to
automate their ClojureScript builds.  It also provides a test runner, originally
intended for use with e.g. [phantomjs](http://phantomjs.org/) (though there are
[rumors](https://github.com/cemerick/clojurescript.test/issues/10) that it works
nicely with `slimerjs` as well) to run tests that use existing JavaScript test
frameworks.  However, you can easily use the same facility to run
clojurescript.test tests.

This is an excerpt of the lein-cljsbuild configuration that this project uses to
run its own clojurescript.test tests (look in the `project.clj` file for the full
monty):

```clojure
:plugins [[lein-cljsbuild "1.0.0"]
          [com.cemerick/clojurescript.test "0.2.2"]]
:cljsbuild {:builds [{:source-paths ["src" "test"]
                      :compiler {:output-to "target/cljs/testable.js"
                                 :optimizations :whitespace
                                 :pretty-print true}}]
            :test-commands {"unit-tests" ["phantomjs" :runner
                                          "this.literal_js_was_evaluated=true"
                                          "target/cljs/testable.js"
                                          "test/cemerick/cljs/test/extra_test_command_file.js"]}}
```

Everything here is fairly basic, except for the `:test-commands` entries, which
describes the shell command that will be executed when lein-cljsbuild's test
phase is invoked (either via `lein cljsbuild test`, or just `lein test` because
its hook is registered).  In this case, it's going to run `phantomjs`, passing
as arguments:

1. The path to the clojurescript.test test runner script (denoted by
`:runner`, which I'll explain momentarily…), and
2. Either paths to ClojureScript compiler output (a lein-cljsbuild `:output-to`
value defined elsewhere in the `project.clj`), _or_ paths to other arbitrary
JavaScript files (useful for injecting external libraries, polyfills, etc), _or_
arbitrary JavaScript expressions (useful for e.g. configuring runtime test
properties...see the subsection below on using this capability, especially in
conjunction with advanced compilation).


##### Node.js
To run your tests with [node.js](http://nodejs.org/) instead of phantomjs, just change the
executable name and the `:runner` keyword in your `:test-commands` vectors like so:

```
:test-commands {"unit-tests" ["node" :node-runner
                              ; extra code/files here...
                             ]}
```

**Note that you must compile your ClojureScript code with
`:optimizations :advanced` to run it on node.js.**

##### Rhino
To run your tests with [rhino](https://developer.mozilla.org/en/docs/Rhino), change the executable name and the `:runner` keyword in your `:test-commands` vectors like so:
```
:test-commands {"unit-tests" ["rhino" "-opt" "-1" :rhino-runner
                              ; extra code/files here...
                             ]}
```
Note that rhino doesn't support any HTML or DOM related functions and objects so it can be used mainly for business-only logic or you have to mock all DOM functions by yourself.

clojurescript.test bundles test runner scripts for various environments
(currently, phantomjs, node.js and rhino).  As long as you add clojurescript.test to
your `project.clj` as a `:plugin`, then it will replace any occurrences of
`:runner`, `:node-runner` and `:rhino-runner` in your `:test-commands` vectors with the path to
the corresponding test runner script.

**Wanted: runners for other JavaScript environments, e.g. XUL, the
headed browsers of all sorts, etc**

All test runner scripts load the output of the ClojureScript compilation, run
all of the tests found therein, reports on them, and fails the build if
necessary.

clojurescript.test supports all of Google Closure's compilation modes, including
`:advanced`.

#### Configuring tests via JavaScript files/expressions in `:test-commands`

As noted above, you can have arbitrary JavaScript files and/or expressions
loaded before or after your compiled ClojureScript.  One of the most useful
aspects of this is that you can configure properties of your tests; for example,
when using [double-check](https://github.com/cemerick/double-check), you can
control the number of iterations checked by each `defspec` test by setting a
Java system property.  While JavaScript doesn't have a corollary of system
properties, you can add a JavaScript expression to your `:test-commands`
vector(s) that sets a property on some globally-accessible object, e.g.:

```clojure
:test-commands {"rigorous" ["phantomjs" :runner
                            "this.defspec_iters=10000000"
                            "target/cljs/testable.js"]}
```

Then, in your ClojureScript test file(s), you can look up this dynamically-set
value, using a default if it's not set:

```clojure
(def iteration-count (or (this-as this (aget this "defspec_iters")) 1000))
```

The use of `aget` and a string property lookup is necessary to
ensure that the property name will not be renamed/obfuscated by Google Closure
when run with `:advanced` optimizations.  Prior examples of this practice
touched `window`, but that name is undefined in node.js; using `this` when
setting and looking up the test configuration value makes it so that the same
code (and configuration) can be used in any test environment.  


## Limitations

* Bug: filenames and line numbers are not currently reported properly.

## Differences from `clojure.test`

* docstrings bear little to no semblence to the library's actual operation
* Namespace test hooks must be defined using the `deftesthook` macro

### Runtime

* `*report-counters*` is now bound to an atom, not a ref
* `*testing-vars*` now holds symbols naming the top-levels under test, not vars
* `*test-out*` is replaced by `*test-print-fn*`, which defaults to `nil`, and
  is only bound to `cljs.core/*print-fn*` if it is bound to a non-nil value.
* `run-tests` is now a macro; `run-tests*` does the same, but does not offer a
  no-arg arity
* `use-fixtures` is now a macro, and there is no underlying multimethod to
  extend as in `clojure.test`.

### Errors

* Stack traces from caught exceptions are obtained via
  [`Error.stack`](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Error/Stack),
which appears to only be supported in Chrome, FF, Safari, and IE 10+. The value
of `Error.stack` in Rhino (at least, the version specified for use by
ClojureScript) is always an empty string; other JavaScript environments may be
similar.
* File and line numbers of reported exception failures may be missing in
  JavaScript environments that do not support the `lineNumber` or `fileName`
properties of `Error`.

### Removed

* `*load-tests*` is now private, and will probably be removed.  The use case
  for Clojure (which is rarely taken advantage of AFAICT) seems irrelevant for
ClojureScript; if you do or don't want tests in production, you just change
your cljsc/lein-cljsbuild configuration.
* `file-position` was already deprecated and unused
* Not applicable
 * `get-possibly-unbound-var`
 * `function?`
 * `*stack-trace-depth*`

## Need Help?

Send a message to the [ClojureScript](http://groups.google.com/group/clojurescript)
mailing list, or ping `cemerick` on freenode irc or
[twitter](http://twitter.com/cemerick) if you have questions
or would like to contribute patches.

## License

Copyright © 2013 Chas Emerick and other contributors.  Known contributors to `clojure.test` (which was the initial raw ingredient for this project) are:

* Stuart Sierra
* Rich Hickey
* Stuart Halloway
* Phil Hagelberg
* Tassilo Horn
* Mike Hinchey

Distributed under the Eclipse Public License, the same as Clojure.
