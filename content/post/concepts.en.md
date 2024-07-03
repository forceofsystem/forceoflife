---
title: 'Introduction to C++20: Concepts'
date: 2024-07-03T09:13:27+08:00
draft: false
comments: true
toc: true
tags:
  - C++
  - Computer
---

<!--more-->

## Concepts

Sometimes, we want to use template to meet multiple requirements, like this:

```c++
template<typename Seq, typename value>
Value sum(Seq s, Value v)
{
	for (const auto &x : s)
		v += x;
	return v;
}
```

The `sum()` requires that:

- A sequence, `Seq`, that supports `begin()` and `end()` so that the range-for will work.
- An arithmetic type, `Value`, that supports `+=` so that elements of the sequence can be added.

We call such requirements `concepts`.

### Use of Concepts

We can use concepts to restrict `sum()`'s paraeters better than using `typename`, like this:

```c++
template<Sequence Seq, Number Num>
Num sum(Seq s, Num v)
{
	for (const auto &x : s)
		v += x;
	return v;
}
```

If you want to compile this code, you will receive a error, because we don't define `Sequence` and `Number`. A concept is a compile-time predicate specifying how one or more types can be used. But let we skip the definition of `Sequence` and `Number`.

You may think you have fully met `sum()`'s' requirements. However, consider this line:

```c++
v += x;
```

This line implies that elements `x` from `s` can be added into v, i.e., we should be able to add elements of a `Sequence` to a `Number`.We can do that:

```c++
template<Sequence Seq, Number Num>
	requires Arithmetic<std::ranges::range_value_t<Seq>, Num>
Num sum(Seq s, Num v) {
	for (const auto x : s) {
		v += x;
	}
	return v;
}
```

This saves us from accidentally tring to calculate the `sum()` of a `vector<string>` or a `vector<int*>` while still accepting `vector<int>` and `vector<complex<double>>`. You may think this code which makes no sense, however, there may be some inplicit type conversion, which allow the elements in `Sequence` can be added into `Num` that we don't want it to be. With `Arithmetic`, we can restrict these behaviours more precisely. We will also define `Arithmetic` later, don't worry.

Anyway, even if you insist that `Arithmetic` is useless, I still need to introduce a new definition about concepts to you. Consider the keyword `requires`, it's called a **requirements-clause**. The `template<Sequence Seq>` notation is a shorthand for an explicit use of `requries Sequence<Seq>`. If you liked verbosity, you could equivalently have written

```c++
template<typename Seq, typename Num>
	requires Sequence<Seq> && Number<Num> && Arithmetic<range_value_t<Seq>, Num>
Num sum(Seq s, Num n)
{
	//...
}
```

but I don't like it.

There is another notation we can use:

```c++
template<Sequence Seq, Arithmetic<range_value_t<Seq>> Num>
Num sum(Seq s, Num n);
```

Furtheremore, someone may have noticed the definition of `Arithmetic`, but before talking the definition, we are supposed to learn more about concepts's usage.

### Concept-based Overloading

We can overload concepts based on their properties, much as we do for functions. Consider a slightly simplified standard-library function `advance()` that advances an iterator:

```c++
template<forward_iterator Iter>
void advance(Iter p, int n)
{
	while (n--)
		++p;
}

template<random_access_iterator Iter>
void advance(Iter p, int n)
{
  p += n;
}
```

Like other overloading, this is a compile-time mechanism implying no run-time cost, and where the compiler doesn't find a best choice, it gives an ambiguity error.

### Valid Code

The question of whether a set of template arguments offers what a template requires of its template parameters ultimately boils down to whether **some expressions are valid**.

Using a **requires-expression**, we can check if a set of expressions is valid. We can write `advance()` without the use of the standard-library concept `random_access_iterator`:

```c++
template<forward_iterator Iter>
	requires requries(Iter p, int i) { p[i]; p + i;}
void advance(Iter p, int n)
{
	p += n;
}
```

The first `requires` starts the requirements-clause and the second one starts the **requires-expression**. A **requires-expression** is a predicate that is true if the statements in it are valid code and false if not.

**If you see requires requires in your code, it is probably too low level and will eventually become a problem.** You can consider require-expression the assembly code of generic programming, much like how assembly code is to ordinary programming.

### Definition of Concepts

Now, we can start to learn how to define a concept. Before this, you need to know it's easier to use a concept from a good library than to write a new one.

```c++
template<typename T>
concept Equality_comparable =
	requires (T a, T b) {
		{ a == b } -> Boolean;
		{ a != b } -> Boolean;
	};
```

