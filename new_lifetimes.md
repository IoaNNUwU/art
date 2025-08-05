I recently came up with an idea on how to make references easier to use and want to hear your feedback and ideas on this topic. This is not a Rust language proposal - I just want to explore some ideas.

**What is the problem with references?**

Lifetimes are checked by the borrow checker. There is no physical limitation on taking a second mutable reference or taking an immutable one when a mutable is already present. Lifetimes can also dynamically change depending on your code. For example, by adding a new `println!(my_ref)` statement at the end of your function, you are telling the borrow checker to automatically increase the lifetime of `my_ref` to include this line.

**Solution?**

Taking a reference to a value creates a new lifetime. What if instead of checking those scopes in the background and dynamically changing lifetimes after any code change, we declared them using a pair of curly braces?

Instead of:

```rust
fn substring(text: &str) -> &str { &text[0..5] }

fn main() {
    let text = String::from("Hello World");
    let substring: &str = substring(&text);
    println!(substring);
}
```

You would have this:

```rust
fn main() {
    let text = String::from("Hello World");

    with &text { // <-- clearly defined lifetime of this reference
        let substring: &str = substring(text);
        println!(substring);
    }
}
```

Using the `with` keyword, you define the lifetime of this reference. Note that the `text` variable has type `&str` inside this scope, which means you don't have access to the original `text: String` variable, and there is no way to take a mutable reference to this original variable.

- With this syntax, borrow checking mostly turns into a **check if all pairs of curly braces are matching** game.

The `with` keyword is the only way to create **new** references (and define a new lifetime). But what do I mean by **new** reference?

Consider this:
```rust
fn substring(text: &str) -> &str { 
    &text[0..5] // <-- This line doesn't create a new reference and new lifetime
}

struct User { id: u32, name: String }

impl User {
    fn get_name(&self) -> &str {
        &self.name // <-- This line doesn't create a new reference and new lifetime
    }
}
```

The `&` operator in Rust doesn't always create a new reference and new lifetime. Auto-dereferencing fields behind a reference is the default behavior. This means you have to use `&` to get back to reference form, but in essence `&self.name` offsets the existing `&self` pointer without creating a new lifetime. This means the majority of the code after this addition stays the same.

*Unfortunately*, not all code stays the same. The worst syntax hit is methods. The basic idea is to disallow the creation of arbitrary new references, which means you cannot simply call methods on an owned structure.

```rust
struct User { id: u32, name: String }

fn main() {
    let user = User { id: 10, name: String::from("Hello World") };

    with &user { // define new lifetime, then call `get_name`
        let name: &str = user.get_name();
        println!("{}", name);
    }
    
    // This is not allowed
    let name = user.get_name();
}
```

One exception would be methods that don't return references. For example, `Vec::capacity()` creates a new lifetime when called on an owned vec as it takes `&self` as an argument, but this lifetime is so short it cannot possibly collide with anything, so `with` syntax is unnecessary in this case.

Another example using iterators, default Rust code:

```rust
fn main() {
    let strings: Vec<String> = vec!["hello", "world", "rust", "programming"].iter().map(|s| s.to_string()).collect();

    let st: Vec<&str> = strings.into_iter()
        .filter(|s: &String| s.len() > 4)
        .map(|s: String| &s) // does not compile - cannot return data owned by the current function
        .collect();

    println!("{:?}", st);
}
```

Same example using the `with` keyword:

```rust
fn main() {
    let strings: Vec<String> = vec!["hello", "world", "rust", "programming"].iter().map(|s| s.to_string()).collect();

    // .into_iter() consumes self which means there is no need for new lifetime and `with` usage
    let st: Vec<&str> = strings.into_iter()
        .filter(|s: &String| s.len() > 4)
        .map(|s: String| with &s { s }) // semantically obvious why you cannot return s here
        .collect();                     // as s lives inside this defined scope

    println!("{:?}", st);
}
```

Example using `.iter_mut()`:

```rust
fn main() {
    let mut strings: Vec<String> = vec!["hello", "world", "rust", "programming"].iter().map(|s| s.to_string()).collect();

    // `.iter_mut()` does not consume self which means we have to use `with` 
    // to define new lifetime and then call `iter_mut`
    with &mut strings {
        let st: Vec<&mut String> = strings.iter_mut()
            .filter(|s: & &mut String| s.len() > 4)
            .map(|s: &mut String| {
                s.insert(3, '#');
                s
            })
            .collect();

        println!("{:?}", st);
    }
}
```

As you can see in the examples above, the only problematic place is the creation of a new reference. If you already have a reference (for example, you got it as an argument in the function definition), you can just use it as always.

