# FFI Destruct
[![crates badge]][crates.io] [![docs badge]][docs.rs] [![build badge]][build]

[crates badge]: https://img.shields.io/crates/v/ffi-destruct.svg?logo=rust
[crates.io]: https://crates.io/crates/ffi-destruct
[docs badge]: https://img.shields.io/docsrs/ffi-destruct/latest?label=docs.rs&logo=docs.rs
[docs.rs]: https://docs.rs/ffi-destruct
[build badge]: https://github.com/I-Info/FFI-destruct/actions/workflows/build.yml/badge.svg
[build]: https://github.com/I-Info/FFI-destruct/actions/workflows/build.yml

Generates destructors for structures that contain raw pointers in the FFI.

## About
The `Destruct` derive macro will implement `Drop` trait and free(drop) memory for structures containing raw pointers.
It may be a common procedure for FFI resource management.

## Safety
All the raw pointer dropping operations are unsafe and almost unchecked. 
**Should only drop resources managed by Rust side.**

## Supported types
Both `*const` and `*mut` are acceptable. 
But currently, only single-level pointers are supported.

| type       | handler                           | note                                                                                             |
| ---------- | --------------------------------- | ------------------------------------------------------------------------------------------------ |
| `* c_char` | `::std::ffi::CString::from_raw()` | C-style string. Likely type path: </br> `std::ffi::c_char` `libc::c_char` `std::os::raw::c_char` |
| `* <T>`    | `::std::boxed::Box::from_raw()`   | Anything heap-allocated by Rust. Something likely from: </br> `Box::into_raw(Box::new(<T>))`     |

## Example
Provides a structure with several raw pointers that need to be dropped manually.
```rust
use ffi_destruct::{extern_c_destructor, Destruct};
use std::ffi::*;

#[derive(Destruct)]
pub struct MyStruct {
    field: *mut std::ffi::c_char,
}

pub struct AnyOther(u32, u32);

// Struct definition here, with deriving Destruct and nullable attributes.
#[derive(Destruct)]
pub struct Structure {
    // Default is non-null.
    c_string: *const c_char,
    #[nullable]
    c_string_nullable: *mut c_char,

    other: *mut MyStruct,
    #[nullable]
    other_nullable: *mut MyStruct,

    // Do not drop this field.
    #[no_drop]
    not_dropped: *const AnyOther,

    // Raw pointer for any other things
    any: *mut AnyOther,

    // Non-pointer types are still available, and will not be added to drop().
    pub normal_int: u32,
    pub normal_string: String,
}

// (Optional) The macro here generates the destructor: destruct_structure()
extern_c_destructor!(Structure);

fn main() {
    // Some resources manually managed
    let tmp = AnyOther(1, 1);
    let tmp_ptr = Box::into_raw(Box::new(tmp));

    let my_struct = Structure {
        c_string: CString::new("Hello").unwrap().into_raw(),
        c_string_nullable: std::ptr::null_mut(),
        other: Box::into_raw(Box::new(MyStruct {
            field: CString::new("Hello").unwrap().into_raw(),
        })),
        other_nullable: std::ptr::null_mut(),
        not_dropped: tmp_ptr,
        any: Box::into_raw(Box::new(AnyOther(1, 1))),
        normal_int: 114514,
        normal_string: "Hello".to_string(),
    };

    let my_struct_ptr = Box::into_raw(Box::new(my_struct));
    // FFI calling
    unsafe {
        destruct_structure(my_struct_ptr);
    }

    // Drop the manually managed resources
    unsafe {
        let _ = Box::from_raw(tmp_ptr);
    }
}
```

After expanding the macros:
```rust
#[macro_use]
#![feature(prelude_import)]
#![allow(dead_code)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
use ffi_destruct::{extern_c_destructor, Destruct};
use std::ffi::*;
pub struct MyStruct {
    field: *mut std::ffi::c_char,
}
impl ::std::ops::Drop for MyStruct {
    fn drop(&mut self) {
        unsafe {
            let _ = ::std::ffi::CString::from_raw(self.field as *mut ::std::ffi::c_char);
        }
    }
}
pub struct AnyOther(u32, u32);
pub struct Structure {
    c_string: *const c_char,
    #[nullable]
    c_string_nullable: *mut c_char,
    other: *mut MyStruct,
    #[nullable]
    other_nullable: *mut MyStruct,
    #[no_drop]
    not_dropped: *const AnyOther,
    any: *mut AnyOther,
    pub normal_int: u32,
    pub normal_string: String,
}
impl ::std::ops::Drop for Structure {
    fn drop(&mut self) {
        unsafe {
            let _ = ::std::ffi::CString::from_raw(
                self.c_string as *mut ::std::ffi::c_char,
            );
            if !self.c_string_nullable.is_null() {
                let _ = ::std::ffi::CString::from_raw(
                    self.c_string_nullable as *mut ::std::ffi::c_char,
                );
            }
            let _ = ::std::boxed::Box::from_raw(self.other as *mut MyStruct);
            if !self.other_nullable.is_null() {
                let _ = ::std::boxed::Box::from_raw(
                    self.other_nullable as *mut MyStruct,
                );
            }
            let _ = ::std::boxed::Box::from_raw(self.any as *mut AnyOther);
        }
    }
}
#[no_mangle]
pub unsafe extern "C" fn destruct_structure(ptr: *mut Structure) {
    if ptr.is_null() {
        return;
    }
    let _ = ::std::boxed::Box::from_raw(ptr);
}
fn main() {
    let tmp = AnyOther(1, 1);
    let tmp_ptr = Box::into_raw(Box::new(tmp));
    let my_struct = Structure {
        c_string: CString::new("Hello").unwrap().into_raw(),
        c_string_nullable: std::ptr::null_mut(),
        other: Box::into_raw(
            Box::new(MyStruct {
                field: CString::new("Hello").unwrap().into_raw(),
            }),
        ),
        other_nullable: std::ptr::null_mut(),
        not_dropped: tmp_ptr,
        any: Box::into_raw(Box::new(AnyOther(1, 1))),
        normal_int: 114514,
        normal_string: "Hello".to_string(),
    };
    let my_struct_ptr = Box::into_raw(Box::new(my_struct));
    unsafe {
        destruct_structure(my_struct_ptr);
    }
    unsafe {
        let _ = Box::from_raw(tmp_ptr);
    }
}
```