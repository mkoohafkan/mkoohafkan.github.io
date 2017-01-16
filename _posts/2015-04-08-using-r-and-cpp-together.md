---
layout:     post
title:      Using R and C++ together
date:       2015-04-08 12:00:00
summary:    Taking advantage of compiled code in R has never been easier, but you still need to pay attention to syntax.
categories: codemonkey rcpp r
commentIssueId: 17
redirect_from:
  - /codemonkey/rcpp/r/2015/04/08/using-r-and-cpp-together/
---

I recently developed the R package [rivr](https://github.com/mkoohafkan/rivr) 
for teaching open-channel hydraulics. 
Solving unsteady open-channel flow problems is often computationally expensive,
and an [interpreted language](http://en.wikipedia.org/wiki/Interpreted_language) 
like R is usually not the right choice for this sort of program. However, the 
[Rcpp](http://dirk.eddelbuettel.com/code/rcpp.html) package makes it really 
easy to embed C++ in your package. This means you can get the best of both 
worlds by seamlessly linking compiled code into your R package. There are some 
[pretty great references](http://gallery.rcpp.org/) for how and when you should 
use C++ in R, but I think the best way to include it in packages is via 
[Rcpp attributes](http://cran.r-project.org/web/packages/Rcpp/vignettes/Rcpp-attributes.pdf)---especially 
since it plays nice with `devtools` and `roxygen2`.

Programming with the C++ library provided by `Rcpp` looks a lot like 
programming in R. Consider the following function, first implemented in R and 
then equivalently in C++:

{% highlight r %}
# this is a silly function written in R
my_r_func = function(a, b){
  res = a ;
  if(a < b){
    for(i in  a:b){ 
      res = res*b ;
    }
  } else {
    for(i in b:a){
      res = res*a
    }
  }
  return(res)
}
{% endhighlight %}

{% highlight cpp %}
// this is a silly function written in C++
int my_cpp_func = function(int a, int b){
  int res = a ;
  if(a < b){
    for(int i = a; i < b ; i++){ 
      res *= b ;
    }
  } else {
    for(int i = b; i < a; i++){
      res *= a ;
    }
  }
  return(res) ;
}
{% endhighlight %}

They look basically the same! The most obvious differences are that 
(1) C++ functions and variables need to have their types explicitly defined, 
(2) loops are defined slightly differently, (3) C++ lines need to end with a 
semicolon, (4) you can use shortcuts like `a *= b` instead of writing `a = a*b`, 
and (5) C++ comments use `//` instead of `#`. Otherwise a lot of things are the
same; `if` and `while` statements have similar syntax, you use curly braces 
`{}` to group logic blocks, and you use `return` and `break` statements to exit 
out of functions and loops. 

There are a few other important differences that can trip you up though, and 
some of them are subtle. I've  made a short list of things for R programmers to 
keep in mind when writing C++ code with `Rcpp`.

#### Vector and matrix indices start at 0, rather than 1.

Consider the following function, first written in R and then in C++.

{% highlight R %}
r_select_element = function(s, i){
  return(s[i])
}
{% endhighlight %}

{% highlight cpp %}
double cpp_select_element(NumericVector s, int i){
  return(s[i]);
}
{% endhighlight %}

{% highlight R %}
# after loading C++ function in R
myvector = seq(5)
r_select_element(myvector, 1)    # will return first element
cpp_select_element(myvector, 0)  # will return first element
cpp_select_element(myvector, 1)  # will return second element

r_select_element(myvector, 5)    # will return last element
cpp_select_element(myvector, 4)  # will return last element
cpp_select_element(myvector, 5)  # will produce error
{% endhighlight %}

#### Use parentheses to refer to matrix elements. 

Indexing a vector in C++ uses square brackets `[]` (like R), but for matrices 
you must use parentheses `()`. To select an entire row or column of a matrix, 
use `_` in C++ .

{% highlight R %}
# indexing in R
A = matrix(rep(0, 12), nrow = 3) # create 3x4 matrix filled with zeros
x = A[1,1]                       # select element in first row and column
y = A[1,]                        # select first row
z = y[1]                         # select first element
{% endhighlight %}

{% highlight C++ %}
// indexing in C++
NumericMatrix A(3, 4, 0);        // create 3x4 matrix filled with zeros
double x = A(1, 0)               // select element in first row and column
double x = A[0, 0]               // will fail
NumericVector y = A(0,_)         // select first row
double z = y[0]                  // select first element
double z = y(0)                  // will fail
{% endhighlight %}

#### double quotes `""` are type `string`, but single quotes `''` are type `char`. 

This one took me an embarrassingly long time to figure out. You can't use 
logical operators to directly compare a `string` to a `char`, so you need to
be consistent on both sides of the equality.

{% highlight R %}
# string comparision in R
mystring = "foobar"
mystring[5] == "a"               # will return TRUE
mystring[5] == 'a'               # same as above
{% endhighlight %}

{% highlight cpp %}
// string comparision in C++
std::string mystring = "foo"
mystring.at(4) == 'a'            // will return TRUE
mystring.at(4) == "a"            // will fail
{% endhighlight %}

#### You can't store a function in a variable in C++, but you can use pointers to pass functions. 

Pointers allow you to point to specific locations in the program memory, which 
means you can induce 
[side-effects](http://en.wikipedia.org/wiki/Side_effect_(computer_science)) 
like change the underlying value of a variable or reference a function. 
You can do a lot with pointers, but they can also get you in trouble.

{% highlight R %}
# functions as variables in R
my_add = function(a, b){                 # create function
  return(a + b)
}
my_r_func = my_add                       # store function in variable
my_r_func(1, 2)                          # call function
{% endhighlight %}

{% highlight cpp %}
// using pointers to reference functions in C++
double cpp_my_add(double a, double b){   // create function
  return(a + b);
}
double (*cpp_myfunc) (double, double);   // create pointer by prepending with *
cpp_myfunc = &c_my_add;                  // point to existing function using &
cpp_myfunc(1, 2);                        // call function
{% endhighlight %}

Pointing to variables has slightly different syntax. I can't really think of an
equivalent operation in R!

{% highlight cpp %}
NumericVector myvec(10, 0);              //initialize a NumericVector of zeroes
NumericVector *pointvec;                 // create pointer by prepending with *
pointvec = &myvec                        // point to variable using &
*pointvec[0] = 5;                        // assign to first element via pointer

// you can also point to individual elements
double *pointelement;
pointelement = &myvec[0];
*pointelement = 5;                       // same as above 
{% endhighlight %}

Those were the main pitfalls I ran into as an R programmer learning C++. 
Hopefully these tips and references will let you hit the ground running.
 