# type-rules

A tool to easily constrain a struct and recover errors.

## Table of Contents

1. [Install](#install)
2. [Basic checking](#basic-checking)
3. [Advanced checking](#advanced-checking)
4. [Make your own rule](#make-your-own-rule)
5. [Rules list](#rules-list)

## Install
```toml
# Cargo.toml
[dependencies]
type-rules = { version = "0.1.2", features = ["derive", "regex"] }
```

## Basic checking

You can declare a struct and impose some constraints on each field 
and check the validity like this:
```rust
use chrono::prelude::*;
use type_rules::Validator;
//Don't forget to import the used rules.
use type_rules::rules::{
  MaxLength, 
  MinMaxLength, 
  RegEx, 
  Opt,
  MaxRange,
};

#[derive(Validator)]
struct NewUser {
    #[rule(MaxLength(100), RegEx(r"^\S+@\S+\.\S+"))]
    email: String,
    #[rule(MinMaxLength(8, 50))]
    password: String,
    #[rule(Opt(MaxRange(Utc::now())))]
    birth_date: Option<DateTime<Utc>>
}

let new_user = NewUser {
    email: "examples@examples.com".to_string(),
    password: "OPw$5%hJ".to_string(),
    birth_date: None,
};
assert!(new_user.check_validity().is_ok());
let new_user = NewUser {
    email: "examples@examples.com".to_string(),
    password: "O".to_string(),
    birth_date: None,
};
assert!(new_user.check_validity().is_err()); //Value is too short
```

## Advanced checking

To check recursively, you can use the `Validate` rule

```rust
use type_rules::rules::{MaxLength, RegEx, Validate, MinMaxLength};
use type_rules::Validator;

#[derive(Validator)]
struct EmailWrapper(#[rule(MaxLength(100), RegEx(r"^\S+@\S+\.\S+"))] String);

#[derive(Validator)]
struct User {
    #[rule(Validate())]
    email: EmailWrapper,
    #[rule(MinMaxLength(8, 50))]
    password: String,
}
```

You can use expressions directly in rule derive attribute.

For example, you can use const or function directly in the checker parameters:

```rust
use type_rules::rules::MaxRange;
use chrono::prelude::*;
use type_rules::Validator;

#[derive(Validator)]
struct BirthDate(#[rule(MaxRange(Utc::now()))] DateTime<Utc>);
```
```rust
use type_rules::rules::MinLength;
use type_rules::Validator;

const MIN_PASSWORD_LENGTH: usize = 8;

#[derive(Validator)]
struct Password(#[rule(MinLength(MIN_PASSWORD_LENGTH))] String);
```

Or use expressions to express a checker directly.
Here is an example of using a rule with more complex values:

```rust
use std::env;
use type_rules::rules::MaxLength;
use type_rules::Validator;

fn generate_max_payload_rule() -> MaxLength {
    MaxLength(match env::var("MAX_PAYLOAD") {
        Ok(val) => val.parse().unwrap_or_else(|_| 10000),
        Err(_) => 10000,
    })
}

#[derive(Validator)]
struct Payload(#[rule(generate_max_payload_rule())] String);
```

In this case the `generate_max_payload_rule` function is executed at each check

## Make your own rule

If you need a specific rule, just make a tuple struct (or struct if you make the declaration outside the struct definition)
that implements the `Rule` feature :

```rust
use type_rules::Rule;
use type_rules::Validator;

struct IsEven();

impl Rule<i32> for IsEven {
    fn check(&self, value: &i32) -> Result<(), String> {
        if value % 2 == 0 {
            Ok(())
        } else {
            Err("Value is not even".into())
        }
    }
}

#[derive(Validator)]
struct MyInteger(#[rule(IsEven())] i32);
```

## Rules list

Here a list of the rules you can find in this crate, 
if you want more details go to the rule definition.

Check the length of a `String` or `&str`:
- `MinLength`: Minimum length ex: `MinLength(5)`
- `MaxLength`: Maximum length ex: `MaxLength(20)`
- `MinMaxLength`: Minimum and maximum length ex: `MinMaxLength(5, 20)`

Check the range for anything that implements `PartialOrd<Self>` like all numeric/floating types
or dates with `chrono`:
- `MinRange`: Minimum range ex: `MinRange(5)`
- `MaxRange`: Maximum range ex: `MaxRange(20)`
- `MinMaxRange`: Minimum and maximum range ex: `MinMaxRange(5, 20)`

Check the size of a `Vec<T>` :
- `MinSize`: Minimum size ex: `MinSize(5)`
- `MaxSize`: Maximum size ex: `MaxSize(20)`
- `MinMaxSize`: Minimum and maximum size ex: `MinMaxSize(5, 20)`

others :

- `Opt`: Apply another rule to inner value of an `Option` ex: `Opt(MinMaxRange(1, 4))`
- `Validate`: Recursive checking ex: `Validate()`
- `All`: Rule to constrain a collection to valid the specified rule ex: `All(MinLength(1), "You can't use empty string")`
- `RegEx`: check if a `String` or `&str` matches the regex. 
  You need the `regex` feature to use it.
  ex: `RegEx(r"^\S+@\S+\.\S+")`
