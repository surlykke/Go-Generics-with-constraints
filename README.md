# Constraints for Go Generics

The Go team has published a [proposal]() for generics in Go.

The proposal describes _contracts_ as a way of _constraining_ type parameters.
Please read the proposal for details, but, briefly, a contract is a short 
function body, using a generic type parameter:

Eg: 

```Go
contract Add(t T) {
	t + t
}
```
This contract, `Add`, would be used to require that the operator `+` should 
apply to type `T`.

Here I'd like to present an alternative to contracts: _constraints_. 

Constraints limits what types may be substituted for a parameter in a generic 
definition, in two ways:

- Require types to implement **interfaces**. 
- Limit what **underlying types** a type may have.

## Why underlying types

Several people have suggested using interfaces to constrain types.
The idea is very appealing in that it uses a construct already in Go. 
Interfaces, however, have the limitation that they cannot express 
applicabillity of operators. 

For that I believe underlying types are ideal. In Go, the question of whether 
an operator may be applied to a type comes down to what underlying type it has. 
In fact, the contract `Add` above implies that `T`'s underlying type must be
either `string` or one of the numbertypes.

In my opinion the contract `Add` expresses that in an unnecessarily convoluted 
way.

Go's type system is very strict in that:

* The use of operators is determined by underlying type.
* The operands of arithmetic- or comparison binary operators must be of same 
  type. For arithmetic operators the type of the result will be that of the 
  operands.
* Legality of casts between types is determined by underlying types.

This strictness is a big asset of Go, and should be taken advantage of.

## Constraining with interfaces

So this proposals first way to constrain a type parameter is _requiring it to 
implement an interface_. Let's say we have:

```Go
type Loggable interface { 
	Log()
}
```

We can use `Loggable` in a generic declaration:

```Go
func (type T Loggable) DoLog(t T) {
	T.Log() // Allowed because T is constrained by Loggable 
}
```

Instantiating `DoLog` whith a type not implementing `Loggable` is an error.

## Underlying types

The second way of constraining is _limiting what underlying types a type 
parameter may have_.
An underlying type constraint is written as a comma separated list of allowed 
underlying types enclosed in curly braces.
Eg.

```Go
{uint8, uint16, uint32}
```
means the underlying type must be `uint8`, `uint16` or `uint32`.

We can use this in a generic declaration:

```Go
func(type T {uint8, uint16, uint32}) Mul(t1, t2 T) T {
	return t1*t2
}
```
Here the compiler knows that `*` may be applied to any of `T`'s possible 
underlying types, and hence to `T`.
Also, due to the strictness of Go's type system, we know that `t1*t2` will 
yield a result of type `T`.

On the other hand, a declaration like this:

```Go
func(type T {string, uint8}) Mul(t1, t2 T) T {
	return t1*t2
}
```
would give an error, as `string` is an allowed underlying type, and it does not
support the multiplication operator.

Instantiating `Mul` with a type of which the underlying type is not in the 
allowed set, is an error.

## Combining constraints

Constraints may be combined using the `&` operator (conjunction):

```Go
Loggable & {uint8, uint16, uint32}
```
This constraint will be satified by a type implementing `Loggable` _and_ having 
one of `uint8`, `uint16` or `uint32` as its underlying type.

An example:

```Go
func MulAndLog(type T Loggable & {uint8, uint16, uint32})(t1,t2 T) {
	(t1*t2).Log()
}
```
Here, the compiler knows from `T`'s possible underlying types that `*` may be 
applied.
It knows that the result of the multiplication is of type `T`, so method 
`Log()` may be invoked.

Some combinations are useless, say: `{uint8} & {uint16}`, as a type cannot 
have two underlying types. The compiler should issue an error when encountering
an unsatisfiable constraint.

## Named constraints

A constraint may be given a name. For that we introduce a new keyword, 
`constraint`. An example:

```Go
constraint MyConstraint Loggable & {uint8, uint16, uint32}
```

`MyConstraint` may now be used in generic declarations like:

```Go
func myFunc(type T MyConstraint) (t T) ...
```

Or in the creation of new contraints:

```Go
MyConstraint & SomeOtherInterface
```

## Generic constraints

Constraints may be based on generic interfaces:

```Go
type interface(type T) Cloneable {
	Clone() T
}

func copy(type T Cloneable(T)) (t T) {
	return t.Clone()
}
```

Also, when map, slice, array or channel are used as underlying type, they may
depend on type parameters. Ie:

```Go
constraint ListOf(type T) {[]T}
```

