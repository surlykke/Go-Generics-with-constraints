_Note: This project serves as a placeholder.
Once it is done I hope to have it added to [Go Proposals](https://github.com/golang/proposal).
If that happens this project will be deleted_

# Generics with concepts for Go

This is a proposal for adding generics with concepts to the Go language. 
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

So I propose generics for Go with _concepts_:

Generics means that types and functions may be _parametrized_, ie. depend on type parameters. 
They may be instantiated with _concrete types_ substituted for the type parameters.

Concepts are used to associate a _contract_ with a type parameter in much the same way that 
an interface can be used to impose a contract on an ordirnay parameter.
Like interfaces, concepts define a _method set_, but a concept may also define a set of _fields_ 
and/or a set of _operators_.

What you can do with a type parameter in a generic definition is that which it's associated concept defines. 
You may call methods on the type parameter, access fields on it or apply operators to it, only if it is associated 
with a concept that has those methods, fields or operators.

The aim is that the correctness of a generic definition can be verified at the point of _definition_:  
When instantiating a generic type, the compiler only needs to verify that the concrete types substituted for the type parameters 
implements the given concepts, and then the validity of the instantiation follows.

I believe that with this, the compiler can be made simpler, the code will be more readable and that it will allow for better tooling.

## Generics

First, let's lay out generics looks.

Type- and function definitions at package level may be generic. That means they may depend on type parameters. An example:

```Go
\T/ type List struct {
	value T
	next *List\T/
}

\T/ func Push(l *List\T/, t T) *List\T/{
	return &List\T/{t, l}
}
```

Here, we do not call any methods on `value`, access any fields, or apply any operators on `T`. As we haven't said anything about `T`,
we can't make any assumptions about what can be done with it.

We can use our generic definition like this:

```Go
var myList *List\int/ = nil

myList = Push\int/(&myList, 2)
myList = Push\int/(&myList, 4)

for l := &myList; l != null; l = l.next {
	fmt.Println(l.value)
}
```

### A note about the syntax

As you can see the syntax I use for generic differs somewhat from what Taylor has proposed.

`\` and `/` are used as delimiters. I prefer not to overload  `[]` or `()` with a new usage, and I prefer
`MyFunc\Foo/(f)` to `MyFunc(Foo)(f)`.

Also, I prefer to have the type parameter list _before_ keywords `type` and `func` in declarations. 
This, I feel, makes generic definition 'stand out' more.

To my knowledge, there is no other legal use of backslash in Go outside strings and runes. 
Hence there is no need for a keyword, like `template` or `gen`, to mark the start of a generic declaration, 
Whenever the compiler sees `\` where a package level may appear, it'll know that a generic declaration is coming.

The main drawback is that if you nest: `Foo\Baa\T//` the last pair of forward slashes will be interpreted as starting a line comment. 
You'll have to insert a space: `Foo\Baa\T/ /`, or perhaps slightly nicer: `Foo\ Baa\T/ /`.

This is reminiscent of the `>>` issue in C++ template programming (so you could view it as a (weird) homage to C++). 
In my mind it's not a huge problem. 

An other note: I'm not including type deduction in this proposal. 
I prefer to state explicitly the value of type parameters, and I prefer to think of `Push\int/(l, 2)` 
as a call to a _distinct_ function with the _name_ `Push\int/` rather than a call to a function `Push` with a type parameter `int`.

I would like to point out, though, that neither the choice of delimiters nor the absence of type deduction are essential to this proposal. 
It could easily be rewritten to use another pair of delimiters and type deduction could be added. 
Both at the cost of some increase in compiler complexity.

## Declaring concepts

A concept declaration allows explicit definition of methods, fields and/or other concepts that the concept extends.

```EBNF
ConceptDecl = 'concept' '{' { ConceptMemberSpec } '}' .
ConceptMemberSpec = ConceptMethodSpec | ConceptFieldSpec | ConceptName .
ConceptMethodSpec = identifier Signature .
ConceptFieldSpec = identifier TypeName .
```


An example of a concept with a method:

```Go
\T .Log()/ func DoLog(t T) {
	T.Log() // Allowed because T is required to have method Log()
}
```

An example of a concept with a field:

```Go
\Entity .Id long/ func Equal(e1, e2 Entity) bool {
	return e1.Id == e2.Id
}
```
To implement this concept a concrete type must have a field named `Id` (which obviously means it's a `struct`)

We can use `Entity` in a generic function declaration:

```Go
\T Entity/ func ShowId(t T) { 
	fmt.Println("Id is: ", t.Id)
}
```

We can define a type that implements `Entity`:

```Go
type Employee struct {
	Id uint64
	Name string
}

```

and use it:

```Go
var employee = Employee{1, "Chr. Surlykke"}

ShowId\Employee/(employee)
```

An example of a generic concept:

```Go
\T/ concept Cloneable {
	Clone() T
}
```

which we can use in a generic function:

```Go
\T Cloneable\T/ / func copy(t T) {
	return t.CLone()
}
```

So with a type implementing `Cloneable`: 

```Go
type Person struct {
	Name string
}

(p Person) func Clone() Person {
	return Person{p.Name}
}

```

one can do:

```Go
var person1 = Person{"John Doe"}

var person2 = copy\Person/(person1)
```

### Operators and built-in functions

There will be no syntax to _explicitly_ associate operators and built-in methods on concepts. 
Instead we will define a set of _built in_ concepts that have operators and standard functions associated with them.

One example of a built in concept is `Ordered`. 

`Ordered` defines the comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=` and is implemented by 
string-, floating-point- and integer types.

`Ordered` may be used in a generic function like this:

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
  * Implemented by: 
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
* `Ptr\T/`
  * Operators: `*`
  * Built in functions: _none_
  * Implemented by: pointer types pointing to `T` 


A few clarifications:
 
* A concept that is implemented by a type T is also implemented by any type that has T as underlying type.  
  Eg: `type AppleCount int` implements `Integer`, `Number`, `Additive`
* Some of the built-in concepts have _built-in functions_ associated with them. 
  For example `Slice` defines the functions `make`, `len` and `cap`, so you can do stuff like:

```Go
  \T, S Slice\T/ / func F {
      var s S = make(S, 0, 0)
	  ...
}
```

  This constitutes a way of hooking up generics and Concepts with the generics that is built into Go today.

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
\T .LessThan(T) bool/ func Max(t1, t2 T) T {
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


