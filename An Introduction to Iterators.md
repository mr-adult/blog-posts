# An Introduction to Iterators
///2023-10-25 An introduction to the iterator design pattern and its inner machinery.

If you're not already aware, an iterator is the design pattern that powers your language's flavor of non-index-based for loop. Some examples from common languages would be:

```python
# Python
list = [0, 1, 2]
for item in list:
	# 0, 1, 2,
	print(item + ", ")
```

```Java
// Java
int[] list = new int[] {0, 1, 2};
for(int item : list) {
	// 0, 1, 2,
	System.out.println(item + ", ");
}
```

```C#
// C#
int[] list = {0, 1, 2};
foreach(int item in list) {
	// 0, 1, 2,
	Console.WriteLine(item + ", ");
}
```

```rust
// Rust
let list = [0, 1, 2];
for item in list {
	println!("{}, ", item);
}
```

I think you get the point. You may have asked yourself, how does this actually work? The answer, of course, is iterators.

## So What is an Iterator?
An iterator is very simple conceptually, but is very powerful. You can think of it as a movable cursor that can only move in one direction but also points into a collection (whether real or simply calculable using a few pieces of state) of some sort. In our list example above, it would look something like this:

```
[0, 1, 2]
 |
 current value
```

Every language specifies the interface of an iterator a little bit differently, but it is composed of two pieces. The first piece is a function that moves the cursor of the iterator to its next position. The second piece is some mechanism for getting the value of that next element. Most languages combine both into one action/method (this includes Python, Java, JavaScript, Rust, and others). Some others decide to break it into two different methods/properties (C# is the only one I know of). Pretty straightforward, right? There's two more details:
1. An iterator by definition starts _just before_ the first element in the collection. In our list example, that means that the iterator would start here. It's a little bit weird, but it makes the machinery work a little nicer. 

```
{nothing} [0, 1, 2]
     |
     current value
```

2. Once it sends the signal that there are no more elements, calling the **next()** method again is outside of the iterator specification and is allowed to do anything. This behavior is _not defined_, so functions built to work with all iterators should never call **next()** again once they receive the ending signal.

The actual interfaces for iterators include the following:
- [Iterator](https://wiki.python.org/moin/Iterator) in Python
- [Iterator\<T\>](https://docs.oracle.com/javase/8/docs/api/java/util/Iterator.html) in Java
- [IEnumerator\<T\>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.getenumerator?view=net-7.0) in C#
- [IntoIterator](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html) in Rust
- [The iterable protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) in JavaScript

We could then use our example list iterator as follow. I'm going to use the Rust definition of iterators, but it should be relatively easy to translate to your preferred language.

```rust
let array = [0, 1, 2];

// start - turn the array into an iterator
// {nothing} [0, 1, 2]
//     |
//     current value
let mut iterator = array.into_iter();

// move cursor
// [0, 1, 2]
//  |
//  current value
let first = iterator.next();
println!("{:?}", first); // Some(0). Some just means a non-null value in Rust.

// move cursor
// [0, 1, 2]
//     |
// current value
let second = iterator.next();
println!("{:?}", second); // Some(1)

// move cursor
// [0, 1, 2]
//        |
// current value
let third = iterator.next();
println!("{:?}", third); // Some(2)

// move cursor
// [0, 1, 2] {nothing}
//               |
//       current value
let fourth = iterator.next();
println!("{:?}", third); // None. This is the rust equivalent of null. This indicates we have reached the end of the iterator in rust's iterator implemenation.

// Calling next() after the iterator has completed is undefined behavior!
// Each implementation can do with this whatever it would like.
let fifth = iterator.next(); // None again in the current iterator implementation.
```

You may have already figured this out, but the for-loops discussed above are really just syntactic sugar around a while loop that uses the iterator APIs. Revisiting our original example, we can restructure this code into the lower-level while loop like the compiler will during the build process (this is called desugaring because it removes the syntactic sugar). If you know Rust, you'll know that this code could be written with pattern matching, but this structure is closer to what you'd see in other languages.

```rust
// high level for loop
let list = [0, 1, 2];
for item in list {
	// 0, 1, 2,
	println!("{}, ", item);
}
```

```rust
// desugared equivalent code
let list = [0, 1, 2];
let mut iterator = list.into_iter();
let mut current = iterator.next();
while current.is_some() { // this would be != null in other languages.
	// 0, 1, 2,
	println!("{}, ", item);
	current = iterator.next();
}
```

## Use Cases
Ok, so now you understand the machinery of an iterator. You may be asking yourself what the use cases of iterators are? Aren't they something that the programming language takes care of for you so that you don't have to worry about it? Broadly speaking, yes. Most of the time you are either looping over the items of an array or linked list or some other well-defined data structure that already implements the iterator specification of your language. Several languages also provide methods for deriving iterators from other iterators. The ones I know of are:

1. Rust's [std::iter](https://doc.rust-lang.org/std/iter/index.html) crate
2. C#'s [System.Linq](https://learn.microsoft.com/en-us/dotnet/api/system.linq?view=net-7.0) library

The language-level implementation in tandem with these libraries will take care of 90% of the iterator use cases for you and you really shouldn't reinvent the wheel here. Just use the built-in machinery of your language if it is available to you. 

The places where you _do_ want to use an iterator are the following:
- You have a few pieces of state that can be used to derive a list of some other value. For example, I wrote an iterator to slice up a date range by a certain time span. Give a date range and a slice size, I would calculate a series of smaller ranges. (ex. Given 3/2/2021-12/4/2023 and a time span of 1 month, I would generate 3/2/2021-3/31/2021, 4/1/2021-4/30/2021, ..., 11/1/2023-11/30/2023, 12/1/2023-12/4/2023). This approach lets you calculate that state on-the-fly and avoid synchronization issues between the two pieces of state.
- You have a well-defined algorithm that you would like to build a generic implementation for. This helps avoid duplicate code to perform that algorithm on different types of data while also allowing a lot of flexibility. A good example of this is the library I wrote: [tree_iterators_rs](https://crates.io/crates/tree_iterators_rs). Because tree traversal algorithms are rigorously defined, I could write a generic implementation so that others don't need to continuously repeat the same code.

There are probably other use cases out there, but these are the two I've found.

## Implementation
The implementation of an iterator tends to be pretty simple. I'm going to appeal back to your introduction to recursion class and show you how to implement the Fibonacci sequence using an iterator. To start with, we need to define a struct that holds onto the current and next numbers in the sequence.

```rust
struct FibonacciIterator {
    current: i32, // i32 is just a 32-bit integer (int in most languages).
    lookahead: i32,
}
```

We'll also set up an initialization method to get us started with the traditional 0 and 1 values.

```rust
impl FibonacciIterator {
    fn new() -> FibonacciIterator {
        FibonacciIterator { 
            current: 0, 
            lookahead: 1, 
        }
    }
}
```

And now the iterator implementation. It's pretty straightforward. We just need to do the following few steps:

```rust
impl Iterator for FibonacciIterator {
    type Item = i32; // this defines the type of value we will return from our iterator.

    fn next(&mut self) -> Option<i32> {
		// 1. Generate the next number by summing current and lookahead.
        let next = self.current + self.lookahead; 
                
        // 2. Shift all of the values back 1 position
        let return_val = self.current;
        self.current = self.lookahead;
        self.lookahead = next;
        
        // 3. Return the appropriate value.
        return Some(return_val);
    }
}
```

And that's it! Rust gives us a lot of nice methods to work with iterators, so now we can do things like this:

```rust
// take the first 11 numbers from the sequence and put them into a list
let first_11: Vec<i32> = FibonacciIterator::new()
	.take(11)
	.collect();
// result: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
println!("{:?}", first_11);

// Get the 6th (5 in a 0-indexed system) item in the sequence
let nth = FibonacciIterator::new()
	.nth(5)
	.unwrap();
// result: 5
println!("{}", nth);
```

You may have also noticed that this particular iterator implementation always returns a Some() or non-null value. This is also allowed in the iterator spec, but you have to be careful. For example, the following code would lead to an infinite loop!

```rust
for fib in FibonacciIterator::new() {
	// Do something
}

// We never get here!
```

## Iterables
I would be remiss if I didn't also mention iterables. Iterables are just any collection of values that can provide multiple iterators at once. A list is an excellent example. Iterables are even simpler than iterators. The only machinery they need to have is some sort of "get iterator" method that returns an iterator. This would be:
- [\_\_iter\_\_()](https://wiki.python.org/moin/Iterator) in Python
- [iterator()](https://docs.oracle.com/javase/8/docs/api/java/lang/Iterable.html#iterator--) in Java
- [GetEnumerator()](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.getenumerator?view=net-7.0) in C#
- [into_iter()](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html) in Rust
- [\[Symbol.iterator\]()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/iterator) in JavaScript

In most languages, the actual for loop implementation relies on an iterable, not an iterator. Iterators are consumable, as in they can only be used once. They are not expected to work multiple times. In our example, once I ask for the 3rd Fibonacci number, I cannot ask for it again. An iterable, on the other hand, can give you as many iterators as you would like. I could loop over the same list 1000 times at the same time if I wanted.

## Lazy Iteration
One detail that we glossed over is that iterators use a mechanism called _lazy iteration_. This means they don't do any extra work than what they have to to get you the next value. This allows the for loops to take advantage of short-circuiting logic to exit loops and skip any unnecessary computations. This is also how _continue_ and _break_ keywords are powered, but it also has some caveats. In languages that do not have strict ownership rules like Rust (and even in Rust sometimes), you might get some weird behaviors. Let's take a simple example from C#:

```C#
int[] list = {0, 1};
bool result = false;
IEnumerable<int> derived = list.Select(value => {
	result = !result;
	return value;
});

foreach(int _ in derived) { /* Do nothing */ }
// result: true
Console.WriteLine(result);

foreach(int _ in derived) { /* Do nothing */ }
//result: false
Console.WriteLine(result);
```

Huh? What just happened? The answer is that lazy iteration got us! The .Select() method uses _deferred iteration_. Basically, it doesn't actually call it's closure until the **next()** (or MoveNext() in C#) method is called. We've introduced a side effect into our iterator using that closure. Every time our _derived_ iterable creates an iterator and that iterator has its **next()** method called, we end up flipping the value of result from true to false or false to true. This can cause some very sly bugs to creep into your code. For clarity about what the actual code mechanics are, let's unroll our loops and dissect what is actually happening under the hood.

```C#
int[] list = {0, 1};
bool result = false;
IEnumerable<int> derived = list.Select(value => {
	result = !result;
	return value;
});

// There are 2 elements in our list, so foreach unrolls to 3 MoveNext() calls in C# (2 to get the values, 1 that sends the end signal)
// foreach(int _ in derived) { /* Do nothing */ }
IEnumerator<int> iterator = derived.GetEnumerator();
iterator.MoveNext(); // closure called, result now = true
iterator.MoveNext(); // closure called, result now = false
iterator.MoveNext(); // closure called, result now = true
// result: true
Console.WriteLine(result);

// Again, there are 2 elements in our list, so foreach unrolls to 3 MoveNext() calls in C# (2 to get the values, 1 that sends the end signal)
// foreach(int _ in derived) { /* Do nothing */ }
IEnumerator<int> iterator = derived.GetEnumerator();
iterator.MoveNext(); // closure called, result now = false
iterator.MoveNext(); // closure called, result now = true
iterator.MoveNext(); // closure called, result now = false

//result: false
Console.WriteLine(result);
```

As a best practice, iterators should not introduce side effects. This helps you keep unexpected state changes out of your for loops and prevents any sly bugs from sliding into your programs. Side effects are better avoided altogether.

That about sums up iterators. They are very useful for providing a standard interface that libraries can build on top of and allow all kinds of programs to interface with each other. Some of the most useful libraries in this space are those that allow for multithreading and smart work balancing using the iterator interface. Next time you find yourself generating a list of values from a few pieces of data, consider an iterator.