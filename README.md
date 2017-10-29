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
concept Loggable {
	Log() 
}
```

Concepts can be generic:

```Go
\T/ concept Cloneable {
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

* `Comparable`
  * Operators: `==`, `!=`
  * Built in functions: _none_
  * Implemented by: _Any comparable type_
* `Ordered`
  * Operators: `==`, `!=`, `<`, `>`, `<=`, `>=`
  * Built in functions: _none_
  * Implemented by: _integer types_, _float types_ and `string`
* `Addable`
  * Operators: `+`
  * Built in functions: _none_
  * Implemented by: `string`, _number types_
* `Number`
  * Operators: `+`, `-`, `*`, `/`
  * Built in functions: _none_
  * Implemented by: _number types_
* `Integer`
  * Operators: `+`, `-`, `*`, `/`, `<<`, `>>`, `^`, `&`, `|`; 
  * Built in functions: _none_
  * Implemented by: _integer types_
* `Bool` 
  * Operators: `==`, `!=`, `&&`, `||`, `!`
  * Built in functions:
  * Implemented by: _boolean types_
  * Note: `Bool` is special in that type parameters implementing `Bool` may be used in conditionals. Eg:
    ```Go
	\B Bool/ func F(b B) {
		if b {
			println "It's true"
		} else {
			println "It's false"
		}
	}
	```
* `String` 
  * Operators: `[]`, `+`, `==`, `!=`, `<`, `>`, `<=`, `>=`
  * Built in functions: `make`, `len`, `cap`
  * Implemented by: _string types_
* `Array\T/` 
  * Operators: `[]`
  * Built in functions: `len`, `cap`
  * Implemented by: _array types containing `T`_
* `Slice\T/`
  * Operators: `[]`
  * Built in functions: `make`, `append`, `len`, `cap`, `copy`
  * Implemented by: _slice types containing `T`_
* `Map\K Comparable, V/` 
  * Operators: `[]`
  * Built in functions: `make`, `len`, `delete`
  * Implemented by: _map types keyed by `K`, holding `T`_
* `RChan\T/`
  * Operators: `<-` _(read)_
  * Built in functions: `make`, `close`, `cap`
  * Implemented by: _read channels of `T`_
* `WChan\T/`
  * Operators: `<-` _(write)_
  * Built in functions: `make`, `close`, `cap`
  * Implemented by: _write channels of `T`_
* `Chan\T/`
  * Operators: `<-` _(read and write)_
  * Built in functions: `make`, `close`, `cap`
  * Implemented by: _channels of `T`_

A few clarifications:

* A concept that is implemented by a type T is also implemented by any type that has T as underlying type.  
  Eg: `type AppleCount int` implements `Integer`, `Number`, `Additive`
* `Bool` is included in the table.  That may seem a strange choice: 
  How would you write a generic type or function with a Bool parameter? Well, Go allows this:

  ```
  type MyBool bool
  ```

  which creates a type distinct from bool but succeptible to the same operations. 
  This allows generic definitions handling all such types.
  I must, however, admit I'm not able to come up with a convincing use for such types, 
  so let's say I've added `Bool` for completeness.

* I've also included `String` in the table. 
  You can derive types from string in the same way as for bool, but the usefulness of types with `string` as 
  underlying type is more obvious.


* Some of the built-in concepts have _built-in functions_ associated with them. 
  For example `Slice` defines the functions `make`, `len` and `cap`, so you can do stuff like:

  ```Go
  \T, S Slice\T/ / func F {
      var s S = make(S, 0, 0)
	  ...
  }
  ```

  This constitutes a way of hooking up generics and Concepts with the generics that is built into Go today.

## Embedding 

Just like interfaces, a concept can _embed_ an other 

```Go
concept Foo {
	...
}

concept Baat {
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

## Type methods

Generic types can have methods:

\T C1/ type Foo ...

\T C1/ func (f Foo\T/) M() {

}

The target type must have the same number of type parameters as in the definition of the type.
And each of the type parameters must be declared to implement the same concept (if any) as in the definition of the type.

Hence this is not allowed:

```Go
\T Concept1/ type Foo ...

\T Concept2/ func (f Foo\T/) M() {
}
```

Not even if Concept2 is equivalent to Concept1 or embeds Concept1. 

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

### Specialization 

In C++ template programming you may employ specialization and SFINAE to customize a generic definition for certain types. 

Template specialization relies on overloading, and would not, in my opinion, be a good fit for Go. 

SFINAE basically means that you can create several definitions of a generic type/function and the compiler will decide at 
point of instantiation which one 'works'. This is obviously at odds with the goals of this proposal.

On the other hand some kind of conditional behaviour for generic definitions could be beneficial. Consider the `Max` function:

```Go
\T Ordered/ func Max(t1,t2 T) T {
	if t1 < t2 {
		return t2
	} else {
		return t1
	}
}
```

It would be nice to extend this function to types that are not `Ordered`, but have a method `LessThan`:

```Go
\T/ concept Lessable {
	LessThan(other T) bool
}

\T Lessable\T/ / func Max(t1, t2 T) T {
	if t1.LessThan(t2) {
		return t2
	} else {
		return t1
	}
}
```

We can't have these two definitions at the same time as overloading is not allowed. 
One could consider some form of static type switch:

```Go
\T/ func Max(t1, t2 T) T {
	var t1_less_than_t2 bool
	
	switch\T/ {
	case Ordered:
		t1_less_than_t2 = t1 < t2
	case Lessable: 
		t1_less_than_t2 = t1.LessThan(t2)
	}

	if t1_less_than_t2 {
		return t2
	} else {
		return t1
	}
}
```

The idea could be:

*   The compiler (still) verifies the correctness of this code at the point of definition.
*   On instantiation apply the first switch case that matches (ie. `T` implements the concept).
*   If no case matches, the compiler issues a compile error, something like "Type must implement one of `Ordered`, `Lessable`."
*   A `default` entry is allowed, so that all types can be handled in one switch.    
    
I have not included type switch in this proposal.
While this construct works nicely for the `Max` example, 
I'm not convinced that it's usefulness is broad enough to warrant its addition to Go. 

I feel the subject is complicated enough that it should be considered separately.

TODO: Variadic templates.

### Variadic definitions

Another element left out of this proposal is _variadic definitions_. Something like:

```Go
\T...Concept/ type ?? 
```

I've put a couple of question marks here, because it is unclear, to me at least, how to write an actual definition
based on a variable number of type parameters. C++ uses recursion which in turn relies on overload, and that is not a good 
fit for Go. I think you'd have to add a number of primitives to Go including:

*	A way of getting the number of type parameters at intantiation time
*   A way of addressing the n'th type parameter 
*  	A way of packing a list of variables into a struct and a way of 'spreading' a struct to a list of variables

Overloading with respect to number of type parametes _is_ allowed so it is possible to do:

```Go
\T1/ type tuple struct { t1 T1 }
\T1, T2/ type tuple struct { t1 T1, t2 T2 }
\T1, T2, T3/ type tuple struct { t1 T1, t2 T2, t3 T3 }
```

etc. 
So if you want to do tuples, you can define them and accompanying functions/methods up to, say, 20 elements covering most usecases. 
It will involve some writing, but once done, from a usage perspective about just as useful as if it had been defined with
variadic generics. Go gen may be helpful. All in all I do not think omission of variadic definitions is a major problem.

### Combining concepts 'on the fly'

One element of Java generics I like is:

```Java 
public class C<T extends I1 & I2> {
	...
}
```

Here you combine interfaces 'on the fly', not having to define a new one extending `I1` and `I2`. 

It could be considered to add something like this to Go concepts, maybe:

```Go
\T C1:C2/ type ...
```

meaning T should implement both `C1` and `C2`. 

I have opted not to do so, since I feel that the behaviour of concepts and interfaces should be as closely aligned as possible.
In other words: If this should be added to concepts it should also be added to interfaces. That would be a separate proposal.


