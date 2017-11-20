_Note: This project serves as a placeholder.
Once it is done I hope to have it added to [Go Proposals](https://github.com/golang/proposal).
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
Taylor introduces the idea of _constraints_: Concrete instantions of type parameter could be  required to implement an _interface_. 
It would look like
```Go
gen [T Stringer] type PrintableVector []T
```
and would mean that `PrintableVector` could only be instantiated with types `T` that implement the `Stringer` interface. 

In later proposals Taylor abandons this approach in favor of having the compiler _extract_ constraints from a generic definition.

What I want to do here is develop the idea constraints fully. 

So I propose generics for Go with constraints:

Generics means that types and functions may be _parametrized_, ie. depend on type parameters. 
There may be _constraints_ on a type parameter. The effect of that is twofold: 
It determines what _concrete types_ can be substituted for the parameter and it determines _what you can do_ with the parameter 
in the generic definition.  


The aim is that the correctness of a generic definition can be verified at the point of _definition_:  
When instantiating a generic type, the compiler only needs to verify that the concrete types substituted for the type parameters 
fulfill the constraints given, and then the validity of the instantiation follows.

I believe that with this, the compiler can be made simpler, the code will be more readable and that it will allow for better tooling.

## Generics

Type- and function definitions at the top level may be generic. That means they may depend on type parameters. An example:

```Go
\T/ type List struct {
	value T
	next *List\T/
}

\T/ func Push(l *List\T/, t T) *List\T/{
	return &List\T/{t, l}
}
```

Here, we do not call any methods on `value`, access any fields, or apply any operators on it. As we haven't said anything about `T`,
we can't make any assumptions about what can be done with it.

We can use our generic definition like this:

```Go
var myList List\int/

Push\int/(&myList, 2)
Push\int/(&myList, 4)

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

## Constraints

We will allow type parameters in generic definitions to be associated with _constraints_ on what concrete types can be substituted for them.

There will be 3 kinds of constraints. 

1. Method: Concrete types are required to have a method.
1. Field: Concrete types are required to have a field.
1. Type: Concrete types are required to have a one of a list of types as its underlying type.

Constraints may be combined by conjunction or disjunction.

### Method constraints

A method requirement has this form:

```EBNF
MethodConstraint = '.' identifier Signature .
```
where [identifier](https://golang.org/ref/spec#identifier) and [Signature](https://golang.org/ref/spec#Signature) 
are defined as in the [The Go Programming Language Specification](https://golang.org/ref/spec)

We associate a type parameter with a constraint by following it by a constraint in the type parameter list:

```Go
\T .Log()/ func DoLog(t T) {
	T.Log() // Allowed because T is required to have method Log()
}
```

Now `DoLog` can only be instantiated with types that have a method `Log()`.

### Field constraints

The form of a field constraint is:

```EBNF
FieldConstraint = "." identifier Type.
```

Which could be used like:

```Go
\Entity .Id long/ func Equal(e1, e2 Entity) bool {
	return e1.Id == e2.Id
}
```

### Type constraints

The form of a type constraint is:

```EBNF
TypeConstraint = SingeTypeConstraint { '||' SingleTypeConstraint }
SingleTypeConstraint = ":" (BasicTypeName | TypeLit) .
BasicTypeName = identifier .
```

An example:

```Go
\T :int/ Increment(t T) T {
	return t + 1  // It is known that the underlying type of T is int, so a cast from
	              // literal '1' to T is possible and the operator '+' can be applied
				  // yielding a result of type T
}
```

Constraining to one of more types:

```Go
\T :uint8 || :uint16 || :uint32 || :uint64/ ...
```

### Combining constraints

Constraints may be combined by _conjunction_. 

The conjunction of two constraints `C1` and `C2` is written:
```Go
C1 && C2
```
It is fulfilled by any type that fulfills _both_ `C1` and `C2`.

### Named constraints

_Named_ constraints may be declared at package level. It's done with a new keyword, `constraint`.

A couple of examples:
```Go
constraint Loggable :Log()

constraint Integer :uint8 || :uint16 || :uint32 || :uint64 || 
                :int8 || :int16 || :int32 || :int64 || :uint || :int || :intptr
```

Named constraints may be used just like other constraints. For example:

```Go
\T Loggable/ func DoLog(t T) {
	t.Log()
}
```
or
```Go
\I Integer/ func MultiplyBy2(i I) I {
	return i << 1
}
```

The full syntax for constraints:
```EBNF
Constraint = SingleConstraint { '&&' SingleConstraint } .
SingleConstraint = ( ConstraintName | MethodConstraint | FieldConstraint | TypeConstraint ) .
ConstraintName = identifier .
```
And declaration of named constraints:
```EBNF
NamedConstraintDecl = 'constraint' identifier Constraint . 
```


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
\T/ constraint Lessable {
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
*   On instantiation apply the first switch case that matches (ie. `T` implements the constraint).
*   If no case matches, the compiler issues a compile error, something like "Type must implement one of `Ordered`, `Lessable`."
*   A `default` entry is allowed, so that all types can be handled in one switch.    
    
I have not included type switch in this proposal.
While this construct works nicely for the `Max` example, 
I'm not convinced that it's usefulness is broad enough to warrant its addition to Go. 

I feel the subject is complicated enough that it should be considered separately.




## Overloading generic definitions

Overloading of generic definitions with the _same_ number of type parameters is _not_ allowed: 

```Go
\T Constraint1/ type Foo ... 
                          
\T Constraint2/ type Foo ... // Not allowed
```

The problem would be that if `Baa` satisfies both `Constraint1` and `Constraint2`, then `Foo\Baa/` would be ambigous with
respect to which definition to employ.

Overloading with _different_ number of generic parameters _is_ allowed.
So:
```Go
\S Constraint1/ type Foo ...
\T Constraint2, U Constraint3/ type Foo ...
```
is ok.

## Type methods

Generic types can have methods:

\T C1/ type Foo ...

\T C1/ func (f Foo\T/) M() {

}

The target type must have the same number of type parameters as in the definition of the type.
And each of the type parameters must be declared to satisfy the same constraint (if any) as in the definition of the type.

Hence this is not allowed:

```Go
\T Constraint1/ type Foo ...

\T Constraint2/ func (f Foo\T/) M() {
}
```

when `Constraint1` is different from `Constraint2`

## Runtime

Concrete instantions of generic types and functions should look like ordinary hand coded ones. Given:


```Go
\Foo C/ MyType ...

type Baa ... // Satisfies C
```

you can cast:

```Go
.(MyType\Baa/)
```

The instantiation `MyType\Baa/` is a concrete type just like any other (non-generic) type in Go. 
In fact, from a runtime perspective, the only thing that gives away it's origin in a generic definition is
the appearence of `\` and `/` in it's name.


## Further considerations

Whats written above constitutes my proposal for generics with constraints in Go. 
Here I'll mention some variations and additions that could be considered.

TODO: Variadic templates.

### Variadic definitions

Another element left out of this proposal is _variadic definitions_. Something like:

```Go
\T...Constraint/ type ?? 
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