Generic constraints may _not_ involve constrained type parameters.
Ie. this:

```Go
type MyInterface(type T SomeConstraint) {...}

func myFunc(type U MyInterface) (...) {...} // Error
```

is _not_ allowed.

Nor any constraint declaration of form:

```Go
constraint MyList(type T SomeConstraint)... // Error
```

## Comparable

There will be one _built-in_ constraint: `Comparable`. To satisfy `Comparable`
a type must be just that - comparable, ie. have a comparable underlying type.

An example use of `Comparable`:

```Go
type Map(type K Comparable, V) [K]V
```

`Comparable` is logically equivalent to a constraint listing the infinite set
of comparable fundamental types as underlying.

## Standard constraints

A number of constraints will be so common that they should be included in Go's 
standard library. 
These include:

```Go
constraint Addable {string, uint8, uint16, uint32, uint64,
     uint, int8, int16, int32, int64, int, float32, float64,
	 complex64, complex128}

constraint Arithmetic {uint8, uint16, uint32, uint64, uint,
	 int8, int16, int32, int64, int, float32, float64,
	 complex64, complex128}

constraint Integer {uint8, uint16, uint32, uint64, uint,
	 int8, int16, int32, int64, int}

constraint PositiveInteger {uint8, uint16, uint32, uint64, uint}

constraint Ordered {string, uint8, uint16, uint32, uint64, uint,
	 int8, int16, int32, int64, int, float32, float64}
```

(Obviously one can bikeshed about the names.)

## Comparing constraints

In principle a constraint could be represented by a pair made from the set of 
method signatures and the set of allowed underlying types. 
If the constraint is about interfaces only, the set of underlying types is the
set of all possible underlying types.

If `C1` and `C2` are constraints, the following rules apply:

* The method set of `C1 & C2` is the _union_ of `C1` and `C2`'s method sets.
* The underlying type set of `C1 & C2` is the _intersection_ of `C1` and `C2`'s
  underlying type sets.
* `C1` and `C2`are equal if their method sets are equal and their underlying 
  type sets are equal.
* `C1` _implies_ `C2` if both of these conditions hold :  
  - `C1`'s method set is a _superset_ of `C2`'s method set  
  - `C1`'s underlying type set is a _subset_ of `C2`'s underlying type set.
* A constraint is _unsatisfiable_ (and an error) if it's set of underlying 
  types is empty.

## Partial instantiation

A generic type may be used in another generic declaration if the compiler is 
able to verify that constraints are satisfied. For example:

```Go
type Foo(type T C1) ...

func f(type U C2)(Foo(U) foo) ...
```
Here `C2` must _imply_ `C1` for the declaration to be valid. 

## Built in functions

Go's [built in](https://golang.org/pkg/builtin) functions may be applied to 
type parameters if the parameter is constrained to have underlying types that 
support the built in function in question. 

Specifically, with `T` a generic type parameter:

### Type functions 
* `func make(T Type, size ...IntegerType) Type`: `T`'s underlying type must be 
  map, slice or array.
* `func new(T Type) *Type`: Will work for any type. 

### Container functions

* `func append(t T, elems ...U)`:
  The underlying type of `T` must be `[]U`
* `func len(t T) int`: The underlying type of `T` must be a slice, string, 
  channel, array or pointer to array.
* `func cap(t T) int`: The underlying type of `T` must be a slice, a channel, 
  an array or a pointer to an array.
* `func close(t T)`: The underlying type of `T` must be a writeable channel.
* `func copy(dst, src T) int`: The underlying type of `T` must be a slice.
* `func delete(t T, key K)`: The underlying type of `T` must be `[K]V` for some 
  `V`.

## Assigning and Casting 

Consider, in a context where `t` is of generic type `T` and x is some 
(non-generic) expression:

```Go
t = x
t = (T)x
```

Let's say `T` is constrained to a finite set of underlying types. 
Then assume, for each possible underlying type `u`, that `T` is defined as:
```
type T u
```
and check the validity of the expressions above according to Go's rules. If
that checks outfor each underlying type, the expression may be used in a 
generic definition.

If the set of possible underlying types of `T` is inifinite (ie. `T` is not 
constrained wrt. underlying types or `T` is just constrained to be `Comparable`
) the expressions above are not valid.


## Runtime

Constraints have no presence at runtime. 
It is not possible to cast to constraints, declare variables to instantiate 
constraints or to use reflection to query about constraints.

## Optional features 

What's written above is the core of my proposal.
Here follows a few additions that could be nice to have.

I'll go though them in descending nice-to-have'ness order.

