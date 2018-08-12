---
title: "Modern C++ for C Programmers: Part 6"
date: 2018-07-26T14:35:51+02:00
draft: true
---
**NOTE**: If you like this stuff, come work with me over at PowerDNS -
[aspiring C++ programmers
welcome](https://www.powerdns.com/careers.html#securityDeveloper)!

In [part 5](../cpp-5) we discussed smart pointers, placement
new and the powerful `move constructor`. As you may have gathered by now,
parts 1 through 5 were a pitch to sell modern C++ to existing C specialists.
To do so, I tried to show the best and most immediately useful parts of C++.

As noted [earlier](../cpp-intro), no language is perfect, and not all
features of C++ are as good or as spectacular.  In this final part of
'[Modern C++ for C Programmers](../cpp-into)' you will find some of the
'best of the rest'.

Static assert
=============
Sometimes our code depends on certain things being true, like for example a
`struct` being exactly 16 bytes long. The C preprocessor is stil available
in C++, but by design the preprocessor can't execute code.
`static_assert` can:

```
  static_assert(sizeof(int) == 4, "Int must be 4 bytes");
```

Static asserts are evaluated at compile time and leave no trace in the
resulting object code. Static_asserts are great to test generic assumptions
but truly shine when writing templatized code, where one of the template
parameters for example must be a pointer:

```
#include <type_traits>

template<typename T>
bool isSet(const T& t)
{
  static_assert(std::is_pointer<T>(), "isSet requires a pointer");
  return t != nullptr;
}
```

Static asserts not only prevent bad things from compiling, they can also
enable code to self-document its intended use, turning a screen full
of compilation errors into a friendly message.

Note that this code uses the `type support` function `std::is_pointer`. The
library has [far more tests](https://en.cppreference.com/w/cpp/types) that
can be useful to write robust code.

constexpr
=========
A `constexpr` is a piece of code that can be executed entirely at compile
time, leaving its result ready to use in the object code.  This makes
compiling slower, but the resulting executable faster (and smaller). It also
feels very neat to get essentially static things done at compile time.

Practically speaking, it can also frequently prevent the need to
autogenerate large sets of #defines for example - they can be calculated at
compile time. 

A simple example, a compile time `htonl` for use on little-endian systems:

```
constexpr uint32_t chtonl(uint32_t s)
{
    return 
        ((s & 0x000000FF) << 24) | ((s & 0x0000FF00) << 8)
      | ((s & 0xFF000000) >> 24) | ((s & 0x00FF0000) >> 8);
}
```

With some further work, it is possible to make a version that does the right
thing on big-endian platforms as well.

The [frozen](https://github.com/serge-sans-paille/frozen) library is a good
example of what is possible at compile time:

```
#include <frozen/set.h>

constexpr auto avoidPorts = frozen::make_set<int>({25, 53, 80, 110, 443});

..

do {
	port = random() & 0xffff;
} while (avoidPorts.count(port));
```

The `avoidPorts set` is populated (and sorted!) at compile time. 

`constexpr` further enables really `const` symbols that still show up in
coredumps and backtraces - which is far superior to using a #define.

`constexpr` does not allow for arbitrary code to be executed at compile time
because (understandably) there are a lot of constraints to make sure the
output is really really const. C++17 has some additional `constexpr`
features that make this easier.

Numeric limits
==============
To find limits on numerical types,
[`std::numeric_limits`](https://en.cppreference.com/w/cpp/types/numeric_limits),
can be used like this:

```
#include <limits>
int i;
cout << std::numeric_limits<int>::min() << endl;
cout << std::numeric_limits<decltype(i)>::max() << endl;
```

This prints the minimum and maximum allowed values of an `int`. Note that in
the last line, we use `decltype` to get the type of i. `decltype` is
sometimes necessary to make templates work. It does look a bit ugly but it
leaves no trace in the resulting object code.

`std::numeric_limits<>` offers access to `epsilon` and many other attributes
of a type in a uniform way that is way easier than looking up #defines like
`ULLONG_MAX`.

You will not be surprised to learn that in modern C++, all the
[`std::numeric_limits`](https://en.cppreference.com/w/cpp/types/numeric_limits) methods are `constexpr`.

iostreams
=========
As mentioned in earlier parts, the C++ iostreams should not be seen as a
replacement for either POSIX or C streams based i/o. The C++ iostreams are
not very performant and can be utterly frustrating to work with. They still
have their uses however. 

```
#include <fstream>

...
ifstream in("/etc/passwd");
if (!istrm.is_open()) {
  std::cout << "failed to open\n";
} else {
  string line;
  int count = 0;
  while(getline(in, line)) 
    ++count;
  cout << "Read " << count << " lines\n";
}
```

If all you need to do is reading a (small) numbers of lines of text,
iostreams are just fine. In terms of formatting output, this is by far not
as easy as `fprintf`, and it is very easy to mess up even, since `iostreams`
carry state. So if you modify the output of floats, you do so for
subsequent floats too.

A useful component of the `iostreams` library is `ostringstream` which
allows for the easy generation of strings. This a `iostream` that creates a
string:

```
ostringstream msg;
msg << "The current temperature is " << temp << " degrees";
std::string status = msg.str();
```

However, frequently it may be easier to do:

```
string status = "The current temperature is " + to_string(temp);
status += " degrees";
```

[`std::to_string`](https://en.cppreference.com/w/cpp/string/basic_string/to_string)
is a useful function in general. C++17 offers a very high performance
allocation free variant that is locale independent called
[`std::to_chars`](https://en.cppreference.com/w/cpp/utility/to_chars).

To get `printf`-like behaviour, consider using
[`boost::format`](https://www.boost.org/doc/libs/1_67_0/libs/format/doc/format.html)
which enables things like:

```
cout << boost::format("Temperature is %.02f\n") % temp;
```

Measuring time
==============
C++ is used heavily in particle physics, powering the foundational
[`ROOT`](https://root.cern.ch/) library which helped discover the Higgs
Boson and much more.  Particle physicists have very exacting needs in terms
of timing, and in somewhat of a mixed blessing, they found a willing ear
over at the C++ standards committee.

As a result, the C++ infrastructure for measuring times and durations is.. 
extravagant.  Getting the current time(stamp) is not a cheap operation these
days (in any language), since different cores or CPUs may be running at
different speeds.  Depending on your application, you may prefer a fast but
potentially non-monotonous timesource over a slow but always correct one.

C++ currently offers us access to `system_clock`, `steady_clock` and
`high_resolution_clock`. In the near future, this will be expanded to
`gps_clock`, `tai_clock` and `utc_clock`, which seems to indicate the
astronomers managed to infiltrate the C++ standards committee as well.

If all you want to do is measure how long something took, this is useful:

```
  using namespace std::chrono;

  auto start = system_clock::now();
 
  // do something

  auto end = system_clock::now();
 
  duration<double> elapsed_seconds = end-start;
  std::time_t end_time = system_clock::to_time_t(end);

  std::cout << "finished computation at " << std::ctime(&end_time)
            << "elapsed time: " << elapsed_seconds.count() << "s\n";
```

Note that it is also possible to get a duration in 'nanoseconds' expressed
as an integer, or even in dedicated particle physics units like
[`shakes`](https://en.wikipedia.org/wiki/Shake_(unit)). 
Details are [here](https://en.cppreference.com/w/cpp/chrono/duration).

Scoped enums
============
I don't know about you, but I've never been happy that `enum`s, despite
having a name, project into the scope of where they were defined.  In other
words, there can only be one global `enum` that defines `BLUE`.

Modern C++ has `scoped enum`s, also known as `class enum`s:

```
enum class Color { red, green = 20, blue };

Color r = Color::blue;

switch(r)
{
    case Color::red  : std::cout << "red\n";   break;
    case Color::green: std::cout << "green\n"; break;
    case Color::blue : std::cout << "blue\n";  break;
}

// int n = r; // error: no scoped enum to int conversion
int n = static_cast<int>(r); // OK, n = 21
```

Note how these enums populate names within their own name - `Color::red`
versus `red`.

This example is from the most excellent
[cppreference.com](https://en.cppreference.com/w/cpp/language/enum) site.

Modern `enum`s are typesafe and better in almost every way compared to the
classic ones.

Initializer lists
=================
Initializer lists are remarkably simple yet powerful constructs. They have
appeared in a few earlier examples, and they power constructions like:

```
  vector<int> vi{1, 2, 3, 4};
  vector<string> vs{"foo", "bar", "baz"};
```

When the compiler sees a `braced-init-list` in the right context, it will
convert it into a `std::initializer_list`. This is a lightweight proxy
container meant to pass lists around without copying them. It is not a
storage mechanism itself.

In the two examples above, `std::vector` has a constructor that accepts
`initializer_list`s compatible with the ones we created. 

A neat use is within range-for iteration:

```
  for(auto a : { 1, 10, 15, 25} )
    printAverages(a);
```

Or, if we have a bunch of vectors which all need to be averaged:

```
  vector<double> x, y, z;
  for(auto& a : {x, y, z})
    doAverage(a);
```



Regular expressions, raw strings
================================
> Some people, when confronted with a problem, think "I know, I'll use
> regular expressions." Now they have two problems. -- jwz

Despite this, regular expressions have their uses, and they are a standard
part of many other languages too.

Again an example adapted from the excellent [cppreference.com
site](https://en.cppreference.com):

```
  std::string s = R"(Some people, when confronted with a problem, think: 
"I know, I'll use regular expressions."
Now they have two problems.)";

```

Here we define a string to search. Note that I used the new C++ 'raw' string
capability, whereby you start a string with `R"(` and end it with `)"`,
and in between everything gets copied. If you are worred about `)"`
appearing in your string, you can also do:

```
  std::string s = R"foo(Some people, when confronted with a problem, think: 
"I know, I'll use regular expressions."
Now they have two problems.)foo";
```

We can now regex to our heart's content:

```
  std::regex word_regex("(\\S+)");
  auto words_begin = 
        std::sregex_iterator(s.begin(), s.end(), word_regex);
  auto words_end = std::sregex_iterator();
 
  std::cout << "Found "
            << std::distance(words_begin, words_end)
            << " words\n";

  for_each(words_begin, words_end, [](const auto& w) {
    cout << "Word: " << w.str() << endl;
  });
```

Note that regular expressions aren't tied to `std::string`, they work on any
collection of characters. The result of a regular expression is exposed as a
regular iterator, so it can also be traversed like that. 

[C++ regular expressions](https://en.cppreference.com/w/cpp/regex) can
match, search and replace, and support various regex dialects and case
sensitivity options.

Summarizing
===========
I sincerely hope that these posts have given you a new appreciation of
(modern) C++, and that if you make the jump into C++, this will make you a
better programmer. I also hope you will have fun using this beast of a
language. 

Books to read
=============
If you liked what you read about modern C++, I recommend the following books
to get started:

 * [A Tour of C++ (2nd edition,
   2018)](https://smile.amazon.com/Tour-2nd-Depth-Bjarne-Stroustrup/dp/0134997832/)
   by Bjarne Stroustrup. In 256 pages, this provides a cogent introduction
   to C++ 2014.
 * Effective C++ - lists common pitfalls, wonderful features and problematic areas to avoid. Most recently updated in 2005.
 * More Effective C++ - more of the same, but getting a bit dated. Still worth your while.
 * Effective STL - last updated in 2001, and like Effective C++, but then with a stronger focus on the standard library.
 * Effective Modern C++ - not the best of the 'effective' series, but has a
   lot of wisdom.
 * 'C++ Primer' by Stanley Lippman cs. The most recent edition is updated for C++2011.
 * The C++ Programming Language (4th edition, Bjarne Stroustrup)
   An astounding book. It has wisdom on every page. It also has 1400 pages. A
   [hilarious review](http://www.theregister.co.uk/2013/06/17/verity_stob_is_bjarne_again/) by Verity Stob can be found on The Register. In TCPL we read "C is not a big language, and it is not well served by a big book.". Hence the 1400 pages for C++. Even though the book is far too huge to read from cover to cover (although I think I've now hit almost every page), everyone should have a copy. It does indeed explain all of C++, and it does so pretty well. The book also is strong on C++ idioms that will make your life easier. 