One more example:

```rust
fn main() {
    println!("Please enter your name:");

    let mut name = String::new();

    io::stdin().read_line(&mut name).expect("Failed to read line");

    let trimmed_name = name.trim();
    println!("Hello, {}!", trimmed_name);
}
```

Becomes:

```rust
fn main() {
    println!("Enter your name:");

    let mut name = String::new();

    with &mut name {
        io::stdin().read_line(name).expect("Failed to read line");
    }

    with &name {
        let trimmed_name = name.trim();
        println!("Hello, {}!", trimmed_name);
    }
}
```

- In my opinion, it's easier to reason about lifetimes with this syntax change. What do you think?

### Syntax sugar

Let's see how this syntax translates to Rust.

```rust
let value: Type = .. ; // owned value

with &value { // value: &Type
    // Code using reference
}
with &mut value { // value: &mut Type
    // Code using mutable reference
}
```

Can be represented like this in Rust:

```rust
let value: Type = .. ; // owned value

{ // define new scope and shadow value
    let value: &Type = &value;
    // Code using reference
}

{
    let value: &mut Type = &mut value;
    // Code using mutable reference
}
```

So yes, you can do something really similar in Rust. Creating well-defined scopes for your references is considered a really good practice. My idea is to force this scope creation for every new reference and force-shadow the owned value in this scope (which also means during the lifetime of this reference). This gives real meaning to borrow checking rules. Inside this scope, you cannot use a mutable reference nor an owned value. **By force-shadowing its name, you physically disallow the user from using references in the wrong way** and not by some set of borrow-checker rules.

Also, note that this change simplifies the way you follow existing borrowing rules and doesn't change them in any way. You cannot create multiple mutable references or mutable and immutable references simultaneously with this new syntax, as in Rust. The only difference is how those rules are enforced on the user—by the borrow checker in Rust and by semantics in my examples.

### No more lifetimes?

Consider this example:

```rust
fn trim<'a, 'b>(text: &'a str, len: &'b str) -> &'a str {
    let len: usize = len.parse().unwrap();
    &text[0..len]
}
```

The Rust compiler forces lifetime usage in this example. The `&'a str` return type depends on the first argument with the `'a` lifetime. You might think, this information is only necessary in conventional borrow-checking. And what I mean by that is you have to analyze lifetimes inside functions to understand which depends on which to define final lifetimes in the outer function. But if those scopes are already defined by `with {}` blocks, you have a guarantee that none of those references can escape this scope, which means it's not important on which exact lifetime the returned one depends. 

Rust example:

```rust
fn main() {
    let len = 10;
    let text = String::from("Hello World");

    let trimmed = trim(&text, &len);

    len += 1; // it is ok to modify or drop len because `trimmed` doesn't depend on it
    // drop(text);  <-- cannot move out of text because it is borrowed

    println!("{}", trimmed);
}
```

With new `with` syntax:

```rust
fn main() {
    let len = 10;
    let text = String::from("Hello World");

    with &text {
        with &len {
            let trimmed = trim(text, len);
            
            // len += 1;  <-- You cannot modify len here
            // drop(text);  <-- text has type &String, original value is shadowed, no way to drop it

            println!("{}", trimmed);
        }
    }
}
```

Note that this trick is only possible because you cannot physically get access to the original value, which means you don't need to calculate intersections between this lifetime and, for example, a mutable one. `with` guarantees there are no other references to the same value in its scope.

But it is reasonable to expect to be able to return `trimmed` from `with &len` scope because trimmed only depends on `&text`:

```rust
fn main() {
    let len = 10;
    let text = String::from("Hello World");

    with &text {
        let trimmed = with &len { 
            let trimmed = trim(text, len);
            
            // because len has type here &i32 you cannot modify it here
            // len += 1  <-- This is not allowed

            trimmed
        }
        len += 1 // You can modify len here because `with &len` scope ended
        println!("{}", trimmed);
    }

    // Or like this - you can create `&len` without `with` keyword because trim's return type doesn't depend
    // on it which means this lifetime is very short.
    with &text {
        let trimmed = trim(text, &len);
        len += 1 // You can modify len here because trimmed doesn't depend on len
        println!("{}", trimmed);
    }
}
```

Also good example of why lifetimes are still neccesary is if the first argument to this function is `'static`, then it's reasonable to expect to be able to return this value from function as if it was the owned value.

### Conclusion

What do you think about this? Did I miss something obvious and it cannot possibly work? Do you think its easier to understand lifetimes if they're clearly defined by pair or curly braces?
