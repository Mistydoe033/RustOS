# Freestanding Rust Binary

## Overview
This project features a Rust executable that does not link to the standard library, enabling the execution of Rust code on bare metal without an underlying operating system.

## Panic Implementation
In a `no_std` environment, a custom panic handler must be implemented, the standard library provides its own by default.

### Kernel Development
A minimal 64-bit Rust kernel for the x86 architecture is built on top of the freestanding Rust binary, allowing the creation of a bootable disk image.

## Memory-Related Intrinsics
Certain memory-related functions in the `compiler_builtins` crate are not enabled by default since they are typically provided by the C library on the system. Key functions include:
- `memset`: Sets all bytes in a memory block to a specified value.
- `memcpy`: Copies one memory block to another.
- `memcmp`: Compares two memory blocks.

While these functions are not currently necessary for compiling the kernel, they will be required as additional code is integrated, such as when copying structures.

## VGA Text Buffer
The easiest method for outputting text at this stage involves using the VGA text buffer, a special memory area mapped to the VGA hardware that contains the contents displayed on the screen.
To print a character in VGA text mode, it is necessary to write to the text buffer of the VGA hardware.

The VGA text buffer is a two-dimensional array with typically 25 rows and 80 columns, directly rendered to the screen.
Each array entry describes a single screen character in the following format:

| Bit(s) | Value                          |
|--------|--------------------------------|
| 0-7    | ASCII code point               |
| 8-11   | Foreground color               |
| 12-14  | Background color               |
| 15     | Blink                          |

The first byte represents the character in ASCII encoding, specifically code page 437, which includes some additional characters and slight modifications.
The second byte defines the display characteristics, the first four bits represent the foreground color,
the next three bits represent the background color, and the last bit determines if the character should blink.
Colors are represented using a C-like enum, with the `repr(u8)` attribute ensuring each variant is stored as a `u8`.
Actually 4 bits would be sufficient, but Rust doesn’t have a `u4` type.

## Text Buffer
Structures are created to represent a screen character and the text buffer.
Due to undefined field ordering in default structs, the `repr(C)` attribute is used to guarantee correct field layout, similar to a C struct.
The `Buffer` struct employs `repr(transparent)` to maintain the same memory layout as its single field.

A writer type is implemented to facilitate writing to the screen.
This writer writes to the last line and shifts lines up when a line is filled (or on `\n`).
The `column_position` field tracks the current position in the last row, while the current foreground and background colors are specified by `color_code`.
A reference to the VGA buffer is stored in `buffer`, necessitating an explicit lifetime to inform the compiler of the reference's validity.
The `'static` lifetime indicates the reference is valid for the entire program runtime, as is true for the VGA text buffer.

## Volatile
To utilize volatile writes for the VGA buffer, the `volatile` library is employed.
This crate provides a `Volatile` wrapper type with `read` and `write` methods that leverage the `read_volatile` and `write_volatile` functions from the core library,
ensuring that reads and writes are not optimized away by the Rust compiler.

## Spinlocks
For synchronized interior mutability, the standard library offers `Mutex`, which provides mutual exclusion by blocking threads when a resource is locked.
However, since this basic kernel lacks blocking support or the concept of threads, a simple mutex known as a spinlock is utilized.
Instead of blocking, threads repeatedly attempt to lock the spinlock in a tight loop, consuming CPU time until the mutex becomes available.

## Safety
Only a single unsafe block is present in the code, required to create a `Buffer` reference pointing to `0xb8000`.
All subsequent operations are safe due to Rust's bounds checking for array accesses, preventing accidental writes outside the buffer.
The type system encodes the necessary conditions, ensuring a safe interface to the outside.
