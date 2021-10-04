
Paul van Loon
2021-10-04

Double initializer idiom
========================

C initializers can be used to initialize various C types, such as basic types, arrays, unions and structs. What I've never encountered before in other code, and what I recently created, is the use of what I call the "double initializer" idiom, where you initialize both a pointer and the pointed at value in the same expression.

Example:
```
int *p;
*(p=malloc (sizeof (int)))=2;
```
both p and *p are initialized in the same expression.

For such a trivial initialization, it might seem not very useful to use this idiom, however when using it to initialize both struct and pointers to structs, it will lead to much denser code than typically seen. However, I first need to show another idiom how to use an initializer to initialize pointers to structs. I've also not encountered this idiom in other code. What typically can be found is using a struct initializer like this:

```
struct s { int a, b; };
struct s v=(struct s){ 1, 2 };
```
Or with designated initializers like this:
```
struct s v=(struct s){ .a=1, .b=2 };
```
On StackOverflow the following idiom can be found to initialize a pointer to a struct [1]:
```
struct s *v=&(struct s){ 1, 2 };
```
However, the new idiom I created to initialize a pointer looks like this:
```
struct s *v;
*v=(struct s){ 1, 2 };
```
With this idiom, it is now possible to use the Double Initializer idiom to initialize both the pointer to the struct, and the pointed to struct in the same expression:
```
struct s *v;
*(v=malloc (sizeof (struct s)))=(struct s){ 1, 2 };
```
With a typedef and a define:
```
#define new(T) (malloc (sizeof (T))
typedef struct s S;
```
we get the surprisingly dense but readable:
```
S *v;
*(v=new (S))=(S){1, 2};
```
This comes very close to e.g. C++'s direct initialization syntax.

References
----------

[1] https://stackoverflow.com/questions/50860042/initialize-pointer-to-a-struct-with-a-compound-literal
