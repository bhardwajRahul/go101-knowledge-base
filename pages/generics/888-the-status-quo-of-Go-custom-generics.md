
# The Status Quo of Go Custom Generics

The previous chapters explain the basic knowledge about Go custom generics.
This chapter will list some missing features in the current design and
implementation of Go custom generics.

{#embed-type-parameter}
## Embedding type parameters is not allowed now

Due to design and implementation complexities, currently (Go 1.25), type parameters are
disallowed to be embedded in either interface types or struct types.

For example, the following type declaration is illegal.

```Go
type Derived[Base any] struct {
	Base // error
	
	x bool
}
```

Please view [this issue](https://github.com/golang/go/issues/43621) for reasons.

<!--
https://github.com/golang/go/issues/49030
https://github.com/golang/go/issues/24062
-->

## The method set of a constraint is not calculated completely for some cases

The Go specification states:

> The method set of an interface type is the intersection of the method sets of each type in the interface's type set.

However, currently (Go toolchain 1.25), only the methods explicitly specified in interface types are calculated into method sets.
For example, in the following code, the method set of the constraint should contain both `Foo` and `Bar`,
and the code should compile okay, but it doesn't (as of Go toolchain 1.25).

```Go
package main

type S struct{}

func (S) Bar() {}

type C interface {
	S
	Foo()
}

func foobar[T C](v T) {
	v.Foo() // okay
	v.Bar() // v.Bar undefined
}

func main() {}
```

This restriction is planned [to be removed in future Go toolchain versions](https://github.com/golang/go/issues/51183).

## No ways to specific a field set for a constraint

We know that an interface type may specify a method set.
But up to now (Go 1.25), it could not specify a (struct) field set.

There is a proposal for this: https://github.com/golang/go/issues/51259.

The restriction might be lifted from future Go versions.

## Fields of values of type parameters are not accessible

Currently (Go 1.25), even if all types in the type set of a constraint
are structs and they share some common fields, the common fields still
can't be used.

For example, the generic functions in the following example all fail to compile.

```Go
package main

type S1 struct {
	X int
}

type S2 struct {
	X int `json:X`
}

type S3 struct {
	X int
	Z bool
}

type S4 struct {
	S1
}

func F12[T S1 | S2](v T) {
	_ = v.X // error: v.x undefined
}

func F13[T S1 | S3](v T) {
	_ = v.X // error: v.x undefined
}

func F14[T S1 | S4](v T) {
	_ = v.X // error: v.x undefined
}

func main() {}
```

A temporary (quite verbose) workaround is to specify/declare some getter and setter methods
for the involved constraints and concrete types.

For a special case, the following code also doesn't compile.

```Go
type S struct{x, y, z int}

func mod[T S](v T) {
	_ = v.x // error: v.x undefined
}
```

The restriction (described in the current section) was [added just before
Go 1.18 is released](https://github.com/golang/go/issues/51576).
It might be removed since a future Go version.

<!--
https://github.com/golang/go/issues/48522
https://github.com/golang/go/issues/50417
https://github.com/golang/go/issues/50233
-->

## Type switches on values of type parameters are not supported now

It has been mentioned that [a type parameter is an interface type from semantic view](555-type-constraints-and-parameters.md#type-parameters-are-interfaces).
On the other hand, a type parameter has wave-particle duality.
For some situations, it acts as the types in its type set.

Up to now (Go 1.25), values of type parameters may not be asserted.
The following two functions both fail to compile.

```Go
func tab[T any](x T) {
	if n, ok := x.(int); ok { // error
		_ = n
	}
}

func kol[T any]() {
	var x T
	switch x.(type) { // error
	case int:
	case []bool:
	default:
	}
}
```

The following modified versions of the above two functions compile okay:

```Go
func tab2[T any](x T) {
	if n, ok := any(x).(int); ok { // error
		_ = n
	}
}

func kol2[T any]() {
	var x T
	switch any(x).(type) { // error
	case int:
	case []bool:
	default:
	}
}
```

There is a proposal to use type switches directly on type parameters, like:

```Go
func kol3[T any]() {
	switch T {
	case int:
	case []bool:
	default:
	}
}
```

Please subscribe [this issue](https://github.com/golang/go/issues/45380) to
follow the progress of this problem.

## Generic methods are not supported

Currently (Go 1.25), for design and implementation difficulties,
generic methods (not methods of generic types) are
[not supported](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#methods-may-not-take-additional-type-arguments).

For example, the following code are illegal.

```Go
import "sync"

type Lock struct {
	mu sync.Mutex
}

func (l *Lock) Inc[T ~uint32 | ~uint64](x *T) {
	l.Lock()
	defer l.Unlock()
	*x++
}
```

How many concrete methods do the `Lock` type have?
Infinite! Because there are infinite uint32 and uint64 types.
This brings much difficulties to make the `reflect` standard package keep backwards compatibility.

There is [an issue](https://github.com/golang/go/issues/49085) for this.

## There are no ways to construct a constraint which allows assignments involving types of unspecific underlying types

And there are not such predeclared constraints like the following supposed `assignableTo` and `assignableFrom` constraints.

```Go
// This function doesn't compile.
func yex[Tx assignableTo[Ty], Ty assignableFrom[Tx]](x Tx, y Ty) {
	y = x
}
```

## There are no ways to construct a constraint which allows conversion involving types of unspecific underlying types

And there are [not such predeclared constraints](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#no-way-to-express-convertibility) like the following supposed `convertibleTo` and `convertibleFrom` constraints.

```Go
// This function doesn't compile.
func x2y[Tx convertibleTo[Ty], Ty convertibleFrom[Tx],
		// The second value argument is
		// for type inference purpose.
		](xs []Tx, _ Ty) []Ty {
	if xs == nil {
		return nil
	}
	ys := make([]Ty, len(xs))
	for i := range xs {
		ys[i] = Ty(xs[i])
	}
	return ys
}

var bs = []byte{61, 62, 63, 64, 65, 66}
var ss = x2y(bs, "")
var is = x2y(bs, 0)
var fs = x2y(bs, .0)
```

Currently, there is an ungraceful workaround implementation:

```Go
func x2y[Tx any, Ty any](xs []Tx, f func(Tx) Ty) []Ty {
	if xs == nil {
		return nil
	}
	ys := make([]Ty, len(xs))
	for i := range xs {
		ys[i] = f(xs[i])
	}
	return ys
}

var bs = []byte{61, 62, 63, 64, 65, 66}
var ss = x2y(bs, func(x byte) string {
	return string(x)
})
var is = x2y(bs, func(x byte) int {
	return int(x)
})
var fs = x2y(bs, func(x byte) float64 {
	return float64(x)
})
```

The workaround needs a callback function, which
makes the code verbose and much less efficient,
though I do admit it has more usage scenarios.







<!--
## No ways to construct a constraint which is only satisfied by interface types pr by non-interface types.

The current constraint design lacks of two abilities:
1. The ability of specifying a type argument must be an interface type.
2. The ability of specifying a type argument must not be an interface type.

https://groups.google.com/g/golang-nuts/c/EL6A2jFa92k
-->
