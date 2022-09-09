
Paul van Loon
2022-09-09

Generic functions with variable arguments
=========================================

C variable argument functions, that is functions declared with an ellipsis (...) as last function parameter, allow calling such functions with a potentially varying number of arguments.
The standard library declares several such functions, of which `printf` is probably the most well-known one.
The definition of such functions require processing of a list of a variable number of arguments, for which macros are defined in the `<stdarg.h>` header file.
Such processing is postponed until runtime when the function is actually called. Because limited compile time checks can be done, these functions can potentially lead to run time errors when called with the wrong number of parameters, or the wrong type of parameters.

However, these functions offer a unique capability to support genericity, as well as support "late binding". Here I describe idiom that uses variable argument functions, which I have not encountered before.

Genericity and late binding
---------------------------

Consider the problem of sorting a list of elements of unknown type. A generic sorting algorithm could sort such a list, if it would know how to order the elements, and now their sizes.
The `qsort` function in the standard library can do that by accepting a (function) pointer to a comparison function, as well as the element size. The signature of the comparison function is:
`int comparison_fn_t (const void *, const void *)` while the sort function itself has signature:
`void qsort (void *array, size_t count, size_t size, comparison_fn_t compare)`

Now consider a sort function that needs an extra parameter to the comparison function. If the type of the parameter would be `int`, we could design a sort function and comparison function as:
`void psort (void *array, size_t count, size_t size, (int)(*compare)(const void*,const void*,int), int)`
where we pass the parameter as last argument to `psort`, and `psort` passes it as last argument to the comparison function.

But for every variant in the type of parameter, with this approach, we need to create a separate sort function. But here we can resort to a different technique, which I here introduce, using variable argument functions.
If we make a generic sort function take a variable number of arguments in its declaration, then pass that to the comparison function, it's only the comparison function that need to be aware of the type of the parameter,
hence encapsulating type information (at least for the parameter), and making the sort function more generic.

The prototype of this generic sort function, let's call it `generic_sort`, looks like:
`void generic_sort (void *array, size_t count, size_t size, generic_comparison_fn_t compare, ...)`
Now what should the signature of generic_comparison_fn_t look like? To that end we need to make a partial implementation using variable argument macros:

```
void generic_sort (void *array, size_t count, size_t size, generic_comparison_fn_t compare, ...) {
  ..
  va_list ap;
  va_start (compare);
  ..
  compare (a, b, ap);
  ..
  va_end ap;
  ..
}
```

Therefore the signature of `generic_comparison_fn_t` should look like:
`int generic_comparison_fn_t (const void *, const void *, va_list ap)`
and the partial implementation of an instance of `generic_compare` would have to include:

```
int generic_compare (const void *, const void *, va_list ap) {
  ..
  int parameter=va_arg (ap, int);
  ..
  return ..;
}
```

But this pattern not only covers a single argument, but by virtue of the variable argument list feature and the `va_list` type, *any* number of arguments.
We therefore have introduced a truly generic sort function that can accept comparison functions with a variable number of arguments, without having to have intimate knowledge about such comparison functions.
The invariant that must be guarded by the programmer is that calls to `generic_sort` use the correct number and types of arguments as required by the `generic_comparison_fn_t` instance.

When looking at how to call such functions, we note a different property:
```
  int compare_updown (const void*, const void*, enum { UP, DOWN });
  ..
  generic_sort (a, n, s, compare_updown, UP);
  ..
  generic_sort (b, m, t, compare_updown, DOWN);
  ..
```
Calling `generic_sort` with `compare_updown` and parameters `UP` and `DOWN` effectively enables the concept of "late binding" in C, where the parameters can be seen to be bound late (i.e. at runtime), instead of the usual early binding, which is accomplished by the common use of multiple static functions for `qsort`, e.g. two functions `compare_up` and `compare_down`. Such late binding is a desired property in itself, and the same pattern can be used for that.

Generic evaluators
------------------

We looked at genericity and late binding, for generic functions that take a single function argument, with a variable number of parameters.
By the virtue of the `va_arg` macro in some compilers (e.g. GNU GCC), it *is* supported to keep processing the `va_list` after a call to function that accepts a `va_list` argument.
That enables a whole class of generic functions, which I call *generic evaluators*; consider a function `eval` with a variable number of arguments, where the list of variable arguments start after a certain argument 'A'.
By convention make the first argument a pointer to a function accepting a `va_list` as at least one of its parameters, or 0. Let the generic evaluator implement:
```
typedef .. V;
typedef V (*F)(va_list);
typedef .. T;

T eval (..A, ...) {
  ..
  va_list ap;
  va_start (ap, A);
  ..
  for (F f;f=va_arg (ap, F);) {
    ..
    f (ap);
    ..
  }
  ..
}
```

then each of the functions of type F will consume just as much arguments as the need, and the immediately following argument is again a function pointer of type F, or 0.
Without loss of generality, the use of 0 can be replaced by a prefix argument that counts the number of functions that are going to get called by the generic evaluator,
but that requires duplication of information of the number of arguments.

Which such a generic evaluator, a whole plethora of classic algorithms can be expressed as function calls where the function argument list can be seen as doing "list processing".
Of course each argument can be any expression, possibly the result of calling another generic evaluator.

One such classic algorithm is the interpreter, and implementing an interpreter as a generic evaluator, makes DSL-like constructs feasible in plain C.
