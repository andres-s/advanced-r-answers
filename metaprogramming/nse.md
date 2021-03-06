Capturing expressions
---------------------

 1. Let `g = function(x) deparse(substitute(x))`. Why does

    ```r
    g(a + b + c + d + e + f + g + h + i + j + k + l + m +
      n + o + p + q + r + s + t + u + v + w + x + y + z)
    ```

    return a character vector of length 2? Write a wrapper around `deparse()` so that it always returns a single string.

    The `deparse` function takes a `width.cutoff` argument, which takes integer values in `[20, 500]`. `deparse` will start creating a new character element if it has reached the end of a symbol and has already printed at least `width.cutoff` bytes into the current character element.

    ```r
    g2 = function(x) paste(deparse(substitute(x)), collapse = " ")
    ```

 2. Why does `as.Date.default()` use `substitute()` and `deparse()`? Why does
    `pairwise.t.test()` use them?

    `as.Date.default()` uses `substitute()` and `deparse()` for printing an error message when the argument can't be converted to `Date`:

    ```r
    stop(gettextf("do not know how to convert '%s' to class %s", 
         deparse(substitute(x)), dQuote("Date")), domain = NA)
    ```

    `pairwise.t.test()` uses `substitute()` and `deparse()` to print out the names of the columns being tested.


 3. `pairwise.t.test()` assumes that `deparse()` always returns a length one
    character vector. Can you construct an input that violates this expectation?

    ```r
    aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa <- 1
    bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb <- 2
    pairwise.t.test(c(aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa, bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb), as.factor('a', 'b'))
    #>
    #>     Pairwise comparisons using t tests with pooled SD 
    #>
    #> data:  c(aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa,  and as.factor(c("a", "b"))     bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb) and as.factor(c("a", "b")) 
    #> 
    #>   a
    #> b -
    #>
    #> P value adjustment method: holm     
    ```

    Here a call to `paste` has created a character vector of length two after "data:".

 4. Explain the output of
    ```r
    f <- function(x) substitute(x)
    g <- function(x) deparse(f(x))
    g(1:10)    
    g(x)
    g(x + y ^ 2 / z + exp(a * sin(b)))
    ```

    This is clearer written slightly differently:

    ```r
    f2 <- function(x) substitute(x)
    g2 <- function(y) deparse(f(y))
    
    g2(1:10)
    #> [1] "y"

    g2(x)
    #> [1] "y"

    g2(x + y ^ 2 / z + exp(a * sin(b)))
    #> [1] "y"
    ```


Non-standard evaluation in subset
---------------------------------

 2. We can define `subset` using `substitute` and `eval`:
    ```r
    subset2 <- function(x, condition) {
      condition_call <- substitute(condition)
      r <- eval(condition_call, x)
      x[r, ]
    } 
    ```
    However, this doesn't return the correct type when dataframe passed into the first argument has only one column:
    ```r
    sample_df2 <- data.frame(x = 1:10)
    subset2(sample_df2, x > 8)
    #> [1]  9 10
    ```
    Fix it.

    ```r
    subset3 <- function(x, condition) {
        condition_call <- substitute(condition)
        r <- eval(condition_call, x)
        x[r, , drop = FALSE]
    }
    sample_df3 <- data.frame(x = 1:10)
    subset3(sample_df3, x > 8)
    #>     x
    #> 9   9
    #> 10 10
    ```

 3. The real subset function (`subset.data.frame()``) removes missing values
    in the condition. Modify your `subset()` to do the same: drop the offending rows.

    ```r
    subset4 <- function(x, condition) {
        condition_call <- substitute(condition)
        r <- eval(condition_call, x)
        x[r & !is.na(r), , drop = FALSE]
    }
    sample_df4 <- data.frame(x = c(1:10, NA))
    subset4(sample_df4, x > 8)
    #>     x
    #> 9   9
    #> 10 10
    ```

 4. What happens if you use `quote()` instead of `substitute()` inside of
    `subset2()`?

    We get an error:

    ```r
    subset5 <- function(y, condition) {
        condition_call <- quote(condition)
        r <- eval(condition_call, y)
        y[r, ]
    }
    sample_df5 <- data.frame(x = c(1:10, NA))
    subset5(sample_df5, x > 8)
    #>  Error in eval(expr, envir, enclos) : object 'x' not found
    ```

    To understand why this occurs, try setting the `enclos` argument of `eval` to be `NULL`:

    ```r
    subset5_2 <- function(y, condition) {
        condition_call <- quote(condition)
        r <- eval(condition_call, y, enclos = NULL)
        y[r, ]
    }
    subset5_2(sample_df5, x > 8)
    #>  Error in eval(expr, envir, enclos) : object 'condition' not found
    ```

    In the call to `subset5`, `quote` is capturing the symbol `condition` (as opposed to `substitute`, which captures the expression slot of the `condition` promise object). The call to `eval`

         1. tries to find `condition` int the `sample_df5` environment and fails;
         2. so it looks in the environment where `eval` is called (see `?eval`: the default argument to `enclos` is `parent.frame`, ie. the parent of the `eval` frame); it finds the promise object tied to the formal argument `condition`, which it tries to evaluate;
         3. it evaluates `x > 8` in the global environment which fails as `x` has not been defined.
    
    By setting `enclos = NULL`, this process stops at step 1 and an error is thrown immediately.

 5. The second argument in `subset()` allows you to select variables. It treats variable names as if they were positions. This allows you to do things like `subset(mtcars, , -cyl)` to drop the cylinder variable, or `subset(mtcars, , disp:drat)` to select all the variables between `disp` and `drat`. How does this work? I've made this easier to understand by extracting it out into its own function.

    ```r
    select <- function(df, vars) {
        vars <- substitute(vars)
        var_pos <- setNames(as.list(seq_along(df)), names(df))
        pos <- eval(vars, var_pos)
        df[, pos, drop = FALSE]
    }
    select(mtcars, -cyl)
    ```

    
