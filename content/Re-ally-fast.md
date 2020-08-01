+++
title = "Re-ally fast"
description = "The praise ripgrep deserves"
date = 2020-07-29
+++

Lately I have been trying to get into the [Rust](https://www.rust-lang.org)
ecosystem to get a feel of the tools and the language. In the process I came
across a few tools that I would highly recommend to try out.

- [ripgrep](https://github.com/BurntSushi/ripgrep) - A blazing fast alternative to GNU grep
- [alacritty](https://github.com/alacritty/alacritty) - A GPU accelerated terminal emulator
- [exa](https://github.com/ogham/exa) - A replacement for `ls` written in Rust

The tools `exa` is for a more pretty `ls`, may not appeal to everyone. Similar is the case
for `alacritty`, it is intended to be used by people who are comfortable with `tmux` as it
doesn't have tabs, kind of a bummer but if you roll with tmux, it should not make any
difference anyways.

The tool that I am excited about is, `ripgrep` which is a crazy fast alternative for GNU grep.
It has a lot more features than grep and the speed it chugs through text using regular expressions
is amazing.

A more detailed and in-depth explanation and benchmarks can be found at
[ripgrep is faster than {grep, ag, git grep, ucg, pt, sift}](https://blog.burntsushi.net/ripgrep/)
which is blog entry from the creators.

## Some numbers

Let's first see ripgrep in action (so that there is some substance to my claims).

![Destruction](/ripgrep.png)

The above test is to do a case insensitive search on a text file which is close to
1.2GB of size. The `grep` took more than 24s while `ripgrep` casually completed the search within
half of a second which is just pure destruction in terms of speed. The `shasum` part is to make
sure the output of both the tools are correct. I would definitely trust `grep` for it's correctness
and the above screenshot shows both produce the same output hashes, which means `ripgrep`'s output is
also correct.

Unless I stumbled upon a pair of texts that tend to produce the same [SHA-256](https://en.wikipedia.org/wiki/SHA-2) hash.
[SHAttered](https://shattered.io/) 256 times bada**!.

## What's up with ripgrep ?

### Restricted but fast regex library
A portion of the speed gain comes from not complying to the [PCRE](https://en.wikipedia.org/wiki/Perl_Compatible_Regular_Expressions)
standards which supports all features including look-ahead and look-behind features.

eg: If you try the following regex in Rust, it fails
```rust
extern crate regex; // 1.3.9

use regex::Regex;

fn main() {
    let haystack = "world's";
    let re = Regex::new(r"\w+(?<!s)\b").unwrap();
    println!("{}", re.is_match(haystack));
}
```
with the following error
```
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
regex parse error:
    \w+(?<!s)\b
       ^^^^
error: look-around, including look-ahead and look-behind, is not supported
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

On the other hand, python3 implements look-ahead and look-back syntax.
```py
Python 3.7.4 (default, Oct 12 2019, 18:55:28)
[Clang 11.0.0 (clang-1100.0.33.8)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import re
>>> t = re.compile(r"\s\w+(?<!s)\b")
>>> t.findall("hello world's")
[' world']
```

### Better optimizations to the literal string comparision
Regex being a general purpose search mechanism can be slow and can be beaten by classical
string search algorithms like [Boyer-Moore](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string_search_algorithm).

The regular brute-force string searches or the python `in` keyword work by traversing the haystack until it finds
the first character of the needle, then it extracts the substring of the length of the needle and then matches. If
the string does not match, increment to the next char in the haystack and continue.
```py
>>> needle="rust"
>>> haystack="searching for rust lang"
>>> for i in range(len(haystack) - len(needle) + 1):
...     if haystack[i: i + len(needle)] == needle:
...             print ("Found at", i)
...
Found at 14
```

This can be seen as aligning the needle in all possible ways against the haystack and then
comparing to see which one matches perfectly. Something like
```
A N P A N M A N -
P A N - - - - - -
- P A N - - - - -
- - P A N - - - -
- - - P A N - - -
- - - - P A N - -
- - - - - P A N -
```

What Boyer-Moore does is, with some pre-computation magic, it will try to find
the best positions that needs to actually be tested. It does so by finding the
byte offsets at which the last byte of the needle matches the haystack. If the
last byte does not match, there is no point visiting the chars before that byte
offset in that alignment. This search of "candidates" is done by [`memchr`](https://man7.org/linux/man-pages/man3/memchr.3.html)
which can find the first position of a char in a memory area. The fancy thing
about `memchr` is that, it compiles down to [SMID](https://en.wikipedia.org/wiki/SIMD)
instruction.

This means that it can find the _search candidates_ really fast due to vectorized
instructions. The `regex` crate gwill go to great lengths to extract literal strings
from the regular expressions and then find _search candidates_ in order to minimise
the regex search time.

Most of the tools search for a needle in a haystack that is made up of lines of text,
i.e. delimited lines. The `regex` crate will find the lines that may match and
then run the full regex search on it saving a lot of time from the regex overhead.

For multiple literals, i.e. `"you|me"`, the `regex` crate will use plain Aho-Corasick
or a vectorized algorithm called [Teddy](https://github.com/BurntSushi/aho-corasick/blob/master/src/packed/teddy/runtime.rs)
when enabled.

To be honest, I don't understand a lot of it. I will try and implement the Boyer-Moore algorithm
someday maybe to get a better understanding of what is going on.

### Substring search algorithms
- The [z-algorithm](https://cp-algorithms.com/string/z-function.html) is a pretty good choice
which requires `O(n + m)` worst case time and space if the needle and the haystack don't change.

- If there are a lot of needles to search from a small number of haystacks, probably a
[Trie](https://en.wikipedia.org/wiki/Trie) would be more suited. This will not scale
for a large number of haystacks as all of the tries will be needed to be present in memory
at the same time which is not possible. For example, in the above 1.2G exmaple, having
that amound of storage occupied all the time may not be necessary.

- If the needle isn't changing and there are a lot of haystacks, some algorithm that
pre-processes the needle might help, just like Aho-Corasick or Boyer-Moore.

### Disclaimer
> All of this is shamelessly ripped off of the [Burntsushi/ripgrep](https://blog.burntsushi.net/ripgrep/)
blog and it's highly recommended to read that if you need a clearer picture.
