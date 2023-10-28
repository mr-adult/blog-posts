///A deeper dive into the iterator design pattern and how it is effectively leveraged and built on top of to create powerful libraries.

In my previous post, I went over an introduction to the iterator design pattern. If you haven't read that yet, you can find it [here](https://adamfortune.com/blog/AnIntroductiontoIterators).

Today I would like to dive deeper into how libraries are able to effectively leverage the consistent interface of iterators. I'll start by discussing the System.Linq library in C# (this is very similar to the std::iter crate in Rust). Afterwards, I will discuss the [Rayon](https://crates.io/crates/rayon) crate (which is very similar to C#'s [Parallel.ForEach](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)) and how it is able to effectively task-share across multiple threads using iterators.
## Derived Iterators (System.Linq)
Because all iterators provide a consistent interface, we can create additional derived iterators that are able to build on top of any iterator implementations (including themselves) and reduce the boilerplate of re-implementing the same modifications constantly. Some examples are filter, map, and reduce. I will walk you through how the filter method is derived for iterators. When we implement these derived iterators for other iterators, we need to dive into the world of generic types. This allows us to create a blanket implementation for all iterators that other implementations get for free. Because of the high complexity of Rust syntax when using generics, I'll be writing my code in C#.

Firstly, we'll create an [extension method](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods) to let us use this function like a method on any and all blanket iterable implementation. This is equivalent to [.Where() in System.Linq](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.where?view=net-7.0). It will need to take both the source iterable and a predicate function as arguments.

```C#
using System;
using System.Collections;
using System.Collections.Generic;

public static class IterableExtensions 
{
	// This extension method simply gives us the nice syntax of calling .Filter() on our iterable.
	public static IEnumerable<T> Filter<T>(this IEnumerable<T> source, Predicate<T> predicate) 
	{
		return new FilterEnumerable(source, predicate);
	}
}
```

We can then implement the iterable as follows. It's mostly just boilerplate to be able to give multiple iterators out at the same time:

```C#
public class FilterEnumerable<T> : IEnumerable<T> 
{
	IEnumerable<T> Source;
	Predicate<T> Predicate;
	
	public FilterEnumerable(IEnumerable<T> source, Predicate<T> predicate) 
	{
		this.Source = source;
		this.Predicate = predicate;
	}

	// This is a bit unfortunate, but C# has a base interface that yields items of type object, so we also have to explicitly implement that.
	IEnumerator IEnumerable.GetEnumerator() { return this.GetEnumerator(); }
	public IEnumerator<T> GetEnumerator() {
		return new FilterEnumerator<T>(this.Source.GetEnumerator(), this.Predicate);
	}
}
```

And finally, we can implement the iterator.

```C#
public class FilterEnumerator<T> : IEnumerator<T> 
{
	IEnumerator<T> Source;
	Predicate<T> Predicate;

	// Again, C# has a base interface that yields items of type object, so we also have to explicitly implement that.
	object IEnumerator.Current { get { return this.Current; } }
	public T Current { get { return this.Source.Current; } }
	
	public FilterEnumerator(IEnumerator<T> source, Predicate<T> predicate) 
	{
		this.Source = source;
		this.Predicate = predicate;
	}
	
	public bool MoveNext() 
	{
		// This is the juicy part. We basically need to skip anything that doesn't meet the predicate function. Cue a while loop!
		// MoveNext will return false when we don't have any more items, so we can conveniently use it as our while loop condition.
		while (this.Source.MoveNext()) 
		{
			// We check the predicate and skip if it wasn't met.
			if (!this.Predicate(this.Source.Current)) 
			{ 
				continue; 
			}
			
			// If we get here, the predicate was met, so stop and let the caller know we have an item. 
			return true;
		}
		
		// If we get here it means we exhausted our source iterator. That means this iterator is also exhausted.
		return false;
	}
	
	// Dispose and Reset can be ignored in most C# iterator implementations.
	public void Dispose() {}
	public void Reset() {}
}
```

