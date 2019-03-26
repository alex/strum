# Strum

[![Build Status](https://travis-ci.org/Peternator7/strum.svg?branch=master)](https://travis-ci.org/Peternator7/strum)
[![Latest Version](https://img.shields.io/crates/v/strum.svg)](https://crates.io/crates/strum)
[![Rust Documentation](https://docs.rs/strum/badge.svg)](https://docs.rs/strum)

Strum is a set of macros and traits for working with
enums and strings easier in Rust.

# Compatibility

Strum is compatible with versions of rustc >= 1.26.0. That's the earliest version of stable rust that supports
impl trait. Pull Requests that improve compatibility with older versions are welcome, but new feature work
will focus on the current version of rust with an effort to avoid breaking compatibility with older versions.

Versions of rust prior to 1.31.0 don't support importing procedural macros by path. See [this wiki page](https://github.com/Peternator7/strum/wiki/Macro-Renames) if you are finding that one of Strum's macros collides with a macro being imported by a different crate. *You do not need this in versions of rust >= 1.31.0*

# Including Strum in Your Project

Import strum and strum_macros into your project by adding the following lines to your
Cargo.toml. Strum_macros contains the macros needed to derive all the traits in Strum.

```toml
[dependencies]
strum = "0.14.0"
strum_macros = "0.14.0"
```

And add these lines to the root of your project, either lib.rs or main.rs.

```rust
// Strum contains all the trait definitions
extern crate strum;
#[macro_use]
extern crate strum_macros;

// Instead of #[macro_use], newer versions of rust should prefer
use strum_macros::{Display, EnumIter}; // etc.
```

# Contributing

Thanks for your interest in contributing. The project is divided into 3 parts, the traits are in the
`/strum` folder. The procedural macros are in the `/strum_macros` folder, and the integration tests are
in `/strum_tests`. If you are adding additional features to `strum` or `strum_macros`, you should make sure
to run the tests and add new integration tests to make sure the features work as expected.

# Debugging

To see the generated code, set the STRUM_DEBUG environment variable before compiling your code.
`STRUM_DEBUG=1` will dump all of the generated code for every type. `STRUM_DEBUG=YourType` will
only dump the code generated on a type named `YourType`.

# Strum Macros

| Macro | Description |
| --- | ----------- |
| [EnumString](#EnumString) | Converts strings to enum variants based on their name |
| [Display](#Display) | Converts enum variants to strings |
| [AsRefStr](#AsRefStr) | Converts enum variants to `&'static str` |
| [IntoStaticStr](#IntoStaticStr) | Implements `From<MyEnum> for &'static str` on an enum |
| [EnumIter](#EnumIter) | Creates a new type that iterates of the variants of an enum. |
| [EnumProperty](#EnumProperty) | Add custom properties to enum variants. |
| [EnumMessage](#EnumMessage) | Add a verbose message to an enum variant. |
| [EnumDiscriminants](#EnumDiscriminants) | Generate a new type with only the discriminant names. |
| [EnumCount](#EnumCount) | Add a constant `usize` equal to the number of variantes. |

Strum has implemented the following macros:

## EnumString

auto-derives `std::str::FromStr` on the enum. Each variant of the enum will match on it's own name.
This can be overridden using `serialize="DifferentName"` or `to_string="DifferentName"`
on the attribute as shown below.
Multiple deserializations can be added to the same variant. If the variant contains additional data,
they will be set to their default values upon deserialization.

The `default` attribute can be applied to a tuple variant with a single data parameter. When a match isn't
found, the given variant will be returned and the input string will be captured in the parameter.

Here is an example of the code generated by deriving `EnumString`.

```rust
extern crate strum;
#[macro_use] extern crate strum_macros;
#[derive(EnumString)]
enum Color {
    Red,

    // The Default value will be inserted into range if we match "Green".
    Green { range:usize },

    // We can match on multiple different patterns.
    #[strum(serialize="blue",serialize="b")]
    Blue(usize),

    // Notice that we can disable certain variants from being found
    #[strum(disabled="true")]
    Yellow,
}

/*
//The generated code will look like:
impl std::str::FromStr for Color {
    type Err = ::strum::ParseError;

    fn from_str(s: &str) -> ::std::result::Result<Color, Self::Err> {
        match s {
            "Red" => ::std::result::Result::Ok(Color::Red),
            "Green" => ::std::result::Result::Ok(Color::Green { range:Default::default() }),
            "blue" | "b" => ::std::result::Result::Ok(Color::Blue(Default::default())),
            _ => ::std::result::Result::Err(::strum::ParseError::VariantNotFound),
        }
    }
}
*/
```

Note that the implementation of `FromStr` by default only matches on the name of the
variant. There is an option to match on different case conversions through the
`#[strum(serialize_all = "snake_case")]` type attribute. See the **Additional Attributes**
Section for more information on using this feature.

## Display

Print out the given enum. This enables you to perform round trip style conversions
from enum into string and back again for unit style variants.
`Display` choose which serialization to used based on the following criteria:

1. If there is a `to_string` property, this value will be used. There can only be one per variant.
2. Of the various `serialize` properties, the value with the longest length is chosen. If that
    behavior isn't desired, you should use `to_string`.
3. The name of the variant will be used if there are no `serialize` or `to_string` attributes.

```rust
// You need to bring the type into scope to use it!!!
use std::string::ToString;

#[derive(ToString,Debug)]
enum Color {
    #[strum(serialize="redred")]
    Red,
    Green { range:usize },
    Blue(usize),
    Yellow,
}

// It's simple to iterate over the variants of an enum.
fn debug_colors() {
    let red = Color::Red;
    assert_eq!(String::from("redred"), red.to_string());
}

fn main() {
    debug_colors();
}
```

## AsRefStr

Implements `AsRef<str>` on your enum using the same rules as
`ToString` for determining what string is returned. The difference is that `as_ref()` returns
a `&str` instead of a `String` so you don't allocate any additional memory with each call.

## IntoStaticStr

Implements `From<YourEnum>` and `From<&'a YourEnum>` for `&'static str`. This is
useful for turning an enum variant into a static string.
The Rust `std` provides a blanket impl of the reverse direction - i.e. `impl Into<&'static str> for YourEnum`.

```rust
extern crate strum;
#[macro_use] extern crate strum_macros;

#[derive(IntoStaticStr)]
enum State<'a> {
    Initial(&'a str),
    Finished
}

fn print_state<'a>(s:&'a str) {
    let state = State::Initial(s);
    // The following won't work because the lifetime is incorrect so we can use.as_static() instead.
    // let wrong: &'static str = state.as_ref();
    let right: &'static str = state.into();
    println!("{}", right);
}

fn main() {
    print_state(&"hello world".to_string())
}
```

## EnumIter

Iterate over the variants of an Enum. Any additional data on your variants will be set to `Default::default()`.
The macro implements `strum::IntoEnumIter` on your enum and creates a new type called `YourEnumIter` that is the iterator object.
You cannot derive `EnumIter` on any type with a lifetime bound (`<'a>`) because the iterator would surely
create [unbounded lifetimes](https://doc.rust-lang.org/nightly/nomicon/unbounded-lifetimes.html).

```rust
// You need to bring the type into scope to use it!!!
use strum::IntoEnumIterator;

#[derive(EnumIter,Debug)]
enum Color {
    Red,
    Green { range:usize },
    Blue(usize),
    Yellow,
}

// It's simple to iterate over the variants of an enum.
fn debug_colors() {
    for color in Color::iter() {
        println!("My favorite color is {:?}", color);
    }
}

fn main() {
    debug_colors();
}
```

## EnumMessage

Encode strings into the enum itself. This macro implements the `strum::EnumMessage` trait.
`EnumMessage` looks for `#[strum(message="...")]` attributes on your variants.
You can also provided a `detailed_message="..."` attribute to create a seperate more detailed message than the first.

The generated code will look something like:

```rust
// You need to bring the type into scope to use it!!!
use strum::EnumMessage;

#[derive(EnumMessage,Debug)]
enum Color {
    #[strum(message="Red",detailed_message="This is very red")]
    Red,
    #[strum(message="Simply Green")]
    Green { range:usize },
    #[strum(serialize="b",serialize="blue")]
    Blue(usize),
}

/*
// Generated code
impl ::strum::EnumMessage for Color {
    fn get_message(&self) -> ::std::option::Option<&str> {
        match self {
            &Color::Red => ::std::option::Option::Some("Red"),
            &Color::Green {..} => ::std::option::Option::Some("Simply Green"),
            _ => None
        }
    }

    fn get_detailed_message(&self) -> ::std::option::Option<&str> {
        match self {
            &Color::Red => ::std::option::Option::Some("This is very red"),
            &Color::Green {..}=> ::std::option::Option::Some("Simply Green"),
            _ => None
        }
    }

    fn get_serializations(&self) -> &[&str] {
        match self {
            &Color::Red => {
                static ARR: [&'static str; 1] = ["Red"];
                &ARR
            },
            &Color::Green {..}=> {
                static ARR: [&'static str; 1] = ["Green"];
                &ARR
            },
            &Color::Blue (..) => {
                static ARR: [&'static str; 2] = ["b", "blue"];
                &ARR
            },
        }
    }
}
*/
```

## EnumProperty

Enables the encoding of arbitary constants into enum variants. This method
currently only supports adding additional string values. Other types of literals are still
experimental in the rustc compiler. The generated code works by nesting match statements.
The first match statement matches on the type of the enum, and the inner match statement
matches on the name of the property requested. This design works well for enums with a small
number of variants and properties, but scales linearly with the number of variants so may not
be the best choice in all situations.

Here's an example:

```rust
# extern crate strum;
# #[macro_use] extern crate strum_macros;
# use std::fmt::Debug;
// You need to bring the type into scope to use it!!!
use strum::EnumProperty;

#[derive(EnumProperty,Debug)]
enum Color {
    #[strum(props(Red="255",Blue="255",Green="255"))]
    White,
    #[strum(props(Red="0",Blue="0",Green="0"))]
    Black,
    #[strum(props(Red="0",Blue="255",Green="0"))]
    Blue,
    #[strum(props(Red="255",Blue="0",Green="0"))]
    Red,
    #[strum(props(Red="0",Blue="0",Green="255"))]
    Green,
}

fn main() {
    let my_color = Color::Red;
    let display = format!("My color is {:?}. It's RGB is {},{},{}", my_color
                                        , my_color.get_str("Red").unwrap()
                                        , my_color.get_str("Green").unwrap()
                                        , my_color.get_str("Blue").unwrap());
}
```

## EnumDiscriminants

Given an enum named `MyEnum`, generates another enum called `MyEnumDiscriminants` with the same variants, without any data fields.
This is useful when you wish to determine the variant of an enum from a String, but the variants contain any
non-`Default` fields. By default, the generated enum has the following derives:
`Clone, Copy, Debug, PartialEq, Eq`. You can add additional derives using the
`#[strum_discriminants(derive(AdditionalDerive))]` attribute.

Here's an example:

```rust
extern crate strum;
#[macro_use] extern crate strum_macros;

// Bring trait into scope
use std::str::FromStr;

#[derive(Debug)]
struct NonDefault;

#[allow(dead_code)]
#[derive(Debug, EnumDiscriminants)]
#[strum_discriminants(derive(EnumString))]
enum MyEnum {
    Variant0(NonDefault),
    Variant1 { a: NonDefault },
}

fn main() {
    assert_eq!(
        MyEnumDiscriminants::Variant0,
        MyEnumDiscriminants::from_str("Variant0").unwrap()
    );
}
```

You can also rename the generated enum using the `#[strum_discriminants(name(OtherName))]`
attribute:

```rust
extern crate strum;
#[macro_use] extern crate strum_macros;
// You need to bring the type into scope to use it!!!
use strum::IntoEnumIterator;

#[allow(dead_code)]
#[derive(Debug, EnumDiscriminants)]
#[strum_discriminants(derive(EnumIter))]
#[strum_discriminants(name(MyVariants))]
enum MyEnum {
    Variant0(bool),
    Variant1 { a: bool },
}

fn main() {
    assert_eq!(
        vec![MyVariants::Variant0, MyVariants::Variant1],
        MyVariants::iter().collect::<Vec<_>>()
    );
}
```

The derived enum also has the following trait implementations:

* `impl From<MyEnum> for MyEnumDiscriminants`
* `impl<'_enum> From<&'_enum MyEnum> for MyEnumDiscriminants`

These allow you to get the *Discriminants* enum variant from the original enum:

```rust
extern crate strum;
#[macro_use] extern crate strum_macros;

#[derive(Debug, EnumDiscriminants)]
#[strum_discriminants(name(MyVariants))]
enum MyEnum {
    Variant0(bool),
    Variant1 { a: bool },
}

fn main() {
    assert_eq!(MyVariants::Variant0, MyEnum::Variant0(true).into());
}
```

## EnumCount

For a given enum generates implementation of `strum::EnumCount`,
which returns number of variants via `strum::EnumCount::count` method,
also for given `enum MyEnum` generates `const MYENUM_COUNT: usize`
which gives the same value as `strum::EnumCount` (which is usefull for array sizes, etc.).

```rust
extern crate strum;
#[macro_use]
extern crate strum_macros;

use strum::{IntoEnumIterator, EnumCount};

#[derive(Debug, EnumCount, EnumIter)]
enum Week {
    Sunday,
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
}

fn main() {
    assert_eq!(7, Week::count());
    assert_eq!(Week::count(), WEEK_COUNT);
    assert_eq!(Week::iter().count(), WEEK_COUNT);
}
```

## ToString

**Deprecated** Prefer using [`Display`](#Display). All types that implement `std::fmt::Display` have a default implementation of `ToString`.**

## AsStaticStr

**Deprecated since version 0.13.0.** Prefer [IntoStaticStr](#IntoStaticStr) instead.

# Additional Attributes

Strum supports several custom attributes to modify the generated code. At the enum level, the
`#[strum(serialize_all = "snake_case")]` attribute can be used to change the case used when
serializing to and deserializing from strings:

```rust
extern crate strum;
#[macro_use]
extern crate strum_macros;

#[derive(Debug, Eq, PartialEq, ToString)]
#[strum(serialize_all = "snake_case")]
enum Brightness {
    DarkBlack,
    Dim {
        glow: usize,
    },
    #[strum(serialize = "bright")]
    BrightWhite,
}

fn main() {
    assert_eq!(
        String::from("dark_black"),
        Brightness::DarkBlack.to_string().as_ref()
    );
    assert_eq!(
        String::from("dim"),
        Brightness::Dim { glow: 0 }.to_string().as_ref()
    );
    assert_eq!(
        String::from("bright"),
        Brightness::BrightWhite.to_string().as_ref()
    );
}
```

Custom attributes are applied to a variant by adding `#[strum(parameter="value")]` to the variant.

- `serialize="..."`: Changes the text that `FromStr()` looks for when parsing a string. This attribute can
   be applied multiple times to an element and the enum variant will be parsed if any of them match.

- `to_string="..."`: Similar to `serialize`. This value will be included when using `FromStr()`. More importantly,
   this specifies what text to use when calling `variant.to_string()` with the `ToString` derivation, or when calling `variant.as_ref()` with `AsRefStr`.

- `default="true"`: Applied to a single variant of an enum. The variant must be a Tuple-like
   variant with a single piece of data that can be create from a `&str` i.e. `T: From<&str>`.
   The generated code will now return the variant with the input string captured as shown below
   instead of failing.

    ```rust,ignore
    // Replaces this:
    _ => Err(strum::ParseError::VariantNotFound)
    // With this in generated code:
    default => Ok(Variant(default.into()))
    ```
    The plugin will fail if the data doesn't implement From<&str>. You can only have one `default`
    on your enum.

- `disabled="true"`: removes variant from generated code.

- `message=".."`: Adds a message to enum variant. This is used in conjunction with the `EnumMessage`
   trait to associate a message with a variant. If `detailed_message` is not provided,
   then `message` will also be returned when get_detailed_message() is called.

- `detailed_message=".."`: Adds a more detailed message to a variant. If this value is omitted, then
   `message` will be used in it's place.

- `props(key="value")`: Enables associating additional information with a given variant.

# Examples

Using `EnumMessage` for quickly implementing `Error`

```rust
extern crate strum;
#[macro_use]
extern crate strum_macros;
use std::error::Error;
use std::fmt::*;
use strum::EnumMessage;

#[derive(Debug, EnumMessage)]
enum ServerError {
    #[strum(message="A network error occured")]
    #[strum(detailed_message="Try checking your connection.")]
    NetworkError,
    #[strum(message="User input error.")]
    #[strum(detailed_message="There was an error parsing user input. Please try again.")]
    InvalidUserInputError,
}

impl Display for ServerError {
    fn fmt(&self, f: &mut Formatter) -> Result {
        write!(f, "{}", self.get_message().unwrap())
    }
}

impl Error for ServerError {
    fn description(&self) -> &str {
        self.get_detailed_message().unwrap()
    }
}
```

Using `EnumString` to tokenize a series of inputs:

```rust
extern crate strum;
#[macro_use]
extern crate strum_macros;
use std::str::FromStr;

#[derive(Eq, PartialEq, Debug, EnumString)]
enum Tokens {
    #[strum(serialize="fn")]
    Function,
    #[strum(serialize="(")]
    OpenParen,
    #[strum(serialize=")")]
    CloseParen,
    #[strum(default="true")]
    Ident(String)
}

fn main() {
    let toks = ["fn", "hello_world", "(", ")"].iter()
                   .map(|tok| Tokens::from_str(tok).unwrap())
                   .collect::<Vec<_>>();

    assert_eq!(toks, vec![Tokens::Function,
                          Tokens::Ident(String::from("hello_world")),
                          Tokens::OpenParen,
                          Tokens::CloseParen]);
}
```

# Name

Strum is short for STRing enUM because it's a library for augmenting enums with additional
information through strings.

Strumming is also a very whimsical motion, much like writing Rust code.
