# Rust: Shadowing is not Dropping

<img src="img/dropping_peq.png" title="" alt="" data-align="center">

In Rust, we can *shadow* a variable, creating a new one with the same name. “*The Rust Programming Language*” book shows this example [[shadowing]](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html#shadowing):

  

```rust
fn main() {
   let x = 5;
   let x = x + 1;  
   {  
      let x = x * 2;  
      println!("The value of x in the inner scope is: {x}"); // 12  
   }
   println!("The value of x is: {x}"); // 6
}
```

I thought that the original variable *x* was dropped and its memory deallocated. But it’s not the case; the variable continues alive, and its memory is not deallocated, even if you can’t access the value through its name.

My doubts arose when I played with *Rc* pointers and noted that shadowing an *Rc* pointer increments the *ref counter*. Take a look at this example:

```rust
use std::rc::Rc;

fn main() {  
   let a = Rc::new(5);  
   let ptr = Rc::as_ptr(&a);  
   let a = a.clone(); // Shadows a but increments counter  
   assert_eq!(Rc::strong_count(&a), 2);  
   let b = a.clone();  
   assert_eq!(Rc::strong_count(&a), 3);  
   std::mem::drop(a); // Decrements counter by one unit  
   assert_eq!(Rc::strong_count(&b), 2);  
   std::mem::drop(b);  
   unsafe{assert_eq!(*ptr, 5)}; // Is it alive in the heap?  
}
```

I asked about this code on [Reddit](https://www.reddit.com/r/rust/comments/1bwv1wo/strange_behaviour_of_rc/), and some people showed me how the shadowing mechanism works in Rust.

The first key concept came from user [u/TDplay](https://www.reddit.com/user/TDplay/). They show me that shadowing is not the same as dropping with this example:

```rust
struct LoudDrop(&'static str);  
impl Drop for LoudDrop {  
   fn drop(&mut self) {  
      println!("{} dropped", self.0);  
   }  
}
fn main() {  
   let a = LoudDrop("a1");  
   let a = LoudDrop("a2");  
   drop(a);  
   println!("End of main");
}
```

The output from this code is the following:

```shell
a2 dropped
End of main 
a1 dropped
```

As you can see, the original *a* is shadowed, but it is alive till the end of the *main()* function.

User [u/This_Growth2898](https://www.reddit.com/user/This_Growth2898/) gave me other important key. In my code, when I do `unsafe{assert_eq!(*ptr, 5)}`:

> *“It isn’t checking if the value is “alive”. It checks the value at the pointer in memory, but you don’t know if the memory at that address is allocated or not. Freeing memory doesn’t clean it up. Cleaning memory is not free, so compiler will mostly avoid it and leave garbage values.”*

He put this good example:

```rust
use std::rc::Rc;

fn main() {  
   let ptr;
   {
      let a = Rc::new(5); //could use Box as well
      ptr = Rc::as_ptr(&a);
   } //a is freed here
   unsafe{assert_eq!(*ptr, 5)}; // UB, but the value in memory is still 5!  
}
```

In my example, I think the value is alive, even though the [u/This_Growth2898](https://www.reddit.com/user/This_Growth2898/) comment is very interesting and important. To check my assertion, I did this code:

```rust
fn main() {
   let x = 1;
   let i32_ptr = &x;
   let x = 3.14;
   let f64_ptr = &x;
   let x = 'a';
   let char_ptr = &x;
   assert_eq!(x, 'a');
   assert_eq!(*i32_ptr, 1);
   assert_eq!(*f64_ptr, 3.14);
   assert_eq!(*char_ptr, 'a');
}
```

All the values are alive, and we can access them through references. But then, [This_Growth2898](https://www.reddit.com/user/This_Growth2898/) answered me again:

> *I’m not arguing that they are not alive in this code; I only say that the original way to check if something is alive is not correct, like here:*

```rust
let i32_ptr : * const i32;
let f64_ptr : * const f64;
let char_ptr : * const char;
{
   let x = 1;
   i32_ptr = &x;
   let x = 3.14;
   f64_ptr = &x;
   let x = 'a';
   char_ptr = &x;
}
unsafe{ //all variables are not alive, but still assertions may work!
   assert_eq!(*i32_ptr, 1);
   assert_eq!(*f64_ptr, 3.14);
   assert_eq!(*char_ptr, 'a');
}
```

The question is open. I don’t know if shadowing but not dropping is a good or bad idea, and I don’t know what the goal of this behaviour in Rust language is. I would like to hear other opinions.

Thanks for reading!