### Static switch

Consider this `Max` function:

```Go
func Max(type T Ordered)(t1,t2 T) T { 
	if t1 < t2 {
		return t2
	} else {
		return t1
	}
}
```

We would like to extend this function to types that are not `Ordered`, but 
have a method `LessThan`:

```Go
interface ComparesTo(type T) {
	LessThan(t T) bool 
}

func  Max(type T ComparesTo(T))(t1, t2 T) T {
	if t1.LessThan(t2) {
		return t2
	} else {
		return t1
	}
}
```

We can't have these two definitions at the same time as overloading is not 
allowed. 

To solve this we introduce _static switch_:

```Go
func  Max(type T)(t1, t2 T) T switch {
	case T Ordered:
		if t1 < t2 {
			return t2
		} else {
			return t1
		}
	 case T ComparesTo(T):
		if t1.isLessThan(t2) {
			return t2
		} else {
			return t1
		}
}
```

The rules are: 

* A static switch may only appear in generic function declarations, and only at 
  the top level, immediately after the method signature
* Each case expresson constrains one or more type parameters. Inside that branch 
  methods and operations allowed by the constraints may be used.
* `fallthrough` is not allowed.
* On instantiation with a concrete type, the compiler runs through the cases, 
  applying the first where the the constraints is satisfied.
* If no cases match, the compiler issues an error, in this case something like 
  _type_ must 
  be `Ordered` or `Lessable`.

If a function depends on more than one generic type, it could look like:

```Go
func F(type T1, T2)(t1 T1, t2 T2) Returntype switch {
	case T1 C1, T2 C2: 
		...
	case T1 C3, T2 C4: 
		...
}
```
with `C1`, `C2`, `C3` and `C4` being constraints.

One may find the (ab)use of `switch` obnoxius, but I think the idea of guarding 
code blocks with constraints has merrits.

Unlike overloading, which is heavily used in C++, there is no ambiguities 
on what to apply, and the ability to specialize generic definitions is 
important.

### Generic arrays

With whats described so far we can specify that the underlying type should
be an array, eg: 
```Go
:[5]T
```

This is not very flexible, as it doesn't allow abstraction with respect to
array length, so we could extend the underlying type notation to allow 
_wildcards_ for arraylength. 

The constraint 
```Go
{[%]uint32}
``` 
whould then allow _any_ array of `uint32` as underlying type (Not to be 
confused with `{[]uint32}` which allows _slices_ of `uint32`). 

Arrays aren't used that much in Go programming (explicitly, that is) and I
don't expect generics to change that. 
It might be useful for heavy numeric calculations (like matrix multiplication)
but then again an optimizing compiler may be able to make calculations through
slices as fast as calculations don directly on arrays.
I don't know, actually.

### Fields 

The proposal does not offer any way to specify that a type parameter must have 
a _field_. 

One could add that capability in a couple of ways:

* Extend Go's interfaces to specify fields. This is probably not going to 
  happen. I believe something like this has already been proposed on Go's issue
  tracker (not in the context of generics, though) and rejected.
* Add some way of specifying fields to the constraint syntax. Eg:
  ```Go
  .Id:uint64
  ```
  to constrain a type to have a field `Id` of type `uint64`.
  
  It could be used like:
  ```Go
  func ShowId(type T .Id:uint64)(t T) {
      fmt.Println(t.Id)
  }
  ```
  Evidently the syntax could be varied in several ways. 

In a previous version of this proposal I included fields in constraints, 
but I have since dropped them, as I don't think they are important enough to 
warrant the complexity they introduce.

## Concluding remarks

This proposal is an alternative to the contracs idea that the Go team has 
forwarded. 

I think there is a worthwhile discussion to be had, on whether traits of 
generic type parameters should be specified 'by code' or by a more 'direct' 
declaration.

As you may guess I'm not a huge fan of specification by code. It feels to me 
like a form of unit-testing at compile-time, and I believe that it will lead to 
a high level of complexity, if not in implementation then at least
in readability. 

Another question is whether _this_ proposal is a good one, or whether some 
other declarative approach would be better.  
It is highly probable that better approaches exists, but I'm putting this 
forward, hoping that it may add value to the discussion on Go generics.

## About

This is version 2 of my constraints-for-go-generics proposal. Major changes 
since version 1:

* Constraining Types to have specific _fields_ have been omitted, simplifying 
  things considerably.
* Syntax has been adapted to match the generics proposal from the Go team.
* Paragraphs that where not deemed specifically pertinent to _constraints_ 
  have been ruthlessly removed.

