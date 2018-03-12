_Note: This project serves as a placeholder.  
Once it is done I hope to have it added to 
[Go Proposals](https://github.com/golang/proposal).
If that happens this project will be deleted_

# Generics with constraints for Go

This is a proposal for adding generics with constraints to the Go language. 
It is written by Christian Surlykke, fall of 2017, winter 2017/2018.

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
func [T] Add(x1,x2 T) T {
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
    when instantiating a genering definition.
1.  By that limitation they define _what you can do_ with a type parameter 
    in a generic definition. Eg. an operator can only be applied to a type 
	parameter, if the parameter is constrained so that _all_ allowed types 
	support that operator.

The aim is that the correctness of a generic definition can be verified at the 
point of _definition_:  
When instantiating a generic type, the compiler only needs to verify that the 
concrete types fulfill the given constraints, and the validity of the 
instantiation will follow.

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

Here's an example, from 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md), 
of a generic type `List` of `T` where `T` is a type parameter:

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

We can use our generic definition like this:

```Go
var myList *List[int] = nil

myList = Push[int](myList, 2)
myList = Push[int](myList, 4)

for l := myList; l != null; l = l.next {
	fmt.Println(l.value)
}
```

## Constraints

A constraint limits what concrete types may be substituted for a type 
parameter. 
It can do so in 3 ways:

* Require the type to have specific _methods_
* Require the type to have specific _fields_
* Restrict what _underlying types_ the type may have. 

We'll look at each of these in turn.

### Methods

Let's say we want to ensure a type has the a method `Log()`.  We could define 
this constraint:

```Go
constraint Loggable { 
	Log()
}
```

Here `constraint` is a new keyword which marks the start of a constraint 
declaration. 
The declaration defines the constraint `Loggable`. 
To satisfy `Loggable` a type must have a method with signature `Log()`.

We can use `Loggable` in a generic declaration:

```Go
func [T Loggable] DoLog(t T) {
	T.Log() // Allowed because T is constrained by Loggable 
}
```

As long as we deal with methods only, constraints look a lot like interfaces.

### Fields

A constraint with a field can look like this:

```Go 
constraint Entity {
	Id uint64
}
```

It would be satisfied by any type having a field named `Id` of type `uint64`.
Obviously this means that the type is a struct of sorts.

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

Type parameters in generic constraint declarations _may not_ be constrained:

```Go
constraint Foo {..,}
constraint [T Foo] Baa {...} // Not allowed
```

### Underlying types

A constraint may limit what underlying types a type can have. 
An example

```Go
constraint PositiveInt {
	:uint8, uint16, uint32, uint64, uint
}
```

A type must have one of `uint8`, `uint16`, `uint32`, `uint64` _or_ `uint` as 
its underlying type to satisfy this constraint.

The format of a declaration of underlying types is a line starting with `:` 
followed by a comma separated list of the allowed types. 

A constraint may declare _at most_ one set of underlying types, and absence of 
an underlying type declaration means that _any_ underlying type is allowed.

Underlying type may be one of Go's predeclared boolean, string, numeric or 
string types, or it may be a struct, slice, channel, array, pointer or 
function type.

Declaration of underlying types may _not_ be combined with field declarations: 
If you define a field constraint you implicitly restrict the 
underlying type to be some struct with such a field. 
Hence the the only underlying type constraints that would make sense are 
structs with said field, but with those the original field constraint would be 
superfluous.

### Arrays

Constaints described so far allow specifying that the underlying type should 
be an array, eg:

```
constraint[T] ArrayOf5 {
	:[5]T
}
```

This is not very flexible, as it doesn't allow abstraction with respect to 
array length, so as a special case we allow wildcard for that, using the 
character `#`. 

For example to  declare that the underlying type must be an array of uin32, 
you'd write:

```Go
constraint Uint32Array {
	:[#]uint32
}
```
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

### Comparable

There will be one _built-in_ constraint: `Comparable`. To satisfy `Comparable`
a type must be just that - comparable, ie. have a comparable underlying type.

`Comparable` might be used like this:

```Go
type [K Comparable, V] Map [K]V
```

`Comparable` is logically equivalent to a constraint listing the infinite set
of comparable fundamental types as underlying.

### Embedding 

Analogously to interfaces, a constraint can _embed_ another constraint

```Go
constraint Foo {
	...
}

constraint Baa {
	Foo
	...
}

```

whereby `Baa` gets all methods, fields of `Foo`. 

There is no 'overriding', which means:

* The name of a field or method on `Baa` may not be the same as a name of a 
  field or method on `Foo`
* Having an underlying types constraint both in `Foo` and `Baa` is an error.
* Having a field constraint in `Foo` or `Baa` and an underlying types 
  constraint in `Foo` or `Baa` is an error
* If a constraint embeds `Comparable` (directly or indirectly) it cannot 
  explicitly define a  list of underlying types.

### Comparing constraints

Constraints are comparable. 
Two constraints are equivalent if and only if

* They have the same set of methods.
* They have the same set of fields.
* They allow the same underlying types.

Constraints are partially ordered by a relation 'implies'.
Constraint `C1` implies constraint `C2` if and only if:

* Every method on `C2` is a method on `C1`
* Every field on `C2` is a field on `C1`
* Every underlying type allowed by `C1` is allowed by `C2`

### Standard constraints

A number of constraints will be so common that they might be included in Go's 
standard library. 
These include:

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

constraint PositiveInteger {
	:uint8, uint16, uint32, uint64, uint, 
}

constraint Ordered { 
	:string, 
	 uint8, uint16, uint32, uint64, uint, 
	 int8, int16, int32, int64, int,
	 float32, float64
}
```

### Runtime

Constraints have no presence at runtime. 
It is not possible to cast to constraints, declare variables to instantiate 
constraints or to use reflection to query about constraints.

This concludes our description of constraints. 
Now we'll turn back to generics.


## More on generics

### Type methods

Generic types can have methods:

```Go
type [T C1] Foo ...

func [T C1] (f Foo[T]) M() {

}
```

The target type must have the same number of type parameters as in the 
definition of the type.
And each of the type parameters must be declared to implement the same 
constraint (if any) as in the definition of the type.

Hence this 

```Go
type [T Constraint1] Foo ...

func [T Constraint2] (f Foo[T]) M() {
}
```

is allowed only if Constraint2 is _equal_ to Constraint1.

### Partial instantiation

A generic type may be used in another generic declaration if the compiler is 
able to verify that constraints are fulfilled. For example:

```Go
type[T Constraint1] Foo ...

func[U Constraint2] f(Foo[U] foo) ...
```

Here `Constraint2` must imply `Constraint1` for the declaration to be valid. 

### Overloading generic definitions

Overloading of generic definitions with the _same_ number of type parameters is 
_not_ allowed: 

```Go
type [T Constraint1] Foo ... 
                          
type [T Constraint2] Foo ... // Not allowed
```

The problem would be that if `Baa` implements both `Constraint1` and 
`Constraint2`, then `Foo[Baa]` would be ambigous with respect to which 
definition to employ.

Overloading with _different_ numbers of generic parameters _is_ allowed.
So:

```Go
type [S Constraint1] Foo ...
type [T Constraint2, U Constraint3] Foo ...
```

is ok.

## Examples

### Apples and Oranges

Typesafe fruit-count comparison could be done like:

```Go

func [Counter PositiveInteger] Compare(c1, c2 Counter) int {
	if c1 < c2 {         // We know Counter supports '<'
		return -1
	} else if c1 == c2 { // We know Counter is comparable
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

### Generic singleton factory

```
constraint Initializable {
	Initialize()
}

func[T] TypeOf() Type {
	var t T
	return reflect.TypeOf(t);
}

var repository [Type]interface{}

func[T Initializable] Factory() *T {
	var typeOfT = TypeOf[T]()

	if _, ok := repository[typeOfT]; !ok {
		var t T
		(&t).Initialize();
		repository[typeOfT] = &t
	}

	return repository[typeOfT].(*T)
}

```
which could be used as:

```Go
type MyStruct struct {
	....
}

func (ms *MyStruct) Initialize() {
	...
}


var myStruct = Singleton[MyStruct]();
```

The function `TypeOf` is executed at runtime invoking reflection. One might
consider adding a built-in function TypeOf which could be executed at compile
time (instatiation time, that is).


### Joining sequences

This is an adaption of an example from Taylors 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 

```Go
constraint ByteSequence {
	:[]byte, string
}

func [T ByteSequence] Join(a []T, sep T) T {
	if len(a) == 0 {
		return T([]byte{})
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
* composite: Type is constrained to have a field or underlying type is a struct
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
constraint CallableWithIntAndFloat {
	:func(i int, f float) 
}

func [F CanCallWithIntAndFloat] Caller(f F) {
	f(3, 7.3)
}
```

This will work for functions that have no return value. 

There is no way of covering _all_ functions that take an int and a float as 
arguments.
So, when dealing with callables, constraints, as proposed here, will make 
generics less flexible than what Taylor has proposed.

#### Assignments and conversions

In order to assign/convert to/from a generic type parameter it is nessecary
that:

* The parameter is constrained to have one single underlying type
* What you assign/convert to/from is of that underlying type

Again generics as Taylor has proposed is more flexible. By postponing 
verification of correctnes to the point of instantiation, the compiler can
know the concrete types being substituted for parameters, and allow/disallow
the operations based on that knowledge. 

## Static Switch

Whats written above constitutes my proposal for constraints in Go. 

Here I'll describe static switch, a form of _specialization_ which I have 
considered, but decided not to add to this proposal.

Consider the `Max` function:

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

One could add _static switch_ to Go:

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

The idea would be:
* Each case is guarded by a constraint. Inside the case block methods, fields 
  and operations defined by the guarding constraint may be used.
* On instantiation the compiler should apply the first switch case that matches
  (ie. `T` satisfies the constraint).
* If no case matches, the compiler issues a error, something like: _`T` must 
  fullfil one of constraints `Ordered`, `Lessable`._
* A `default` entry is allowed, so that all types can be handled in one switch.    
    
While this construct works nicely for the `Max` example, I'm not convinced 
that it's usefulness is broad enough to warrant its addition to Go. 
Also I'm not sure about the syntax.

I feel the subject is complicated enough that it should be considered 
separately.

## Comparison with other languages

Here I'll make a brief comparison with Java and C++. 
I can't offer thoughts on any other language as Java and C++ are the only 
statically typed languages with generics that I have any substantial 
experience with. 

### Java

In java generic type parameters are constrained with the keyword `extends` 
like:

```Java
class MyClass<L extends Loggable> {
	private L l;
}
```

`Loggable` may be a class or an interface, and any concrete type substituted 
for `L` must extend/implement `Loggable`. 

Thus any method defined on `Loggable` may here be applied to field `l`. 

This works well in java where everything is a pointer to an object, and where 
inheritance plays a central role.
It has served as inspiration for this proposal, though what's described here 
is quite different due to the language differences.

### C++

C++ has a template programming system that is very powerful. 
It's implementation is based on the idea of template expansion and it is 
heavily used in stdlib. 
Those two factors combined means that using a template type may lead to many 
levels of expansion which in turn, if the event of an error, can lead to quite
horrific error messages. 

The C++ community is attempting to fix this by introducing 'concepts', 
currently slated for inclusion in C++20.  

C++ concepts describe traits that a type parameter or a set of type parameters
may be required to have, and they would be checked at the point of 
instantiation, to allow the compiler to issue simpler warnings. 

C++ concepts are both a a lot more powerful and a lot more complicated than the 
constraint system I've described here.
One factor that allows for some simplicity in this proposal is Go's type 
system: 
Consider the expression `a < b`.
When you encounter that in a Go program, you know that

* `a` and `b` have same type
* The underlying type of `a` and `b` must be `string` or one of the 
  non-complex number types.
* The result of the expression is `bool`

In C++ you'd know a lot less - `a` and `b` may have any type as may the result.

Also, if I understand correctly, while C++ concepts limits what concrete types 
may be substituted for a type parameter, they do _not_ limit what may be done
with a type parameter in a generic definition, so they do not provide 'generic
type safety' like this proposal does.

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
generics in Go (or any language, for that matter). 

This proposal is the best idea for type safe generics I've been able to come 
up with. 
Surely others can come up with something better, perhaps in part inspired by 
this.