Tha value of a concept is always **bool**. The result of an `{...}` specified after a `->` must be a concept. We can slightly modify the `Equality_comparable` to handle nonhomogenous comparisons:

```c++
template<typename T, typename T2 = T> // default template argument
concept Equality_comparable =
	requires (T a, T2 b) {
		{ a == b } -> Boolean;
		{ a != b } -> Boolean;
		{ b == a } -> Boolean;
		{ b != a } -> Boolean;
	};
```

Ultimately, we can define a concept that requires arithmetic to be valid between numbers.

```c++
template<typename S>
concept Sequence = requires(S a) {
	typename std::ranges::range_value_t<S>; // it must provide a value type, and
	typename std::ranges::range_reference_t<S>; // an iterator type.

  // it must be a container
	{ a.begin() } -> std::same_as<std::ranges::iterator_t<S>>;
	{ a.end() } -> std::same_as<std::ranges::iterator_t<S>>;

  // its iterator type must be at least an input_iterator
	requires std::input_iterator<std::ranges::iterator_t<S>>;
	requires std::same_as<std::ranges::range_value_t<S>,
  	std::iter_value_t<std::ranges::iterator_t<S>>>;
};

template<typename T, typename U=T>
concept Number =
requires(T x, U y) {
	x+y; x-y; x*y; x/y;
	x+=y; x-=y; x*=y; x/=y;
	x=x;
	x=0;
};

template<typename T, typename U>
concept Arithmetic = Number<T, U> && Number<U, T>;

template<Sequence Seq, Number Num>
	requires Arithmetic<std::ranges::range_value_t<Seq>, Num>
Num sum(Seq s, Num v) {
	for (const auto x : s) {
		v += x;
	}
	return v;
}
```

Last but not least, the concepts specfied for a template are used to check arguments at the point of use of the template. They are not used to check the use of the parameters in the definition of the template. For example:

```c++
template<equality_comparable T>
bool cmp(T a, T b)
{
	return a < b;
}

bool b0 = cmp(cout, cerr); // error: ostream doesn't support ==
bool b1 = cmp(2, 3); // ok
bool b2 = cmp(2+3i, 3+4i); // error: complex<double> doesn't support <
```

The check of concepts catches the attempt to pass the `ostream`s, but accepts the `int`s and `complex<double>`s because those two types support `==`, though `complex<double>` doesn't support `<`.

Delaying the final check of the template definition until instantiation time gives two benefits:

- We can use incomplete concepts during development.
- We can insert debug, tracing telemetry, etc. code into a template without affecting its interface.

Given concepts, we can strengthen requirements of all such initializations by preceding `auto` by a concept:

```c++
auto twice(Arithmetic auto x) { return x + x; }
auto thrice(auto x) { return x + x + x; }

auto x1 = twice(7); // ok
string s = "hello";
auto x2 = twice(s); // error: a string doesn't meet the Arithmetic's requirements
auto x3 = thrice(s); // ok, x3 = "hellohellohello"
```

Concepts can also constrain the initialization of variables:

```c++
auto ch1 = open_channel("foo"); // works with whatever open_channel() returns
Arithmetic auto ch2 = open_channel("foo"); // error: a channel is not Arithmetic
Channel auto ch3 = open_channel("foo"); // ok: assuming Channel is an appropriate concept and that open_channel() returns one
```

This is a great way to counter overuse of `auto` and to document requirements on code using generic functions. It can also be used to constrain a return type or define a local variable:

```c++
Number auto some_function(int x)
{
	// ...
	return fct(x);
	// ...
}

// a bit verbose
auto some_function(int x)
{
  // ...
  Number auto y = fct(x);
  return y;
  // ...
}
```

### Concepts and Types

Re-consider the last code frament, you may elaborate that we can use type to constrain functions and local variables instead the combination of concepts and `auto`.

We need to compare type and concept first:

- A type:
  - Specifies the set of operations that can be applied to an object, implicitly and explicitly.
  - Relies on function declarations and language rules.
  - Specifies how an object is laid out in memory.
- A single-argument concept:
  - Same as the type.
  - Relies on use patterns reflecting function declarations and language rules.
  - Says **nothing** about the layout of the object.
  - Enables the use of a set of types.

Thus, constrain code with concepts gives more flexibility than constraining with types, and concepts can define the relationship among several arguments.

> In _A Tour of C++_, Bjarne Stroustrup advocates defining most functions as template functions with their arguments constrained by concepts. However, we must use a concept as an adjective rather than a noun, like this:
>
> ```c++
> void sort(Sortable auto &);
> void sort(Sortable &); // not this
> ```
>
> But in my opinion, the future of C++ will be like this:
>
> ```c++
> auto;
> ```
>
> QAQ.
