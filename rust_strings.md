# Rust: How are Strings stored in memory?

<img src="https://miro.medium.com/v2/resize:fit:470/1*LGgyFhWLUwEpYgBNIzjmEw.png" title="" alt="" data-align="center">

All of us know how strings are stored in memory. The Rust Book explain it:

> A `String` is made up of three parts: a pointer to the memory that holds the 
> contents of the string, a length, and a capacity. This group of data is 
> stored on the stack.

For example, if you run the following code:

```rust
let s1 = String::from("Hello");
```

The storage result is shown in Figure 1. On the left is the data group in 
the stack, and on the right is the memory on the heap that holds the 
contents.

<img src="https://miro.medium.com/v2/resize:fit:273/1*OQ2B8gDnHumrj3xfJsC_OQ.png" title="" alt="" data-align="center">

If you use the method *std::mem::size_of_val()* with a reference to the value of the String *s1*, you will obtain *24*. That method gives us the size of a value in the stack, not in the heap. *24* is equal to *3x8*. Each record in the data group of the String in the stack has a size *usize* that is 8 bytes long in a 64-bit computer.

```rust
println!("{}", std::mem::size_of_val(&s1)); // Prints 24 = 3x8
```

Well, and what is the size of the String in the heap? You can use the String methods:

```rust
println!("{}", s1.len()); // Prints 5, the size of the String  
println!("{}", s1.capacity()); // Prints 5, the capacity of the String
```

And what is the address of the String in the heap? You could think that the address of the String in the heap is *&s1*.  You’re mistaken; this is the address of the data group of the String in  the stack. But, you can obtain the address of the String in the heap 
using the method *as_ptr()*:

```rust
println("{:p}", &s1); // 0x7fff58ffb178, the address in the stack  
println("{:p}", s1.as_ptr()); // 0x5f373059dba0, the address in the heap
```

Perhaps you are thinking: if we know the address of the String in the stack, 
can we read the three values dereferencing the pointers? Yes, and not. 
The problem is that the order of the three fields of the String in the stack is not guaranteed. We know that the data block of the String contains the three fields (heap pointer, capacity, and length), but we don’t know the order in which they are stored in the data block. The same String and program store them in a different order if you use a different computer (compiler).

In any case, to read the content of raw pointers, we need to use an *unsafe* environment. Check the following program:

```rust
fn main() {  
   let mut s1 = String::with_capacity(25);  
   s1.push_str("Hello");  
   println!("Size in the stack: {}", std::mem::size_of_val(&s1));  
   let string_ptr= &s1;  
   println!("String pointer in the stak: {:p}", string_ptr);  
   println!("Heap pointer to the string content: {:p}", s1.as_ptr());
   unsafe {  
      let pos1_ptr = string_ptr as *const String as *const usize;  
      println!("Pos 1: {:x}", *pos1_ptr);  
      let pos2_ptr = pos1_ptr.add(1);  
      println!("Pos 2: {:x}", *pos2_ptr);  
      let pos3_ptr = pos1_ptr.add(2);  
      println!("Pos 3: {:x}", *pos3_ptr);  
   }  
}
```

If  you run the previous program, you will obtain the three values by dereferencing the pointers. One of them is the capacity (25 decimal equals 19 hexadecimal), the other is the heap pointer (must be the same as the value obtained with the method *as_ptr())*, and the other is the length of the string (5 in this example). But the order is undefined. I have two computers in my home; one shows the heap pointer in the first position, and the other shows the capacity in the first position. You can also check the program in the [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=5a2d950bf2d1b88d07d2a63febaa82e0), which shows the heap pointer in the second position.

Given this, I think that the best method to obtain the pointer to the content of the String in the heap memory is the *as_ptr()* method.

Perhaps, you are thinking: if we know the location of the String content in the heap memory, we can read the characters. Yes, we can, but we need to know how they are stored.

A String is a Struct with one unique field of type *Vec<u8> (look at [string.rs - source](https://doc.rust-lang.org/src/alloc/string.rs.html#365)):

```rust
pub struct String {  
 vec: Vec<u8>,  
}
```

The content of the heap memory is a vector of *u8 v*alues. In the previous program, we could add a function to read the String allocated in the heap memory char by char:

```rust
fn main() {
   ...  

   for i in 0..s1.len() as usize {  
      println!("{}", read_char(s1.as_ptr(), i));  
   }
}  
fn read_char(heap_ptr: *const u8, position: usize) -> char {  
   let ch_value =unsafe {*heap_ptr.add(position)};  
   ch_value as char  
}
```

This example works because it only contains ASCII chars, but Strings in Rust are composed of UTF-8 chars, and some of these chars use more than one byte. We can’t decode the *Vec<u8>* with the previous code; we won’t obtain an accurate result if the String has some non-ASCII char.

The following code reads the vector from the heap and decodes it with the method *from_utf_8_lossy().*

```rust
fn main() {  
   let mut s1 = String::with_capacity(25);  
   s1.push_str("¡Feliz año!");  
   let v: Vec<u8> = read_vec(s1.as_ptr(), s1.len());  
   println!("{:?}", v);  
   let s = String::from_utf8_lossy(&v);  
   println!("{}", s);  
}  
fn read_vec(heap_ptr: *const u8, length: usize) -> Vec<u8> {  
   let mut v = Vec::<u8>::new();  
   unsafe {  
      for i in 0..length {  
      let ptr = heap_ptr.add(i);  
      v.push(*ptr);  
   }
   v  
}  
```

Well, it’s a bit of a convoluted way to read and print a string of text on the screen, but at the end of the day, we’re here to play, have fun, and learn a little about Rust, right?
