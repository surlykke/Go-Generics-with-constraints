# Generics with constraints for Go

This is a proposal for adding generics with constraints to the Go language. 
It is written by Christian Surlykke 2017/2018.

There have been lots of discussion on how generics in Go could look. 
In particular Ian Lance Taylor have written detailed proposals:
[15292 Generics](https://github.com/golang/proposal/blob/master/design/15292-generics.md), 
[Type Functions](https://github.com/golang/proposal/blob/master/design/15292/2010-06-type-functions.md), 
[Generalized Types](https://github.com/golang/proposal/blob/master/design/15292/2011-03-gen.md), 
[Generalized Types in Go](https://github.com/golang/proposal/blob/master/design/15292/2013-10-gen.md) 
and lastly
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 
outlining how to add generics to Go. 

Most of the discussion has taken place in issue 
[15292](https://github.com/golang/go/issues/15292).

A few places Taylor mentions _concepts_: 
Ways of restricting what concrete types may be used in place of type parameters 
on instantiation.

In 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 
Taylor abandons constraints in favour of having the compiler _extract_ 
restrictions from definitions.

For example, given a generic definition:
```Go
func Add(type T)(x1,x2 T) T {
	return x1 + x2
}
```
a compiler would extract the restriction 'addable' - that a concrete type 
substituted for `T` must support the addition operator, 
and the compiler would use that knowledge in checking the validity of concrete 
instantiations of `Add`.

What I want to do here is double down on the idea of constraining type 
parameters. 
Rather than having the compiler extract restrictions, require the developer to 
declare these restrictions _explicitly_.

So I propose generics for Go with _constraints_. 

Constraints serve a double purpose:

1.  They _limit_ what concrete types may be substituted for a type parameter 
    when instantiating a generic definition.
1.  By that limitation they define _what you can do_ with a type parameter 
    in a generic definition. Eg. an operator can only be applied to a type 
	parameter, if the parameter is constrained so that _all_ allowed types 
	support that operator.

The aim is that the correctness of a generic definition can be verified at the 
point of _definition_.  
When instantiating a generic type, the compiler only needs to verify that the 
concrete types fulfill the given constraints, and the validity of the 
instantiation shall follow.

I believe that with this, the compiler can be made simpler, the code will be 
more readable and that it will allow for better tooling.

This proposal should read as an amendment to Taylor's proposal. 
It is about adding constraints to generics. 
I do not want to go into other discussions relating to generics, such as 
syntax, choice of delimiters, implementation etc. except to the extend that 
constraints have any bearing on these subjects.

I don't use type deduction in this proposal. 
Again, I'm not advocating for or against that, but I  prefer to state 
explicitly the value of type parameters for clarity.

## Generics

Before moving on to constraints, lets briefly recapitulate how generics looks 
according to Taylors proposal.

Type- and function definitions at package level may be _generic_. 
That means they may depend on type parameters.
This is an example, from 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md), 
of a generic type `List` of `T` where `T` is a type parameter:

```Go
type List(type T) struct {
	value T
	next *List(T)
}
```

A generic function may also be defined:

```Go
func Push(type T)(l *List(T), t T) *List(T){
	return &List(T){t, l}
}
```

The generic definitions can be used like this:

```Go
var myList *List(int) = nil

myList = Push(int)(myList, 2)
myList = Push(int)(myList, 4)

for l := myList; l != null; l = l.next {
	fmt.Println(l.value)
}
```

## Constraints

A constraint limits what concrete types may be substituted for a type 
parameter. 
It can do so in 2 ways:

* Require the type to implement _interfaces_
* Restrict what _underlying types_ the type may have. 

### Interfaces

Let's say we want to ensure a type has a method `Log()`.  We can define 
an interface in a well known way:

```Go
type Loggable interface { 
	Log()
}
```

We can use interface `Loggable` to _constrain_ a type parameter in a generic 
declaration:

```Go
func (type T Loggable) DoLog(t T) {
	T.Log() // Allowed because T is constrained by Loggable 
}
```
When instantiating `DoLog` with a concrete type, the compiler should issue an 
error if the type substituted for `T` does not implement `Loggable`.

### Underlying types

We may constrain a type parameter by limiting what underlying types it can 
have. 

An underlying-type constraint has the form of a colon followed by a parenthesized 
list of the allowed underlying types separated by `||`. For example:

```Go
:(uint8 || uint16 || uint32)
```

A type must have one of `uint8`, `uint16` or `uint32` as its underlying type 
to satisfy this constraint.

We can use this in a generic declaration:

```Go
func(type T :(uint8 || uint16 || uint32)) mul(t1, t2 T) T {
	return t1*t2
}
```
Here the compiler knows T to have `uint8`, `uint16` or `uint32` as 
underlying type, hence the multiplication operator can be applied. 
Also, due to the strictness of Go's type system, the result of `t1*t2` will be 
`T`.

OTOH a declaration like this:

```Go
func(type T :(uint8 || string)) mul(t1, t2 T) T {
	return t1*t2
}
```
would give an error, as `T` may have `string` as underlying type, which does 
not support `*`.

If the constraint specifies only _one_ underlying type, the paranthesis may be 
omitted. So `:(uint8)` is equivalent to `:uint8`.

### Arrays

With whats described so far we can specify that the underlying type should 
be an array, eg: `:[5]T`

This is not very flexible, as it doesn't allow abstraction with respect to 
array length, so as a special case we allow wildcard for that, using the 
character `#`. 
So to declare that the underlying type must be an array of uin32, 
you can write: `:[#]uint32`

which would be satisfied by any type having an array of `uint32` as underlying 
type.

A generic variant:

```
constraint[T] Array {
	:[#]T
}
```
Arrays aren't used that much in Go programming (explicitly, that is) and I 
don't expect generics to change that. This array notation is included mainly 
for completeness.

### Combining constraints

Constraints may be combined using the `&&` operator (conjunction):

```Go
Loggable && :(uint8 || uint16)
```

This constraint would be satified by a type implementing `Loggable` _and_ 
having either `uint8` or `uint16` as its underlying type.

### Named constraints

A constraint may be given a name. For that we introduce a new keyword, 
`constraint`. An example:

```Go
constraint MyConstraint Loggable && :(uint8 || uint16)
```

`MyConstraint` may now be used in generic declarations like:

```Go
func [T MyConstraint] myFunc(t T) ...
```

Or in the creation of new contraint names:

```Go
constraint MyOtherConstraint MyConstraint && SomeOtherInterface
```

Named constraints may be generic, ie:

```Go
constraint List(type T) :[T]
```

TODO: Example

### Generic constraints

Constraints may be based on generic interfaces:

```Go
type interface(type T) Cloneable {
	Clone() T
}

func [T Cloneable[T]] copy(t T) {
	return t.Clone()
}
```

Also underlying types may also depend on type parameters, eg: `:[]T`

`
When using a generic interface or a generic underlying type in a constraint, 
these may _not_ be defined using constraints. Ie. this:

```Go
type (type T SomeConstraint) MyInterface {...}

func (type U MyInterface) myFunc(...) {...}
```

is _not_ allowed.

Nor is:

```Go
constraint MyList(type T SomeConstraint) :[]T

func (type T MyList) myFunc(...) {...} 
```

### Comparable

There will be one _built-in_ constraint: `Comparable`. To satisfy `Comparable`
a type must be just that - comparable, ie. have a comparable underlying type.

`Comparable` might be used like this:

```Go
type (type K Comparable, V] Map [K]V
```

`Comparable` is logically equivalent to a constraint listing the infinite set
of comparable fundamental types as underlying.

### Standard constraints

A number of constraints will be so common that they might be included in Go's 
standard library. 
These include:

```Go
constraint Addable :( :string || uint8 || uint16 || uint32 || uint64 || 
     uint || int8 || int16 || int32 || int64 || int || float32 || float64 ||
	 complex64 || complex128 )

constraint Arithmetic ( :uint8 || uint16 || uint32 || uint64 || uint || 
	 int8 || int16 || int32 || int64 || int || float32 || float64 ||
	 complex64 || complex128 )

constraint Integer ( :uint8 || uint16 || uint32 || uint64 || uint || 
	 int8 || int16 || int32 || int64 || int)

constraint PositiveInteger ( :uint8 || uint16 || uint32 || uint64 || uint)

constraint Ordered ( :string || uint8 || uint16 || uint32 || uint64 || uint || 
	 int8 || int16 || int32 || int64 || int || float32 || float64)
```

(One can bikeshed about the names, obviously.)

### Comparing constraints

Constraints are comparable. 
Two constraints are equivalent if and only if

* They have the same set of methods.
* They allow the same underlying types.

Constraints are partially ordered by a relation 'implies'.
Constraint `C1` implies constraint `C2` if and only if:

* Every method on `C2` is a method on `C1`
* Every underlying type allowed by `C1` is allowed by `C2`

## Specialization: Static Switch

To facilliate _specialization_ in generic declarations, we introduce _static
switch_. 

Consider this `Max` function:

```Go
func [T Ordered] Max(t1,t2 T) T {
	if t1 < t2 {
		return t2
	} else {
		return t1
	}
}
```

It would be nice to extend this function to types that are not ordered, but 
have a method `LessThan`:

```Go
constraint [T] Lessable { 
	LessThan(T) bool 
}

func [T Lessable] Max(t1, t2 T) T {
	if t1.LessThan(t2) {
		return t2
	} else {
		return t1
	}
}
```

We can't have these two definitions at the same time as overloading is not 
allowed. 

We can solve this with a static switch.

```Go
func [T] Max(t1, t2 T) T {
	switch[T] {
	case Ordered:
		if t1 < t2 {
			return t2
		} else {
			return t1
		}
	case Lessable: 
		if t1.LessThan(t2) {
			return t2
		} else {
			return t1
		}
	}
}
```

The rules are: 

* Each `case` is guarded by a constraint. Inside that `case` methods 
  and operations allowed by the guarding constraint may be used.
* On instantiation the compiler should apply the first switch case that matches
  (ie. `T` satisfies the constraint). 
  
  Subsequent cases are skipped.
* If no case matches, the compiler issues a error, something like: _`T` must 
  fullfil one of constraints `Ordered`, `Lessable`._
* A `default` entry is allowed, so that all types can be handled in one switch.    
    

### Built in functions

Go's [built in](https://golang.org/pkg/builtin) functions may be applied to 
type parameters if the parameter is constrained to have underlying types that 
support the built in function in question. 

Specifically:

#### Type functions 
* `func make(T Type, size ...IntegerType) Type`: `T`'s underlying type must be 
  map, slice or array.
* `func new(T Type) *Type`: Will work for any type. 

#### Container functions
* `func append(slice V, elems ...U)`:
  The underlying type of `V` must be `[]U`
* `func len(v V) int`: The underlying type of `V` must be a slice, string, 
  channel, array or pointer to array.
* `func cap(v V) int`: The underlying type of `V` must be a slice, a channel, 
  an array or a pointer to an array.
* `func close(c C)`: The underlying type of `C` must be a writeable channel.
* `func copy(dst, src S) int`: The underlying type of `S` must be a slice.
* `func delete(m M, key K)`: The underlying type of `M` must be `[K]V` for some 
  `V`.

### Runtime

Constraints have no presence at runtime. 
It is not possible to cast to constraints, declare variables to instantiate 
constraints or to use reflection to query about constraints.

## More on generics

What's written above concludes my description of constraints. 
Now I'll turn back to generics.


### Type methods

Generic types can have methods:

```Go
type Foo(type T C1)  ...

func (foo Foo(T)]  M() {

}
```

### Partial instantiation

A generic type may be used in another generic declaration if the compiler is 
able to verify that constraints are fulfilled. For example:

```Go
type Foo(type T Constraint1) ...

func f(type U Constraint2)(Foo(U) foo) ...
```
Here `Constraint2` must imply `Constraint1` for the declaration to be valid. 

### Overloading generic definitions

Overloading of generic definitions with the _same_ number of type parameters is 
_not_ allowed: 

```Go
type  Foo(type T Constraint1) ... 
                          
type Foo(type T Constraint2)  ... // Not allowed
```

The problem would be that if `Baa` implements both `Constraint1` and 
`Constraint2`, then `Foo(Baa)` would be ambigous with respect to which 
definition to employ.

Overloading with _different_ numbers of generic parameters _is_ allowed.
So:

```Go
type Foo(type S Constraint1)  ...
type Foo(type T Constraint2, U Constraint3) ...
```

is ok.


## Comparison to Taylors proposal

There are a number of examples in Taylors proposal, most of which can be 
rewritten to use constraints as we have described them here.

In a section about 
[type checking](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md#type-checking), 
Taylor offers a list of restrictions that the compile might extract from 
generic declarations. 

Many of those translate straight forward to constraints:

* addable: `Addable`
* integral: `Integer`
* numeric: `Arithmetic`
* boolean: Underlying type is `bool`
* comparable: `Comparable`
* ordered: `Ordered`
* composite: Type is constrained to have a field or its underlying type is a 
  struct
* points to type U: Underlying type is *U
* indexable with value type U: Underlying type is `[#]U` or `[]U`
* sliceable with value type U: Underlying type is `[#]U` or `[]U`
* map type with value type U: Underlying type is `[K]U` for some `K`
* has field or method F of type U: Type is required to have field or method.
* chan of type U: Underlying type is `chan U`

Others arguably demonstrate weaknesses in generics with constraints:

* callable:
* convertible from U:
* convertible to U:
* assignable from U:
* assignable to U:

#### Callable

We can constrain a type parameter to have a function type as it's 
underlying type:

```Go
constraint CallableWithIntAndFloat :func(i int, f float) 

func Caller[type F CallableWithIntAndFloat)(f F) {
	f(3, 7.3)
}
```

This will work for functions that have no return value. 

There is no way of covering _all_ functions that take an int and a float as 
arguments. 
So, when dealing with callables, constraints, as proposed here, will make 
generics less flexible than what Taylor has proposed.

#### Assignments and conversions

If a generic definition depends on two type parameters it will not be possible 
to assign or cast between them. The constraint system proposed here does not 
allow us to verify at the point of declaration that such an assignment or cast
is legal. 

To fix that, one might go the route chosen by C++ concepts where you can 
introduce constraints _between_ type parameters, ie. simply constrain the pair
`U` and `V` so that instances of V may be assigned to instances of U. 

I believe that would lead to a constraint system much more complicated than 
what is proposed here, and, in my opinion, not worth the bother.

But, in this respect,  generics as Taylor has proposed is certainly more
flexible. 
By postponing verification of correctnes to the point of instantiation, 
the compiler can know the concrete types being substituted for parameters, and 
allow/disallow these operations based on that knowledge.  

### Backward compability

Apart from the introduction of a new keywork - `constraint` - this proposal is 
backwards compatible in the sense that it would not alter the meaning or 
validity of any program valid under Go 1.x. 

One could make it fully backward compatible by replacing `constraint` with 
some, today, nonlegal sequence of characters.

I have no strong preferences regarding this. 
I've used the keywork `constraint` here as it most clearly conveys the 
underlying idea.

## Summary

I'm under no illusions that this proposal will be adopted, but I hope that it 
may in some way benefit the discussion on generics in Go. 

I _do_ believe that 'type safe generics', ie. a form of generics that the 
compiler can check at the point of declaration, is the best way to implement
generics in Go. 

This proposal is the best idea for type safe generics I've been able to come 
up with. 
Surely others can come up with something better, perhaps in part inspired by 
this.