With all the pieces in place, we can actually use our new functionality in some for loops.

```C#
public class Program
{
	public static void Main()
	{
		List<int> list =  new List<int>() { 0, 1, 2, 3, 4 };
		foreach(int item in list.Filter(item => item < 3)) {
			// result:
			// 0  
			// 1  
			// 2 
			Console.WriteLine(item);
		}
	}
}
```

With all of that code, we've managed to replicate the functionality of the .Where() method in System.Linq. It's a very similar process for .map()/.Select(), .flatMap()/.SelectMany() and the other derived iterators. If they don't already exist in your language, it is not terribly difficult to implement them yourself (I'm looking at you JavaScript).

This type of library ensures you don't create all of this boilerplate for fairly trivial modifications to other iterators and lets you continue to take full advantage of _lazy iteration_.

## Iterators and Multi-Threading (Rayon)
Iterators have one other really powerful feature when integrating with libraries. They are fairly easy to effectively multi-thread. The iterator is just some blob of data that will yield a value on demand, so as long as we get the locking right and the iterator itself isn't our performance bottleneck, we can have several threads all pulling values out of that iterator.

I'm going to be referring back to the Fibonacci iterator we created in [An Introduction to Iterators](https://adamfortune.com/blog/AnIntroductiontoIterators). Imagine I want to create a file for each of the first 1000 Fibonacci numbers. I might start with code like this:

```rust
use std::fs::File;

fn main() {
    let mut fib_first_1000 = FibonacciIterator::new().take(1000);
    for fib in fib_first_1000 {
        File::create(&format!("~/{fib}.txt"));
    }
}
```

This is, of course, going to be very slow. With iterators we can effectively parallelize fairly trivially. We just need to:
1. Put our iterator behind a Mutex or some other kind of lock.
2. Tell each thread to pull from the iterator each time it finishes creating the previous file.
3. Make sure that only one thread pulls the terminations value out of the iterator.
4. Join the threads when the iterator is finished.

```rust
use std::thread;
use std::sync::{
    Mutex, 
    Arc,
    atomic::AtomicBool
};

let fib_first_1000 = FibonacciIterator::new().take(1000);
let protected_iter = Arc::new(Mutex::new(fib_first_1000));
let finished = Arc::new(Mutex::new(false));

let num_threads = 8;
let mut handles = Vec::new();
for _ in 0..num_threads {
	let iter_reference = protected_iter.clone();
	let finished_reference = finished.clone();
	handles.push(
		thread::spawn(move || {
			loop {
				let fib;
				// create an inner block so the locks drop out of scope before file system operations.
				{
					// Lock the iterator and get the next value.
					let mut iter_lock = iter_reference.lock().unwrap();
					
					// Make sure one of the other threads didn't already get the last value. We don't want to get into undefined behavior.
					let mut finished_lock = finished_reference.lock().unwrap();
					if *finished_lock { break; }
					let next = iter_lock.next();
					
					match next {
						None => {
							// We got the last value. Mark the loop as finished and break!
							*finished_lock = true;
							break;
						}
						Some(value) => fib = value
					}
				}

				File::create(&format!("~/{fib}.txt")).ok();
			}
		})
	);
}

// rejoin all the threads.
for handle in handles {
	handle.join().ok();
}
```

That's a lot of code and a lot of places to mess up the locking! Luckily, most of it can be generalized for us. That's where Rayon comes in. Rayon is effectively just handling all of these locking shenanigans around the iterator as well as managing a thread pool to avoid spawning threads and reuse the previous ones for us. When we pull in Rayon as a dependency we can turn the previous code into the following:

```rust
use rayon::prelude::*;

let fib_first_1000 = FibonacciIterator::new().take(1000);
fib_first_1000
	.par_bridge()
	.for_each(|fib| {
		File::create(&format!("~/{fib}.txt")).ok();
	});
```

Pretty neat! As you can see, the consistent interface that iterators provide can allow for some really powerful libraries and generalizations. For this reason, I always reach for them instead of building lists up inside of loops.