_Note: This project serves as a placeholder.  Once it is done I hope to have it added to [Go Proposals](https://github.com/golang/proposal).
If that happens this project will be deleted_

# Generics with constraints for Go

This is a proposal for adding generics with constraints to the Go language. 
It is written by Christian Surlykke, fall of 2017.

There have been lots of discussion on how generics in Go could look. 
In particular Ian Lance Taylor have written detailed proposals
[15292 Generics](https://github.com/golang/proposal/blob/master/design/15292-generics.md), 
[Type Functions](https://github.com/golang/proposal/blob/master/design/15292/2010-06-type-functions.md), 
[Generalized Types](https://github.com/golang/proposal/blob/master/design/15292/2011-03-gen.md), 
[Generalized Types in Go](https://github.com/golang/proposal/blob/master/design/15292/2013-10-gen.md) and lastly
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 
outlining how to add generics to Go. 

Most of the discussion has taken place in issue [15292](https://github.com/golang/go/issues/15292).

In [Generalized Types](https://github.com/golang/proposal/blob/master/design/15292/2011-03-gen.md), 
Taylor describes an implementation, where the compiler would analyze a generic definition and extract _restrictions_ from it.

For example, given a generic definition:
```Go
func [T] Add(x1,x2 T) T {
	return x1 + x2
}
```
A compiler should extract the restriction 'addable' - that a concrete type substituted for `T` must support the addition operator, 
and the compiler would use that knowledge in checking the validity of concrete instantiations of `Add`.

What I want to do here is develop an alternative to that: Rather than having the compiler extract restrictions, 
put the onus on the developer to declare these restrictions _explicitly_.

So I propose generics for Go with _constraints_:

Generics means that types and functions may be _parametrized_, ie. depend on type parameters. 
They may be instantiated with _concrete types_ substituted for the type parameters.

Constraints means that a generic declaration may restrict what concrete types can be substituted for 
a type parameter. 

What you can do with a type parameter in a generic definition is limited to what you can do with the types allowed by its constraints.
So to call a method on a type parameter, all types allowed by its constraint must have that method. 
A similar rule applies to fields, operators and functions.

The aim is that the correctness of a generic definition can be verified at the point of _definition_:  
When instantiating a generic type, the compiler only needs to verify that the concrete types substituted for the type parameters 
fulfill the given constraints, and then the validity of the instantiation follows.

I believe that with this, the compiler can be made simpler, the code will be more readable and that it will allow for better tooling.

## Generics

Before moving on to constraints, lets briefly look at generics.

Type- and function definitions at package level may be generic. 
That means they may depend on type parameters. 

Here's an example of a generic type `List` of `T` where `T` is a type parameter:

```Go
type [T] List struct {
	value T
	next *List[T]
}
```

We may also define a generic function:

```Go
func [T] Push(l *List[T], t T) *List[T]{
	return &List[T]{t, l}
}
```

Here, we do not call any methods on `value`, access any fields, or apply any operators to `T`. 
As we haven't said anything about `T`,
we can't make any assumptions about what can be done with it.

We can use our generic definition like this:

```Go
var myList *List[int] = nil

myList = Push[int](&myList, 2)
myList = Push[int](&myList, 4)

for l := &myList; l != null; l = l.next {
	fmt.Println(l.value)
}
```

### A note about the syntax

I follow the syntax proposed by Taylor, except that I don't use type deduction. 
I prefer to state explicitly the value of type parameters, and it seems to me that type deduction is 
a form of overloading. Something which was deliberately left out of Go 1.

This is a minor point, though.
One could add type deduction to this proposal at the cost, I guess, of some increase in compiler complexity.

## Constraints

A constraint limits what concrete types may be substituted for a type parameter. 
It can do so in 3 ways:

* Require the type to have specific _methods_
* Require the type to have specific _fields_
* Restrict what _underlying types_ the type may have. 

We'll look at each of these in turn.

### Methods

Let's say we want to ensure a type has the a method `Log()`.

We could then define a constraint like this:

```Go
constraint Loggable { 
	Log()
}
```

Here `constraint` is a new keyword which marks the start of a constraint declaration.

We can use `Loggable` in a generic declaration:

```Go
func [T Loggable] DoLog(t T) {
	T.Log() // Allowed because T is constrained to be a type that has method Log()
}
```

### Fields

A constraint with a field can look like this:

```Go 
constraint Entity {
	Id uint64
}
```

It would be satisfied by any type having a field name `Id` of type `uint64`

An example use could be:

```Go
func [E Entity] Equal(e1, e2 E) bool {
	return e1.Id == e2.Id
}
```


Fields and methods can be mixed in constraints:

```Go
constraint LoggableEntity {
	Id uint64
	Log()
}
```

There may be any number of methods and fields in a constraint.
Names may not be reused.

### Generic constraints

Constraints may be generic:

```Go
constraint [T] Cloneable {
	Clone() T
}

func [T Cloneable[T]] copy(t T) {
	return t.Clone()
}
```

### Underlying types

A constraint may limit what underlying types a type can have. An example

```Go
constraint PositiveInt {
	:uint8, uint16, uint32, uint64, uint
}
```

A type must have one of `uint8`, `uint16`, `uint32`, `uint64` _or_ `uint` as underlying type to satisfy this constraint.

The format of an underlying type specification is a line starting with `:` followed by a
comma separated list of types. 

A constraint declaration may have _at most_ one underlying type specification.

Absense of an underlying type specification means that any underlying type is allowed.

Underlying type may be one of Go's predeclared boolean, string, numeric or string types, 
or it may be a struct, slice, channel, array, pointer or function type.

A couple of examples:

```Go
constraint SliceOfInt {
	[]int
}
```

A generic variant: 


```Go
constraint [T] Slicy {
	[]T
}
```

Pointers:

```Go
constraint PointerToInt {
	:*int
}
```

A generic pointer constraint

```Go
constraint [T] Pointer {
	:*T
}
```

#### Pseudo types

There are a couple of sets of types that cannot be described by what has been introduced so far. 
To remedy that we introduce a couple of _pseudo type_ for use in constraints.

The first is _comparable_, denoted `==`. It is used like this:

```Go
constraint Comparable {
	:==
}
```
To satisfy constraint `Comparable` a type must be just that - comparable, ie. have a comparable underlying type.

`Comparable` might be used like this:

```Go
type [K Comparable, V] Map [K]V
```
The second is _array_, denoted by [#]<type>. 
To declare that the underlying type must be an array of uin32, you'd write:

```Go
constraint Uint32Array {
	:[#]uint32
}
```

We will not offer a way to specify the length of the array.


### Embedding 

Just like interfaces, a constraint can _embed_ an other constraint

```Go
type Foo constraint {
	...
}

type Baa constraint {
	Foo
	...
}

```

whereby `Baa` gets all methods, fields of `Foo`. 

There is no 'overriding', which means:

* The name of a field or method on `Baa` may not be the same as a name of a field or method on `Foo`
* Having a list of underlying types both in `Foo` and `Baa` is an error.
### Comparing constraints

Constraints are comparable. Two constraints are equivalent if and only if

* They have the same set of methods.
* They have the same set of fields.
* They allow the same underlying types.

Constraints are partially ordered by a relation 'implies'.
Constraint `C1` implies constraint `C2` if and only if:

* Every method on `C2` is a method on `C1`
* Every field on `C2` is a field on `C1`
* Every underlying type allowed by `C1` is allowed by `C2`

### Standard constraints

A number of constraints will be so common that they should be included in Go's standard library. These include:

```Go
constraint Addable { 
	:string, 
	 uint8, uint16, uint32, uint64, uint, 
	 int8, int16, int32, int64, int,
	 float32, float64,
	 complex64, complex128 
}

constraint Arithmetic { 
	:uint8, uint16, uint32, uint64, uint, 
	 int8, int16, int32, int64, int,
	 float32, float64,
	 complex64, complex128
}

constraint Integer { 
	:uint8, uint16, uint32, uint64, uint, 
	 int8, int16, int32, int64, int
}

constraint Ordered { 
	:string, 
	 uint8, uint16, uint32, uint64, uint, 
	 int8, int16, int32, int64, int,
	 float32, float64
}
```

This concludes our description of constraints. Now we'll turn back our attention to generics.


## More on generics

### Type methods

Generic types can have methods:

```Go
type [T C1] Foo ...

func [T C1] (f Foo[T]) M() {

}
```

The target type must have the same number of type parameters as in the definition of the type.
And each of the type parameters must be declared to implement the same constraint (if any) as in the definition of the type.

Hence this 

```Go
type [T Constraint1] Foo ...

func [T Constraint2] (f Foo[T]) M() {
}
```

is allowed only if Constraint2 is equal to Constraint1.


### Overloading generic definitions

Overloading of generic definitions with the _same_ number of type parameters is _not_ allowed: 

```Go
type [T Constraint1] Foo ... 
                          
type [T Constraint2] Foo ... // Not allowed
```

The problem would be that if `Baa` implements both `Constraint1` and `Constraint2`, then `Foo[Baa]` would be ambigous with
respect to which definition to employ.

Overloading with _different_ number of generic parameters _is_ allowed.
So:

```Go
type [S Constraint1] Foo ...
type [T Constraint2, U Constraint3] Foo ...
```

is ok.

### Runtime

Concrete instantions of generic types and functions should look like ordinary hand coded ones. Given:


```Go
type [T] MyType ...
```

and a concrete instatiation, say `MyType[int]`, you can do casts, just like with non-generic types:

```Go
.(MyType[int])
```

The instantiation `MyType[int]` is a concrete type just like any other (non-generic) type in Go. 
In fact, from a runtime perspective, the only thing that gives away it's origin in a generic definition is
the appearence of `[` and `]` in it's name.

Constraints have no presence at runtime. 
It is not possible to cast to constraints, declare variables to instantiate constraints or to use reflection 
to query about constraints.

## Examples

### Apples and Oranges

Typesafe fruit-count comparison could be done like:

```Go

constraint PositiveInt { :uint8, uint16, uint32, uint64, uint }

func [Counter PositiveInt] Compare(c1, c2 Counter) int {
	if c1 < c2 {           // We know all possible underlying types of Counter support '<'
		return -1
	} else if c1 == c2 {   // - and are comparable
		return 0
	} else {
		return 1
	}
}

type Applecount uint32
type OrangeCount uint32

var apples1, apples2 Applecount = 3,4
var oranges1, oranges2 OrangeCount = 7,6

Compare[Applecount](apples1, apples2)    // Yields 1
Compare[Orangecount](oranges1, oranges2) // Yields -1

```

With no way to compare apples and oranges.

### Joining sequences

This is an adaption of an example from Taylors 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 

```Go
constraint ByteSequence {
	:[]byte, string
}

[T ByteSequence ]func Join(a []T, sep T) T {
	if len(a) == 0 {
		return []T{}
	}
	if len(a) == 1 {
		return a[0]
	}
	n := len(sep) * (len(a) - 1)
	for _, v := range a {
		n += len(v)
	}
	b := make(byte[], n)
	bp := copy(b, a[0])
	for _, v := range a[1:] {
		bp += copy(b[bp:], sep)
		bp += copy(b[bp:], v)
	}
	return T(b)
}
```

## Comparison to Taylors proposal

There are a number of examples in Taylors proposal, most of which can be rewritten to use constraints as we have described them here.
We leave that as an excercise for the reader.

In a section about 
[type checking](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md#type-checking), 
Taylor offers a list of restrictions that the compile might extract from generic declarations. In short form these are:

* addable: _Underlying type is `string` or a number type_ 
* integral: _??_ 
* numeric: _??_
* boolean: _Underlying type is `bool`_
* comparable: _Underlying type is a comparable type_
* ordered: _Underlying type is `string` or a non-complex number type_
* callable
* composite
* points to type U
* indexable with value type U
* sliceable with value type U
* map type with value type U
* has field or method F of type U
* chan of type U
* convertible from U
* convertible to U
* assignable from U
* assignable to U

For most of these it is evident how to translate it to constraints. 

#### Callable

for `callable` this proposal is not quite as powerful as Taylors. 
You _can_ constrain a type parameter to have a function type as it's underlying type:

```Go
constraint CallableWithIntAndFloat {
	:func(i int, f float) 
}

func [F CanCallWithIntAndFloat] Caller(f F) {
	f(3, 7.3)
}
```

This will work for functions that have no return value. 

There is no way of covering all functions that take an int and a float as arguments.

#### Assignments and conversions

Assignabillity and convertability follows from the underlying types of the parameters involved. 
Go has these two rules:

* An expression of type `T` is assignable to a variable of type `U` if `T` and `U` have the same underlying type
  and `T` or `U` is an unnamed type.
* An expression of type `T` is convertible to type `U` if `T` and `U` have same underlying type.

With these, one can determine assignability and convertability based on what underlying types type parameters are constrained to.

## Further considerations

Whats written above constitutes my proposal for constraints in Go. 
Now I'll mention some variations and additions that could be considered.

### Specialization 

In C++ template programming you may employ specialization and SFINAE to customize a generic definition for certain types. 

Template specialization relies on overloading, and would not, in my opinion, be a good fit for Go. 

SFINAE basically means that you can create several definitions of a generic type/function and the compiler will decide at 
point of instantiation which one 'works'. This is obviously at odds with the goals of this proposal.

On the other hand some kind of conditional behaviour for generic definitions could be beneficial. Consider the `Max` function:

```Go
func [T Ordered] Max(t1,t2 T) T {
	if t1 < t2 {
		return t2
	} else {
		return t1
	}
}
```

It would be nice to extend this function to types that are not ordered, but have a method `LessThan`:

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

We can't have these two definitions at the same time as overloading is not allowed. 
One could consider some form of _static switch_:

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
```

The idea would be:

* The compiler (still) verifies the correctness of this code at the point of definition. 
* On instantiation apply the first switch case that matches (ie. `T` satisfies the constraint).
* If no case matches, the compiler issues a compile error, 
  in this case something like "Type must implement one of `Ordered`, `Lessable`."
* A `default` entry is allowed, so that all types can be handled in one switch.    
    
I have not included type switch in this proposal.
While this construct works nicely for the `Max` example, 
I'm not convinced that it's usefulness is broad enough to warrant its addition to Go. 
Also I'm not sure about the syntax.

I feel the subject is complicated enough that it should be considered separately.

### Variadic definitions

Another element left out of this proposal is _variadic definitions_. Something like:

```Go
type [T...Constraint] MyVaridicType ?? 
```

This would introduce a variable number of type parameters all bound by `Constraint`.

I've put in a couple of question marks because it is unclear, to me at least, how to write an actual definition
based on a variable number of type parameters. 
C++ uses recursion which in turn relies on overload, and that is not a good fit for Go. 
I think one would have to add a number of primitives to Go including:

* A way of getting the number of type parameters at intantiation time
* A way of addressing the n'th type parameter 
* A way of packing a list of variables into a struct and a way of 'spreading' a struct to a list of variables

I don't think variadics is useful enough to warrant addition of such complexity.

Overloading with respect to number of type parametes _is_ allowed so it is possible to do:

```Go
type [T1] tuple struct { t1 T1 }
type [T1, T2] tuple struct { t1 T1, t2 T2 }
type [T1, T2, T3] tuple struct { t1 T1, t2 T2, t3 T3 }
```
etc. 

So if you want to do tuples, you can define them and accompanying functions/methods up to, say, 20 elements covering most usecases. 
It will involve some writing, but once done, from a usage perspective about just as useful as if it had been defined with
variadic generics. Go gen may be helpful. All in all I do not think omission of variadic definitions is a major problem.

## Comparison with other languages

Here I'll make a brief comparison with Java and C++. I can't offer thoughts on any other language as Java and C++ are the only
statically typed languages with generics that I have any experience with. 

### Java

In java generic type parameters are constrained with the keyword `extends` like:

```Java
class MyClass<L extends Loggable> {
	private L l;
}
```

`Loggable` may be a class or an interface, and any method defined on `Loggable` may here be applied to field `l`. 

This works well in java where everything is a pointer to an object. 
It has served as inspiration for this proposal, though what's described here is quite different due to the language differences.

### C++

C++ has a template programming system that is very powerful. 
It's implementation is based on the idea of template expansion and it is heavily used in stdlib. 
Those two factors combined means that using a template type may lead to many levels of expansion which in turn, if there is an error,
can lead to horrific error messages. 

The C++ community is attempting to fix this by introducing 'concepts', currently slated for inclusion in C++20. 
C++ concepts describe traits that a type parameter may be required to have, 
and they would be checked at the point of instantiation, to allow the compiler to issue simpler warnings. 

C++ concepts are a lot more complicated than the constraint system I've described here. 

One reason is that Go's type system is much simpler. Consider:

```Go
a < b
```

when you see an expresson like that in a Go program, you know that

* `a` and `b` have same type
* The underlying type of `a` and `b` must be `string` or one of the non-complex number types.
* The result of the expression is `bool`

In C++ you know a lot less - the types of `a` and `b` can be anything. An the result is not even guaranteed to be boolean. 


### Backward compability

Apart from the introduction of a new keywork - `constraint` - this proposal is backwards compatible 
in the sense that it would not alter the meaning or validity of any program valid under Go 1.x. 

One could make it fully backward compatible by replacing `constraint` with some, today, nonlegal sequence of characters.

I have no strong preferences regarding this. I've used `constraint` here as it conveys the underlying idea most clearly.

## Summary

I'm under no illusions that this proposal will be adopted, but I hope that it may in some way benefit the discussion on 
generics in Go. 

I do believe that 'type safe generics', ie. a form of generics that the compiler can check at the point of declaration,
is the best way to go.

This proposal is the best idea for type safe generics I've been able to come up with. Maybe others can come up with something
better, perhaps inspired by this.





