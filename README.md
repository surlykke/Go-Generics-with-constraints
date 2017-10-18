_Note: This project serves as a placeholder.
Once it is done I hope to have it added to 
[Go Proposals](https://github.com/golang/proposal).
If that happens this project will be deleted_

# Concepts and Generics for Go

This is a proposal for generics in the Go language using concepts. It is written by Christian Surlykke, fall of 2017.

There have been lots of discussion on how generics in Go could look. 
In particular Ian Lance Taylor have written detailed proposals  
[15292 Generics](https://github.com/golang/proposal/blob/master/design/15292-generics.md), 
[Type Functions](https://github.com/golang/proposal/blob/master/design/15292/2010-06-type-functions.md), 
[Generalized Types](https://github.com/golang/proposal/blob/master/design/15292/2011-03-gen.md), 
[Generalized Types in Go](https://github.com/golang/proposal/blob/master/design/15292/2013-10-gen.md) and 
[Type Parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md) 
outlining various ways of doing generics in Go. 

Most of the discussion has taken place in issue [15292](https://github.com/golang/go/issues/15292).

In the first couple of  proposals Taylor works with the idea that a type parameter in a generic declaration could be _constrained_ by requiring it to implement an _interface_.

In the latest proposal Taylor seems to abandon this approach in favor of having the compiler _extract_ constraints from a generic definition.

What I want to do with this proposal is develop the first idea fully: Introduce a new language construct, `concept` with wich
it can be constrained what kind of operations may be performed on a type parameter in a generic definition.

The aim is that the correctness of a generic definition can be verified at the point of _definition_. 
When instatiating a generic type or function, the compiler should just need to verify that the concrete type satisfies the constraints given, and it should not be nessecary to expand the code and verify it's correctness at that point.

With that I believe it will be possible to write generic code that is more readable.
I believe it will simplify the compiler and I believe it will allow better tooling.

## Concepts

I propose to add to Go a new construct: `concept`. 

A concept defines a set of _methods_, _fields_, _operators_ and built-in _fuctions_. 

A type parameter in a generic definition may be declared to _implement_ a concept. 
This means that any concrete type _instantiating_ the parameter must have the methods, fields, operators and built-in functions that the
concept defines. 

In a generic definition the _only_ allowed operations on a type parameter are those defined by a concept that the parameter implements.


Example: Let us say we have a concept `Loggable` defining a method `Log()`. A generic function can be defined:

```Go
\L Loggable/ func DoLog(l L) {
	l.Log()
}
```

A concrete type may implement `Loggable`:

```Go
type Foo {}

func (f Foo) Log() {
	println "Hello from Foo"
}
```

And you can use it:


```Go
var f Foo

DoLog\Foo/(f)
```


### About the syntax

As you can see, I use `\` and `/` as delimiters. I prefer not to overload  `[]` or `()` with a new usage, and I prefer
`MyFunc\Foo/(f)` to `MyFunc(Foo)(f)`.

To my knowledge, there is no other legal use of backslash in Go outside strings and runes. 
Hence there is no need for a keyword, like 'gen', to mark the start of a generic declaration. `\` will serve that purpose.

The main drawback is that if you nest: `Foo\Baa\T//` the last pair of forward slashes will be interpreted as line comment. 
You'll have to insert a space: `Foo\Baa\T/ /`, or perhaps slightly nicer: `Foo\ Baa\T/ /`.

This is reminiscent of the `>>` issue in C++ template programming. In my mind it's not a huge problem. 
If your editor does syntax coloring, it should be easy to spot, and I assume a check for this could easily be added to golint.

I would like to point out, though, that the choice of delimiters is in no way essential to this proposal. 
It could easily be rewritten to use another pair of delimiters and/or to use a keyword like `gen`. 

This proposal is really about concepts.


## Defining concepts


### Methods

A concept with methods looks very much like an interface:

```Go
type Loggable concept {
	Log() 
}
```

Concepts can be generic:

```Go
\T/ type Cloneable concept {
	Clone() T
}
```
  
So with: 

```
type Person struct {
	Name string
}

(p Person) func Clone() Person {
	return Person{p.Name}
}
	
```

one can do:

```Go
\C Cloneable\C/ / func copy(c C) {
	return c.CLone()
}
```

As is seen here, CRTP is allowed.

### Fields

Fields on concepts are defined like this:

```Go
concept Entity {
	Id uint64
}
```

A generic function: 

```Go
\T Entity/ func ShowId(t T) { 
	fmt.Println("Id is: ", t.Id)
}
```

An implementation:

```Go
type Employee struct {
	Id uint64
	Name string
}
```

wich can be used:

```
var employee = Employee{1, "Chr. Surlykke"}

ShowId\Employee/(employee)
```


### Operators and built-in functions

There will be no syntax to define operators on concepts. 
Instead we will define a set of _built in_ concepts that have operators and standard functions associated with them.

One example of a built in concept is `Ordered`. 
`Ordered` defines the comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=` and is implemented by 
string-, floating-point- and integer types.

As an example:

```Go
\T Ordered/ func Min(t1 t2 T) T {
	if t1 < t2 {
		return t1 
	} else {
		return t2
	}
}
```

The full list of built-in concepts are:

|**Concept**     |Implemented by                     | Operators               | Built in functions              | 
|---------------:| :---------------------------------|:------------------------|:-------------------------------:|
|Comparable      | Any comparable type               |==,!=                    |                                 |
|Ordered         | integers, floats or string        |==, !=, <,>,<=,>=        |                                 |
|Additive        | numbers or string                 |+, +=                    |                                 |
|Number          | numbers                           |+,-,\*,/                 |                                 |
|Integer         | integers                          |+,-,\*,/<<,>>,^,&,&#124; |                                 |
|Bool			 | bool                              |==,!=,&&,||,!            |                                 |
|String          | string                            |[],+,==,!=,<,>,<=,>=     |                                 |
|Array\T/        | array or slice of T               |[]                       | make, len, cap                  |
|Slice\T/        | slice of T                        |[]                       | make, append, len, cap, copy    | 
|Map\K Ordered,V/| map[K]V                           |[]                       | make, len, delete               |
|RChan\T/        | <-chan T                          |<- (read)                | make, close, cap                |
|WChan\T/        | chan<- T                          |<- (write)               | make, close, cap                |
|Chan\T/         | chan T                            |<- (read and write)      | make, close, cap                |


A concept that is implemented by a type T is also implemented by any type that has T as underlying type.

I've included `Bool` in the table.
That may seem a strange choice: 
How would you write a generic type or function with a Bool parameter? Well, Go allows this:

```
type MyBool bool
```

which creates a type distinct from bool but succeptible to the same operations. 
This allows generic definitions handling all such types.
I must, however, admit I'm not able to come up with a convincing use for such types, so let's say I've added `Bool` for completeness.

I've also included `String` in the table. You can derive types from string in the same way as for bool, but here the usefulness 
is more obvious.


As the table shows, some of the built-in concepts have _functions_ associated with them. 
For example `Array` defines the functions `make`, `len` and `cap`, so a for type `S` implementing `Array\T/` the expression

```Go
make(S, initialSize, initialCapacity)
```

is valid and returns an instance of `S`.

## Embedding 

Just like interfaces, a concept can _embed_ an other 

```Go
type Foo concept {
	...
}

type Baa concept {
	Foo
	...
}

```

whereby Baa gets all methods, fields, operators and functions of Foo.

You could say that some of the built-in concepts embeds one another: For example the concept 'Integer' embeds the concept 'Number', 
but since these concepts are to be built-in, and not defined in explicit go-source, that is of minor importance.

## Interfaces as concepts

An interface may be used in place of a concept. Example:

```Go
type Stringer interface {
	String() 
}

\T Stringer/
PrintableVector []T
```

## Overloading generic definitions

Overloading of generic definitions with the _same_ number of type parameters is _not_ allowed: 

```Go
\T Concept1/ type Foo ... 
                          
\T Concept2/ type Foo ... // Not allowed
```

The problem would be that if `Baa` implements both `Concept1` and `Concept2`, then `Foo\Baa/` would be ambigous with
respect to which definition to employ.

Overloading with _different_ number of generic parameters _is_ allowed.
So:

```Go
\S Concept1/ type Foo ...
\T Concept2, U Concept3/ type Foo ...
```

is ok.


## Runtime

Concrete instantions of generic types and functions should look like ordinary hand coded ones. Given:


```Go
\Foo C/ MyType ...

type Baa ... // Implements C
```

you can cast:

```Go
.(MyType\Baa/)
```

The instantiation `MyType\Baa/` is a concrete type just like any other (non-generic) type in Go. 
In fact, from a runtime perspective, the only thing that gives away it's origin in a generic definition is
the appearence of `\` and `/` in it's name.

Concepts have no presence at runtime. 
It is not possible to cast to concepts, declare variables to instantiate concepts or to use reflection to query about concepts.


## Further considerations

Whats written above constitutes my proposal for concepts in Go. 
Here I'll mention some variations and additions that could be considered.

### Extending interfaces rather than introducing concepts

Instead of introducing `concept` as a new language construct, one could extend interfaces to specify fields, 
and add built-in interfaces, as we did with concepts above.

This would have the distinct advantage of not burdening the Go language with a new construct. 
I have not taken that route for 2 reasons:

* 	Concepts and interfaces may share syntax, but they _are_ different. 
	A concept exists only at compile time, whereas an interface is represented at runtime as a pointer with type info.
*	It is, to me at least, not entirely clear that interfaces can be extended without any negative effect 
	on compile- and runtime performance for existing interface usage. We should strive for 'no play, no pay' when
	introducing generics in Go.

TODO: Specialization, Variadic templates.

