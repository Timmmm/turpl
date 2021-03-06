% Drop Flags

The examples in the previous section introduce an interesting problem for Rust.
We have seen that's possible to conditionally initialize, deinitialize, and
*reinitialize* locations of memory totally safely. For Copy types, this isn't
particularly notable since they're just a random pile of bits. However types with
destructors are a different story: Rust needs to know whether to call a destructor
whenever a variable is assigned to, or a variable goes out of scope. How can it
do this with conditional initialization?

It turns out that Rust actually tracks whether a type should be dropped or not *at
runtime*. As a variable becomes initialized and uninitialized, a *drop flag* for
that variable is toggled. When a variable *might* need to be dropped, this flag
is evaluated to determine if it *should* be dropped.

Of course, it is *often* the case that a value's initialization state can be
*statically* known at every point in the program. If this is the case, then the
compiler can theoretically generate more effecient code! For instance,
straight-line code has such *static drop semantics*:

```rust
let mut x = Box::new(0); // x was uninit
let mut y = x;			 // y was uninit
x = Box::new(0);	 	 // x was uninit
y = x;				 	 // y was init; Drop y!
				     	 // y was init; Drop y!
				     	 // x was uninit
```

And even branched code where all branches have the same behaviour with respect
to initialization:

```rust
let mut x = Box::new(0);	// x was uninit
if condition {
	drop(x)					// x gets moved out
} else {
	println!("{}", x);
	drop(x)					// x gets moved out
}
x = Box::new(0);			// x was uninit
							// x was init; Drop x!
```

However code like this *requires* runtime information to correctly Drop:

```rust
let x;
if condition {
	x = Box::new(0);		// x was uninit
	println!("{}", x);
}
							// x might be uninit; check the flag!
```

Of course, in this case it's trivial to retrieve static drop semantics:

```rust
if condition {
	let x = Box::new(0);
	println!("{}", x);
}
```

As of Rust 1.0, the drop flags are actually not-so-secretly stashed in a hidden
field of any type that implements Drop. Rust sets the drop flag by
overwriting the *entire* value with a particular byte. This is pretty obviously
Not The Fastest and causes a bunch of trouble with optimizing code. It's legacy
from a time when you could do much more complex conditional initialization.

As such work is currently under way to move the flags out onto the stack frame
where they more reasonably belong. Unfortunately, this work will take some time
as it requires fairly substantial changes to the compiler.

Regardless, Rust programs don't need to worry about uninitialized values on
the stack for correctness. Although they might care for performance. Thankfully,
Rust makes it easy to take control here! Uninitialized values are there, and
you can work with them in Safe Rust, but you're *never* in danger.