+++
title = "Go use those Traits"
description = "The IO interfaces are one of the best things these languages have"
date = 2020-08-30
+++

**Interfaces** were popularized by Java but originally came from Objective-C
([source](https://stackoverflow.com/a/2758619/6266958)).Until a few days ago I
did not realize how powerful they were. In my opinion, interfaces have been
potrayed in an incomplete manner by all the tutorials. The key idea is correct
but it limits the thinking of what it can be used for. 

I am sure everyone who has written some form of object oriented language would
know this example.
```java
public interface Animal {
    public String speak();
}

public class Cat impements Animal {
    public String speak() { return "I am a God !"; }
}

public class Dog impements Animal {
    public String speak() { return "Where's my hooman !"; }
}
```
And this being used as follows:
```java
Animal[] animals = new Animal[2];
animals[0] = new Cat();
animals[1] = new Dog();

for (Animal a : animals) {
    a.speak();
}
```

This shows a very generic and boring use of interfaces. Accepting multiple types
to make a function generic or grouping a certain group of objects by an
interface type. You would have seen things like this.

```java
public interface Animal {}

public class Cat implements Animal {
    public void walk() { /* Some funny walk */ }
    public void beWeird()  { /* Need I say anything */ }
}

public class Dog implements Animal {
    public void beGoodDoggo() { /* Woof ! */ }
    public void bathe() { throws new RuntimeException(); }
}
```
These 2 classes have no common functionality, yet they share an interface. This
is just to mark these classes in a common group, logically. Useful and bit
intereting use too, but we have something better.

## How do you test a function that taken a file as a parameter

I am sure you would have written, at some point of time, a function that takes a
file path as in input and you do certain operations on it, maybe read to it, or
write to it and return. Something like this.

```java
public int findCharInFile(String path, char c) {
    /** 
    * Some crappy Java code to open a file that results in taking more 
    * effort than manually opening the file and doing a find on it by 
    * visual inspection.
    **/
    return Integer.MAX_VALUE;
}
```
How would you write tests for it ? You could do a dummy test file in your
project with the test strings. For a long time I thought that was an ok way.
Recently, I realized, there are better ways to implement it with higher
testability.

Sorry, too much of Java, let's write something better.

## What's problematic with that interface design


The above interface isn't wrong, but it has some problems with it.
- Testing becomes a problem, if you have more of those functions, your project
  will have a lot of stray files which are filled with different test cases.

- The function signature doesn't signify, it needs a file and not some random
  string. Arguably, it would be just made as `findCharInFile(File file, char c)`
  but it's a generalized perspective with all the languages.

- You don't know anything about the ownership of the file, what should you do
  once you get that File object.

- And the most fascinating, what do you do when it's not a regular file.

## How to solve the above problems

Let's look into a less constraint language which doesn't have such hardcode OOPs
principles. 

[golang](https://golang.org) is the new language for the writing webservers and
things on the internet. I am looking at you, NodeJS. 

I feel bringing JS to the backend was a mistake, or atleast not regulating it.
I have an imperative programming background. I have worked a bit on a few NodeJS
projects. For some reason, it didn't seem receptive to me and it had certain
inconsistencies that made it hard for me to write correct code. I know a lot of
the veterans (irrespective of their background) will disagree with me, but I
didn't feel that with other languages, so, just my opinion, no one needs to
agree to it.

Back to go, so, it's an incredibly well designed language with constructs that
are simple to use and most importantly, read. It looks minimalist yet has a
loaded standard library.

Go has 2 interfaces that has the power to change the way you write code.
They are:
- [`io.Writer`](https://golang.org/pkg/io/#Writer)
- [`io.Reader`](https://golang.org/pkg/io/#Reader)

There are other interfaces too that can help you write even cleaner and self
explanatory code, but let's start with this.

Let's see how the same function would look in go, but with these constructs.
```go
func findCharInFile(rd io.Reader, c char) int {
    contents, _ := ioutil.ReadAll(rd)
    return strings.IndexByte(contents, c)
}
```

This is piece of code has acheived the following things,
- The function signature clarifies exactly what it needs. The `io.Reader`
  interface requires exactly one implementation, that is `Read([]byte) (int,
  error)`. This function has no more bussiness than reading the file.

- This is much more unit-testable. All you need is a constructs that implements
  the `Read([]byte)` function. You can mock one up and send it to the function
  or use one of the builtins.
  eg:
  ```go
  func TestFindCharInFile(t *testing.T) {
    // Creates a *Reader type object which uses the supplied string as it's
    // backend and is Read only.
    rd := strings.NewReader("Contents of your test string")
    if findCharInFile(rd, '$') != -1 {
        t.Fail()
    }
  }
  ```

- This expresses the permissions you have on the stream. The `io.Reader` only
  has a `Read([]byte)` function, so you can't use the filename to anything else
  with it.

- It abstracts the underlying provider of the readable stream. It could be a
  file on the disk, an in-memory buffer, some key value store, even the network.
  It just has to adhere to one property or should have atleast on `trait`, to be
  able to Read out bytes to a bytes buffer.

- It also tells you the ownership of the underlying object. The caller is only
  only allowed to read the stream and not do anything else with it, like close
  the stream when it thinks it's done. Maybe other's a still reading it, that
  would cause a [panic](https://blog.golang.org/defer-panic-and-recover), 
  quite literally.

The same thing goes with `io.Writer`. It's only allowed to `Write([]byte) (int,
error)` and will not allow the user to close it.

### What if you wanted the user to close it.

If you want to hand over or delegate the control of the stream to the caller,
you can use one of the derived interfaces, [ReadCloser](https://golang.org/pkg/io/#ReadCloser).

```go
type ReadCloser interface {
    Reader
    Closer
}
```
You can hand this out and let the caller be able to close the stream. There are
other such interfaces that Go provides by default and you can create your own
too.

## Can Rust do it

We can't not talk about Rust in this case. The title mentions to use the 
[`trait`](https://doc.rust-lang.org/book/ch10-02-traits.html)
which is the Rust way of definiting features. Every trait can be seen as an
interface and whosoever implements the trait can be said to have implemented
that interface.

How would the same code look in Rust.
```rust
fn find_char_in_file(rd: &mut dyn Read, c: u8) -> Result<Option<usize>> {
    let mut buf = Vec::new();
    rd.read_to_end(&mut buf)?;

    for i in 0..buf.len() {
        if buf[i] == c {
            return Ok(Some(i));
        }
    }
    Ok(None)
}
```

Notice the definition, 
```rust
find_char_in_file(rd: &mut dyn Read, c:u8)
```

This implies that the receiver `rd` must implement the
[`io::Read`](https://doc.rust-lang.org/std/io/trait.Read.html) trait and that's all
is required for this function to work. Similarly it's easy to test too.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[derive(Default)]
    struct TestReader();

    impl Read for TestReader {
        fn read(&mut self, &mut &[u8]) -> Result<usize> {
            // Ignoring all the implementation details.
            Ok(0)
        }
    }

    #[test]
    fn test_find_char_in_file() {
        let mut rd = TestReader::default();
        let res = find_char_in_file(&mut rd, 'c' as u8).unwrap();
        assert_eq!(res, None);
    }
}
```

### What about buffering, I don't want to do that on my own.

The [`io::Read`](https://doc.rust-lang.org/std/io/trait.Read.html) trait takes
care of that. The buffered Reads are auto implemented traits given the `read()`
implementation. So, all you need is to write a simple `read` and rust will take
care of providing a more efficient Buffered read.

The same is possible in go, using the [`bufio`](https://golang.org/pkg/bufio)
package. If you want to imply to the caller that you need a more efficient read
implementation in your interface, you can simply replace the `io.Reader` with
`bufio.Reader` interface which implements buffering over an `io.Reader` for you.

So, it would become something like,
```go
func findCharInFile(rd bufio.Reader, c char) int
```
Converting a normal unbuffered reader to a buffered one is easy,
```go
bufRd := bufio.NewReader(rd)
```

## What did I learn

We haven't even talked about doing the same operations with an underlying
network connection and not a file or a device or an in-memory buffer. But it
would work just fine as network connections too are just Read and Write calls.

This helps simplify writing the code. Notice that those functions now no longer
have to deal with opening and closing files, handling errors that are file
related.

This explicitly conveys what the caller is responsible for and what can he do
with it, given that the API can do just what the interface allows.

Helps a ton with testability of the code, one of the "ilities" in software
architecture.

Don't get me wrong, It's not that all of it can't be done in Java. Java also 
has interfaces around `InputStreams` and `BufferedInputStreams`, `Readers` 
and `BufferedReaders` etc. All of this can be accomplished in a similar 
idiomatic and clean way. But's it's too much to write, Java is too verbose ;)

It's just a more useful outlook to interfaces.

## Where did I learn this from

Obviously, I didn't come up with all this. I have been working on a go project
and in order to write better and well organized code, I started looking around
for best practices and found some amazing talks about it. I will link them below
if someone wants to take a look (highly recommended).

- [7 common mistakes in go and when to avoid them by spf13](https://www.youtube.com/watch?v=29LLRKIL_TI)
- [How Do You Structure Your Go Apps](https://www.youtube.com/watch?v=oL6JBUk6tj0)
- [Go anti-patterns](https://www.youtube.com/watch?v=ltqV6pDKZD8)
