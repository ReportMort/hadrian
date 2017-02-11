# Major Syntax Changes
Steven Mortimer  
`r Sys.Date()`  


# Motivation

A number of changes have been introduced in the `aurelius` package to increase 
functionality, usability, and consistency to other R packages. These changes may 
cause existing scripts to fail because they are not backwards compatible with 
prior versions of the package. The purpose of this document is to make more clear 
these changes and how they can be implemented.




# avro.* functions

First, Avro functions were typically prefixed by `avro.*` with the function or 
object name following the period. Periods within function names in R usually 
indicate an S3 method, which is a way of specifying a function with different 
behavior based on the class of object that it's operating on. In this case, these 
functions within `aurelius` were not S3 methods, they were simply prefixed to 
make them more easily identifiable. In an effort to still make them easy to find 
through naming and tab completion, all of these functions now start with `avro_*`. 
In this case they are no longer confused as S3 methods. The behavior is exactly 
the same as prior to this superficial naming change.


```r
library(aurelius)

# avro.int no longer in use
avro_int
#> [1] "int"

# avro.enum no longer in use
avro_enum(list("one", "two"))
#> $type
#> [1] "enum"
#> 
#> $symbols
#> $symbols[[1]]
#> [1] "one"
#> 
#> $symbols[[2]]
#> [1] "two"
#> 
#> 
#> $name
#> [1] "Enum_1"
```

# pfa.* functions

Second, there were a number of functions within `aurelius` which were prefixed 
by `pfa.*`. These functions were supporting PFA document creation, such as, `pfa.expr`,
which would convert an R expression to its PFA equivalent. However, in an effort to 
add producers to the package a generic S3 method `pfa()` has been introduced, which 
always produces a complete, valid PFA document from an R object (usually a model). 
These producers conflict because there are no models of class `expr`, which is how 
R interprets the function named `pfa.expr()`. In order to avoid this confusion 
these older `pfa.*` functions have been renamed to `pfa_*` similar to the change in 
Avro functions mentioned above.


```r

# pfa.expr no longer in use
pfa_expr(quote(2 + 2))
#> $`+`
#> $`+`[[1]]
#> [1] 2
#> 
#> $`+`[[2]]
#> [1] 2
```

# Functions to Read & Write PFA

The two functions `json()` & `unjson()` have been renamed to `read_pfa()` and 
`write_pfa()`. The motivation was to make it more obvious the behavior of these 
functions since there are many JSON related functions, so it's not 
confusing as to what the function `json()` actually does. The new names were taken 
with inspiration from the `xml2` package's naming conventions when dealing with 
XML documents. In addition to these name changes, the `write_pfa()` function has 
been modified to remove all whitespace which reduces the size of generated files. 
Tests with small documents showed ~10% reduction in size. This is similar to minifying
a CSS file to improve speed.


```r

  # convert the lm object to a list-of-lists PFA representation
  lm_model_as_pfa <- pfa(lm(mpg ~ hp, data = mtcars))
  
  # save as plain-text JSON
  write_pfa(lm_model_as_pfa, file = "my-model.pfa")
  
  my_model <- read_pfa("my-model.pfa")
  
```

# Functions to Create PFA

In addition to the changes in importing and exporting PFA, the function `pfa.config()` 
has been renamed to `pfa_document()` (also inspired by `xml2` package). It 
provides a method for creating a valid PFA document and, by default, warns if the 
document is not valid PFA using Titus-in-Aurelius (`pfa_engine()`)


```r

pfa_document(input = avro_double, 
             output = avro_double, 
             action = expression(input + 10), 
             validate = TRUE)
#> $input
#> [1] "double"
#> 
#> $output
#> [1] "double"
#> 
#> $action
#> $action[[1]]
#> $action[[1]]$`+`
#> $action[[1]]$`+`[[1]]
#> [1] "input"
#> 
#> $action[[1]]$`+`[[2]]
#> [1] 10
```

# More Support for Model Producers

The biggest change has been an expansion in model-to-PFA producers. Previously, 
only a couple model types were supported for a limited type of outputs, mainly 
for classification. Direct to PFA translation is available for almost all model 
types created by `gbm`, `glmnet`, and `randomForest` packages. More specifically: 

+ The `randomForest` functions only supported classification problems, now 
supports classification (majority vote) and regression (mean aggregation).

+ The `gbm` functions only supported classification fits, now supports: 

    * gaussian, laplace, tdist, huberized (regression data)
    * coxph (survival data)
    * poisson (count data)
    * bernoulli, adaboost (binary classification data)
    * multinomial (multinomial classification data)
  
+ The `glmnet` functions only supported classification fits, now supports:

    * guassian
    * binomial, 
    * poisson
    * cox 
    * multinomial
  
In addition, there is an option to control the prediction types generated by 
the PFA. For example, you might prefer a multinomial glmnet model to return 
the predicted probabilies of each class, or you might prefer the PFA to return 
only the predicted class. The new `pred_type` option allows users to specify this 
behavior. There is also an argument `cutoffs` which is helpful for classification 
problems where the user might not want to use the default cutoff for determining 
the predicted class. For example, typically in binomial classification if the 
predicted probability exceeds 50%, then that class is predicted. Now, the cutoffs 
function allows predicted classes to be chosen whenever the ratio of predicted 
probability to the cutoff is highest. This strategy was adopted from 
`randomForest.predict()`


```r

  # generate data
  x <- matrix(rnorm(100*3), 100, 3, dimnames = list(NULL, c('X1','X2', 'X3')))
  g3 <- sample(LETTERS[1:3], 100, replace=TRUE)
  
  # fit multinomial model without an intercept
  multinomial_model <- glmnet(x, g3, family="multinomial", intercept = FALSE)
  
  # convert to pfa, where the output is the predicted probability of each class
  # the cutoffs specify that the predicted class should be the one 
  # which is the largest relative to its specified cutoff.
  multinomial_model_as_pfa <- pfa(multinomial_model, 
                                  pred_type = 'response', 
                                  cutoffs = c(A = .1, B = .2, C = .7))
  
```

# S3 Methods to Extract and Build Models

In addition to new model producers S3 methods generic functions have been 
introduced as `extract_params()` and `build_model()` to provide a consistent way 
to retrieve model information that could be contructed into a PFA document. The 
purpose of having a consistent API is to better facilitate building PFA from 
model components whenever the user does not want to use a pre-canned producer.


```r

  # generate data
  dat <- data.frame(X1 = rnorm(100), 
                    X2 = runif(100))
  dat$Y <- ((3 - 4 * dat$X1 + 3 * dat$X2 + rnorm(100, 0, 4)) > 0)
  
  # build the model
  logit_model <- glm(Y ~ X1 + X2, data=dat, family = binomial(logit))
  
  # extract the parameters
  extract_params(logit_model)
#> $coeff
#> $coeff$X1
#> [1] -1.941884
#> 
#> $coeff$X2
#> [1] 0.3902524
#> 
#> 
#> $const
#> [1] 2.330089
#> 
#> $regressors
#> $regressors[[1]]
#> [1] "X1"
#> 
#> $regressors[[2]]
#> [1] "X2"
#> 
#> 
#> $family
#> [1] "binomial"
#> 
#> $link
#> [1] "logit"
```
