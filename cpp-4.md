---
title: "Modern C++ for C Programmers: Part 4"
date: 2018-07-12T13:07:40+02:00
draft: true
---
**NOTE**: If you like this stuff, come work with me over at PowerDNS -
[aspiring C++ programmers
welcome](https://www.powerdns.com/careers.html#securityDeveloper)!

Welcome back!  In [part 3](../cpp-3/) I discussed classes, polymorphism,
references and templates, and finally built a source indexer out of basic
containers that achieves 60MB/s indexing speed.

In this part we continue with further C++ features that you can use to spice
up your code 'line by line', without immediately having to use all 1400
pages of 'The C++ Programming Language'. There is frequent reference to the
indexing example from [part 3](../cpp-3) so you may want to make sure you
know what that is about.

Various code samples discussed here can be found on
[GitHub](https://github.com/ahuPowerDNS/hello-cpp).

If you have any favorite things you'd like to see discussed or
questions, please hit me up on
[@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert) or
bert.hubert@powerdns.com

Lambdas
=======
We have previously encountered these weird looking snippets, for example
like this:

```
std::sort(vec.begin(), vec.end(), 
          [](const auto& a, const auto& b) { return a < b; } 
     );
```

Although lambdas are, in essence, syntactical sugar, their availability has
made modern C++ a far more expressive language. In addition, as noted in
[part 1](../c++-1/), passing code snippets as function pointers severely
restricts what the compiler can to do optimize code. So not only do
lambdas lead to fewer lines of code, the resulting binaries can be faster
too.

C++ lambdas are first class citizens and are compiled just like normal code.
Scripting languages can easily do lambdas since they come with a runtime
interpreter, C++ has no such luxury. So how does it all work?

Here is the anatomy: ``[capture specification](parameters) { actual code
}``. The capture specification can be empty which means the code in the
lambda only 'sees' global variables, and this is a very good default.
Captures can be by value or by reference. In general, if a lambda needs to
capture a lot of detail, ponder if it is still a lambda.

For the parameters, you will very frequently use `auto` there, but it is in
no way mandatory. 

The actual code is then between `{` and `}`, where the only special thing is
that the return type is derived automatically, but you can also override it
if you know what you are doing. A worked example:

```
vector<string> v{"9", "12", "13", "14", "18", "42", "75"};

string prefix("hi ");

for_each(v.begin(), v.end(), [&prefix](const auto& a) {
	cout << prefix + a << endl;
}); // outputs hi 9, hi 12, hi 13 etc
```

The first line employs the fun initializers that allow modern C++ to quickly
fill containers.  The second line creates a prefix string.  Finally the
third line uses the C++ algorithm `for_each` to iterate over the container.

The `prefix` variable is 'captured by reference'.  For passing the
parameters, `const auto& a` could also have been `const std::string&`. 
Finally we print the prefix and the container member.

To sort this vector of strings numerically, we could do:

```
 std::sort(v.begin(), v.end(), [](const auto& a, const auto& b)
            {
              return atoi(a.c_str()) < atoi(b.c_str());
            });
```

A lambda creates an actual object, albeit one of unspecified type:

```
  auto print = [](const vector<std::string>& c) {
    for(const auto& a : c)
        cout << a << endl;
  };

  cout<<"Starting order: "<<endl;
  print(v);
```

We have now stored a lambda in `print`, and we can pass this around and use
it later on too. But what **is** `print`? If we ask a debugger, it may tell
us:

```
(gdb) ptype print
struct <lambda(const std::vector<std::string>&)> {}
```

Depending on what gets captured, the type becomes ever more exotic. It is
for this reason that lambdas are usually passed via `auto` or with generics.

When there is a need to store a lambda, or anything callable actually, there
is `std::function`:

```
  std::function<void(const vector<std::string>&)> stored = print;

  stored(v); // same as print(v)
```

Note that we can also do this:

```
void print2(const vector<string>& vec)
{
	// ..
}

...

std::function<void(const vector<std::string>&)> stored = print2;
```
`std::function` can store other callable things too, like objects with
`operator()` defined.  The downside of `std::function` is that it is not as
fast as calling a function or invoking a lambda directly , so when possible,
try doing that.

A lambda used inside a class can capture `[this]` which means it gains
access to class members. 

To further promote C interoperability, a lambda decays into a plain C
function pointer if it doesn't capture anything, leading to the ability to
do this:

```
  std::vector<int> v2{3, -1, -4, 1, -5, -9, -2, 6, -5};

  qsort(&v2[0], v2.size(), sizeof(int), [](const void *a, const void* b)
        {
          if(abs(*(int*)a) < abs(*(int*)b))
            return -1;
          if(abs(*(int*)a) > abs(*(int*)b))
            return 1;
          return 0;
        });

```

In general, lambdas are awesome but best used for small, inline, constructs.
If you find yourself capturing lots of stuff, you may actually be better off
using a
[functor](https://www.cprogramming.com/tutorial/functors-function-objects-in-c++.html),
which is a class instance you can call (because it has overloaded
`operator()`).

Expanding our indexer
=====================
In the indexer from [part 2](../cpp-2) we ended up with:

```
struct Location
{
	unsigned int fileno;
	size_t offset;
};

std::unordered_map<string, vector<Location>> allWords;
```

This contains an unordered list of all words found in the indexed files,
plus per word a `vector` of `Locations`s where the word was found. We used
an unordered map since this is 40% faster than an ordered one. 

However, if we want to perform lookups for things like "main*", to match
everything that begins with "main", we also need an ordered list of words:

```
  std::vector<string> owords;

  owords.reserve(allWords.size()); // saves mallocs

  for(const auto& w : allWords) 
    owords.push_back(w.first);

  sort(owords.begin(), owords.end()); 
```

Note how this uses the range-for construct to insert only the keys of the
`allWords` map into a vector, as yet unsorted, which we remedy in the final
line.

Interestingly enough, we do not lose out on our 40% speedup since 'sort once
we are done' is faster than keeping everything sorted all the time.

Should we be in the mood, we could attempt to be smarter. As written above,
every word is now present in memory twice, once in `allWords`, once in
`owords`. 

It is a pretty C like thing to not do this and live on the edge for a bit:

```
  std::vector<const string*> optrwords;
  optrwords.reserve(allWords.size());

  for(const auto& w : allWords)
    optrwords.push_back(&w.first);

  sort(optrwords.begin(), optrwords.end(),
       [](auto a, auto b) { return *a < *b;}
       );
```

With this code, we store `const` pointers to the keys in the `allWords`
unsorted map. Then we sort `optrwords`, containing pointers, using a lambda
that dereferences these pointers.

If we index the Linux source tree, which contains around 600,000 unique
words, this does save us around 14 megabytes of memory, which is nice.

The downside however is that we are now storing raw pointers straight into
another container (`allWords`). As long we don't modify `allWords` this is
safe. And for some containers, it is even safe if we do make changes. This
happens to be the case for
[`std::unordered_map`](https://en.cppreference.com/w/cpp/container/unordered_map),
as long as we don't actually delete an entry we store a pointer to.

I think this illustrates a key point of using modern C++.  You can shave 14
megabytes of memory 'if you know what you are doing', but I highly recommend
that you only reach into this 'C' like bag of tricks if you really need to.
But if that is the case, it is good to know it can be done.

Containers and algorithms
=========================
We have seen a variety of
[containers](https://en.cppreference.com/w/cpp/container) so far (`std::vector`,
`std::unordered_map` for example).  In addition, there is a raft of
algorithms that can operate on these containers.  Crucially, through the use
of templates, algorithms are actually completely decoupled from the
containers they operate on.

This decoupling has enabled the standard to specify a larger than usual
amount of generically useful algorithms. We've already encountered
`std::for_each` and `std::sort`, but here's a more exotic one:
`std::nth_element`. 

Going back to our indexer, we have a list of words and how often they occur.
Let's say we want to print the 20 most frequently occurring ones, we'd
normally take the whole list of words, sort them in order of frequency and
then print the top 20.

With `std::nth_element`, we can actually get what we want. First, let's
gather the data to sort, and define the comparison function:

```
  vector<pair<string, size_t>> popcount;
  for(const auto& w : allWords)
    popcount.push_back({w.first, w.second.size()});

  auto cmp = [](const auto& a, const auto& b)
       {
         return b.second < a.second;
       };
```

We are defining a vector containing `pair`s.  A pair is a convenient
templated struct containing two members, called `first` and `second`.  I
find that `pair` inhabits a sweet spot that is quite useful, an 'anonymous
struct' with well known names.  Confusion occurs when pairs get nested into
pairs, or when using `std::tuple`, which is `std::pair` on steroids. Beyond
two simple members, create a struct with named members.

The range-for loop shows one new feature, 'brace initialization', which
means `w.first` and `w.second.size()` (which is the number of occurrences of
this word) are used to construct our pair. It saves a lot of typing.

Finally, we define a comparison function and call it `cmp` so we can reuse
it. Note that it compares in reverse order.

Next up, the actual sorting and printing:

```
  int top = std::min(popcount.size(), (size_t)20);
  nth_element(popcount.begin(), popcount.begin() + top, popcount.end(), cmp);
  sort(popcount.begin(), popcount.begin() + top, cmp);
  
  int count=0;
  for(const auto& e : popcount) {
    if(++count > top)
      break;
    cout << count << "\t" << e.first << "\t" << e.second << endl;
  }
```

This invocation of `std::nth_element` bears some explanation. As noted,
`iterator`s are places within a container. `begin()` is the first entry
and, consistently, `end()` is one beyond the last entry. On an empty
container, `begin() == end()`. 

We pass three iterators to `nth_element`: where to begin the sorting, what
the cutoff is for our 'top 20' and finally the end of the container. What
`nth_element` then does is make sure that the entire top-20 is in fact in
the first 20 positions of the container. It does not however guarantee that
the top-20 itself is sorted. For this reason we do a quick sort of the first
20 entries.

The final 6 lines print the actual top-20, in the correct order.

C++ comes with [many useful
algorithms](https://en.cppreference.com/w/cpp/algorithm) that allow you to
compose powerful programs.  For example,
[`std::set_difference`](https://en.cppreference.com/w/cpp/algorithm/set_difference),
[`std::set_intersection`](https://en.cppreference.com/w/cpp/algorithm/set_intersection)
and
[`std::set_symmetric_difference`](https://en.cppreference.com/w/cpp/algorithm/set_symmetric_difference)
make it trivial to write 'diff' like tools or find out what changed from one
state to another.

Meanwhile, algorithms like
[`std::inplace_merge`](https://en.cppreference.com/w/cpp/algorithm/inplace_merge) and
[`std::next_permutation`](https://en.cppreference.com/w/cpp/algorithm/next_permutation) may prevent
you from having to whip out the Knuth books.

Before doing any kind of data manipulation or analysis, I urge you to [go
through the list](https://en.cppreference.com/w/cpp/algorithm) of existing
algorithms, you will likely find things there that get you most of the way.

Looking things up
=================
As an example, recall that we made a sorted list of words so we could do
prefix lookups. All words ended up in `std::vector<string> owords`. We can
interrogate this flat (and hence very efficient) container in several ways:

 * [`std::binary_search(begin, end,
   value)`](https://en.cppreference.com/w/cpp/algorithm/binary_search) will let
  you know if a value is in there.
 * [`std::equal_range(begin, end,
value)`](https://en.cppreference.com/w/cpp/algorithm/equal_range) returns a
   `pair` of iterators which span all exactly matching entries. 
 * [`std::lower_bound(begin, end,
value)`](https://en.cppreference.com/w/cpp/algorithm/lower_bound) returns an
   iterator that points to the first place `value` could be inserted without
changing the sorting order. `upper_bound` returns the last iterator where
that is true.

As long as we don't have multiple equivalent entries in our container,
`lower_bound` and `upper_bound` are the same. To list all words starting with
`main` from our sorted vector `owords`, we can do:

```
string val("main");

auto iter = lower_bound(owords.begin(), owords.end(), val);

for(; iter != owords.end() && !iter->compare(0, val.size(), val); ++iter) 
	cout<<" "<<*iter<<endl;
```

`std::lower_bound` does the heavy lifting here, performing a binary search
over our sorted `std::vector`. The `for` loop bears a bit of explaining. The
first check `iter != owords.end()` will stop us if `lower_bound` did not
find anything. 

The second check using `iter->compare` performs a substring match on at most
the first 4 characters of a candidate word. Once that no longer matches,
we've iterated beyond the words that start with "main". 

Some more containers
====================
In the previous examples we've used the very basic `std::vector`, which is
contiguous in memory and compatible with C, and `std::unordered_map`, which
is a pretty fast key/value store, but has no ordering.

There are several other useful containers:

 * [`std::map`](https://en.cppreference.com/w/cpp/container/map) an ordered
   map, where you can pass a comparison function if you want, for example to
   get case-insensitive ordering.  Many many examples you will see
   needlessly use `std::map`. This is because pre-2011, C++ had no unordered
   containers.  Ordered containers are wonderful when you need ordering, but
   present unnecessary overhead otherwise.
 * [`std::set`](https://en.cppreference.com/w/cpp/container/set). This is
   like a `std::map<string,void>`, in other words, it is a key value store
   without keys. Like `std::map` it is ordered, which you often do not need.
 * [`std::multi_map`](https://en.cppreference.com/w/cpp/container/multi_map)
   and [`std::multi_set`](https://en.cppreference.com/w/cpp/container/multi_set).
   These work just like regular `set` and `map`, but then allowing multiple
   equivalent keys. This means these containers can't be queried with `[]`
   since that only supports a single key.
 * [`std::deque`](https://en.cppreference.com/w/cpp/container/deque). A
   double-ended queue which is a great workhorse for implementing any kind of
   queue. Storage is not consecutive, but popping and pushing elements from
   either end is fast.

A full list of standard containers can be found
[here](https://en.cppreference.com/w/cpp/container)

Boost containers
================

Although this series of posts focuses on 'core' C++, I would be remiss if I
did not point out a few parts of Boost at this point.  Boost is a large
collection of C++ code, some of which is excellent (and tends to make it
into the C++ standard, which is edited by some of the Boost authors), some
of which is good and then there are some unfortunate parts.

But the good news is, most of Boost is pretty modular: it is not a framework
kind of library where if you use one part, you use all parts.  And in fact,
many of the most interesting parts are include-only, with no need to link in
libraries.  Boost is universally available and freely licensed.

First up is the Boost
[Container](https://www.boost.org/doc/libs/1_67_0/doc/html/container/non_standard_containers.html)
library, which is not a library but a set of includes. This offers niche
containers that are mostly completely compatible with standard library
containers, but offer specific advantages if they match your usecase. 

For example, `boost::container::flat_map` (and set) are like `std::map` and
`std::set` except they use contiguous slabs of memory for cache efficiency.
This makes them slow on inserts, but lightning fast on lookups.

As another example, `boost::container::small_vector` is optimized for
storing a small (templatized) number of elements, which can save a lot of
`malloc` traffic.

Lots more Boost containers can be found
[here](https://www.boost.org/doc/libs/?view=category_containers).

#### Boost.MultiIndex
Secondly, in part 1 of this series I promised I'd stay away from exotics and
"template metaprogramming".  But there is one pearl I must share with you,
something I regard as the golden standard by which I measure programming
languages.  Is the language powerful enough to implement
[Boost.MultiIndex](https://www.boost.org/doc/libs/1_67_0/libs/multi_index/doc/index.html)?

In short, we frequently need objects to be findable in several ways.  For
example, if we have a container of open TCP sessions, we may want to find
sessions based on the 'full source IP, source port, destination IP,
destination port' tuple, but also on only source IP or destination IP.  We
might also like to have time ordering to harvest/close old sessions.

The 'manual' way of doing this is to have several containers in which
objects live, and use all these to find objects using various keys:

```
map<pair<IPEndpoint,IPEndpoint>, TCPSession*> d_sessions;
map<IPEndpoint, TCPSession*> d_sessionsSourceIP;
map<IPEndpoint, TCPSession*> d_sessionsDestinationIP;
multimap<time_t, TCPSession*> d_timeIP;

auto tcps = new TCPSession;
d_sessions[{srcEndpoint, dstEndpoint}] = tcps;
d_sessionsSourceIP[srcEndpoint] = tcps;
d_sessionsDestinationIP[dstEndpoint] = tcps;
...
```

While this works, we suddenly have to do a lot of housekeeping. If we want
to remove a `TCPSession`, we have to remember to erase it from all
containers, for example, and then free the pointer.

[Boost.MultiIndex](https://www.boost.org/doc/libs/1_67_0/libs/multi_index/doc/index.html)
is a work of art that not only offers containers that can be searched in
several ways at the same time, it also offers (un)ordered, unique and
non-unique indexes, plus the use of partial keys for lookups, as well as
'alternate keys', which enable you to find a `std::string` key using a `char
*` (which saves the creation of temporaries).

Here's how we'd lookup TCP sessions.  First let's start with some
groundwork ([full
code](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc)):

```
struct AddressTupleTag{};
struct DestTag{};
struct TimeTag{};

struct Entry
{
  IPAddress srcIP;
  uint16_t srcPort;
  IPAddress dstIP;
  uint16_t dstPort;
  double time;
};
```

The three `Tag`s provide types that identify three different indexes we will
be defining on our container. We then define a struct that our
Boost.MultiIndex container will contain. Note that the keys we want to
search on are in the actual container itself - there is no separation here
between key and value.

Next up the admittedly tricky definition of the container. You might
literally spend an hour getting this right, but once it is right everything
is easy:

```
typedef multi_index_container<
  Entry,
  indexed_by<
    ordered_unique<
      tag<AddressTupleTag>,
      composite_key<Entry,
                    member<Entry, IPAddress, &Entry::srcIP>,
                    member<Entry, uint16_t, &Entry::srcPort>,
                    member<Entry, IPAddress, &Entry::dstIP>,
                    member<Entry, uint16_t, &Entry::dstPort> >
      >,
    ordered_non_unique<
      tag<DestTag>,
      composite_key<Entry,
                    member<Entry, IPAddress, &Entry::dstIP>,
                    member<Entry, uint16_t, &Entry::dstPort> >
      >,
    ordered_non_unique<
      tag<TimeTag>,
      member<Entry, double, &Entry::time>
      >
    >
  > tcpsessions_t;
```

This defines three indexes, one ordered & unique, and two ordered and
non-unique. The first index is on the '4-tuple' of the TCP session. The
second one only on the destination of the session. The last one on the
timestamp.

It is important to note that this template definition creates the entire
container code at compile time. All of this leads to code that, as usual for
templated containers, is as efficient as if you had written it yourself.
Most of the time in fact, a Boost.MultiIndex container will be faster than a
`std::map`.

Let's fill it with some data:

```
  tcpsessions_t sessions;
  double now = time(0);

  Entry e{"1.2.3.4"_ipv4, 80, "4.3.2.1"_ipv4, 123, now};

  sessions.insert(e);
  sessions.insert({"1.2.3.4"_ipv4, 81, "4.3.2.5"_ipv4, 1323, now+1.0});
  sessions.insert({"1.2.3.5"_ipv4, 80, "4.3.2.2"_ipv4, 4215, now+2.0});
```

The first line uses our `typedef` to make an actual instance of our
container, the second line takes the current time and puts it in a double.

Then some magic happens called a [user-defined
literal](https://en.cppreference.com/w/cpp/language/user_literal) which
means that `"1.2.3.4"_ipv4` gets converted to 0x01020304 - **at compile
time**. To observe how this works, head over to
[multi.cc](https://github.com/ahupowerdns/hello-cpp/blob/master/multi.cc) on
GitHub. These party tricks are optional when using C++, but `constexpr`
compile time code execution is pretty neat.

After this has run, our `sessions` container has 3 entries. Let's list them
all in time order:

```
  auto& idx = sessions.get<TimeTag>();
  for(auto iter = idx.begin(); iter != idx.end(); ++iter)
    cout << iter->srcIP << ":" << iter->srcPort<< " -> "<< iter->dstIP <<":"<<iter->dstPort<<endl;
```

This prints:

```
1.2.3.4:80 -> 4.3.2.1:123
1.2.3.4:81 -> 4.3.2.5:1323
1.2.3.5:80 -> 4.3.2.2:4215
```

In the first line, we request a reference to the `TimeTag` index, which we
iterate over as usual in the second line.

Let's do a partial lookup in the 'main' index, which is on the full 4-tuple:

```
  cout<<"Search for source 1.2.3.4, every port"<<endl;
  auto range = sessions.equal_range(std::make_tuple("1.2.3.4"_ipv4));

  for(auto iter = range.first; iter != range.second ; ++iter)
    // print
```

By creating a tuple with only one member, we indicate we want to do our
lookup on only the first part of the 4-tuple. If we had added '`, 80`' to
`std::make_tuple`, we would have found only one matching TCP session instead
of two. Note that this lookup used `equal_range` as described earlier in
this page.

Finally to search on a TCP session's destination:
```
  cout<<"Destination search for 4.3.2.1 port 123: "<<endl;
  auto range2 = sessions.get<DestTag>().
	equal_range(std::make_tuple("4.3.2.1"_ipv4, 123));

  for(auto iter = range2.first; iter != range2.second ; ++iter)
    // print
```

This requests the `DestTag` index, and then uses it to find sessions to
`4.3.2.1:123`.

I hope you will forgive me this excursion outside of standard C++, but as
Boost.MultiIndex is part of almost all code I write, I felt the need to
share it.

Summary
=======
In this long part 4, we have delved into some of the nitty-gritty of
lambdas, and how they can be used for custom sorting, how they can be stored
and when they are a good idea. 

Secondly we explored the interaction between algorithms and containers by
enhancing our code indexer with the ability to look up partial words, by
sorting our unordered container of words into a flat vector. We also looked
how some 'C' like tricks could be used to make this process both use less
memory and be more dangerous. 

We also looked into the rich array of algorithms provided by C++, enabled by
the separation of code between [containers](https://en.cppreference.com/w/cpp/container) and
[algorithms](https://en.cppreference.com/w/cpp/algorithm). Before doing any
data manipulation, check the existing algorithms if there is something that
does what you need already.

Finally we covered further containers found in Boost, including the most
magical and powerful
[Boost.MultiIndex](https://www.boost.org/doc/libs/1_67_0/libs/multi_index/doc/index.html).

In part 5 we will round off this series with a discussion of the ultimate
smart pointer, `std::unique_ptr` and the associated concept of `std::move`.
If you have
any other favorite things you'd like to see discussed or questions, please do hit
me up on [@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert) or
bert.hubert@powerdns.com.

**NOTE**: If you like this stuff, come work with me over at PowerDNS -
[aspiring C++ programmers
welcome](https://www.powerdns.com/careers.html#securityDeveloper)!


<script>
/* dear reader - this is not to track you, but I need some metrics to know
 * how many people read my posts. What you are seeing is a small check to
 * determine if you are an actual visitor or a crawler. 
*/
function realityCheck()
{
	var num = 1*2*3*4*5*6*7*8*9*10;
	num = Math.sqrt(num);
	fetch('forreal.json?'+num+'&'+Math.random());
}
setTimeout(realityCheck, 1000 * 10);
</script>



 


