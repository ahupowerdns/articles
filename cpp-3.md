---
title: "Modern C++ for C Programmers: Part 3"
date: 2018-07-02T21:30:31+02:00
draft: false
---
**NOTE**: If you like this stuff, come work with me over at PowerDNS -
[aspiring C++ programmers
welcome](https://www.powerdns.com/careers.html#securityDeveloper)!

Welcome back!  In [part 2](../cpp-2/) I discussed basic classes, threading,
atomic operations, smart pointers, resource acquisition and (very briefly)
namespaces.

In this part we continue with further C++ features that you can use to spice
up your code 'line by line', without immediately having to use all 1400
pages of 'The C++ Programming Language'.

Various code samples discussed here can be found on
[GitHub](https://github.com/ahuPowerDNS/hello-cpp).

If you have any favorite things you'd like to see discussed or
questions, please hit me up on
[@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert) or
bert.hubert@powerdns.com

Inheritance & polymorphism
==========================
These are features that are sometimes useful, but by no means the first
things to reach for.  It is entirely possible to write C++ that never uses
any kind of inheritance.  In other words, there is no pressure or need to be
"all object oriented".

But, it does have its uses. For example, an event-processing library may
have to deal with arbitrary kinds of events, all of which have to pass
through a single API. It works like this:

```
class Event
{
public:
	std::string getType();
	struct timeval getTime();
	virtual std::string getDescription()=0;

private:
	std::string m_type;
	struct timeval m_time;
};
```

This is the base class called `Event`. It has three methods, two of which
are normal member functions: `getType()` and `getTime()`. These return data
from the two member variables that every `Event` shares.

The third one `getDescription()` is `virtual` which means it is different in
derived classes.  In addition, we have set the function to zero which means
that every derived class will have to define this function. In other words,
you can't do this:

```
Event e; // will error on 'pure virtual getDescription'
```

To make an actual Event that works, we do:
```

class PortScanEvent : public Event
{
public:
	virtual std::string getDescription() override
	{
		return "Port scan from "+m_fromIP;
	}	
private:
	std::string m_fromIP;
};
```

This defines a derived class that inherits from `Event`, which is its 'base
class'.  Note how we define `getDescription` here, and that it is flagged
with `override` which means the compiler will error out unless we are
actually overriding a base class method.

Let's assume we've also created an `ICMPEvent`, we could now write:

```
  PortScanEvent pse;
  cout << pse.getType() << endl; // "Portscan"

  ICMPEvent ice;
  cout << ice.getType() << endl; // "ICMP"
```

This is all entirely conventional, and does not do any magic, except that we
are using an object that is partially defined in its base class.

However, we can also do:

```
  Event* e = &ice;
  cout << e->getDescription() << endl; // "ICMP of type 7"

  e = &pse;
  cout << e->getDescription() << endl; // "Portscan from 1.2.3.4"
```

This defines a pointer to an `Event`, in which we first store a pointer to
an `ICMPEvent`. And lo, it continues to function as an `ICMPEvent`, even
when stored in an `Event` pointer. The next two lines demonstrate how this
also works for a `PortScanEvent`.

`Event` contains metadata which lets us know at runtime what type it
`really` holds, and this metadata is consulted before doing any call that
needs to know the actual class in there. This represents overhead, but is is
the same kind of overhead generated if classes are simulated, as many C
projects end up doing.

Interestingly enough, in the case above, compilers are sometimes able to
'[devirtualise](http://hubicka.blogspot.com/2014/01/devirtualization-in-c-part-1.html)'
calls since from the control flow, they know at compile time what the actual
type of `Event` will be - an optimization not typically implemented in
simulated classes in C.

Finally, if you ever need to know, you can find out what the real type of an
`Event` is like this:

```
  auto ptr = dynamic_cast<PortScanEvent*>(e);
  if(ptr) {
    cout << "This is a PortScanEvent" << endl;
  }
```

If you got it wrong, `ptr` will be 0 (or `nullptr` in modern C++ lingo).

I personally rarely use `runtime polymorphism` in a project, and then almost
exclusively for APIs that need to receive/respond with records or events of
different types. It is of great use whenever you have a collection of
different 'things' that need to be stored in a single data structure. 

Of special note, the moment you find yourself writing functions that start
with a `switch` statement depending on the type of your struct, you are
likely better off using actual C++ inheritance.

A brief note on references
==========================

```
void f(int& x) 
{
	x=0;
}

...

int i = 0;
int& j = i; 
j = 2;
cout << i << endl; // prints 2

f(i); // both i and j are now 0
```

Up to now, I have neglected to describe references, which made it to two
examples in parts 1 and 2 of this series.  Technically, a reference is
nothing other than a pointer.  There is no overhead.  They are so much the
same one may wonder why C++ bothered to provide this alternate syntax. 
Pointers already provided for pass by reference semantics.

There are some finer points to be made, but references do save some typing.
Functions can now return things 'by reference' and not force you to add `*`
or `->` to every use for example. 

This makes it possible for containers to implement: `v[12]=4` for example,
which underneath is `value_type& operator[](size_t offset)`. If this
returned a `value_type*` we'd have to type `*(v[12])=12` everywhere.

Some discussion on pointers versus references can be found
[here](https://stackoverflow.com/questions/7058339/when-to-use-references-vs-pointers)
and
[here](https://stackoverflow.com/questions/8007832/could-operator-overloading-have-worked-without-references)
on Stackoverflow.

Templates
=========
As noted in [part 1](../c++-1), C++ was designed with the "Zero Overhead"
principle in mind, which in its second part states "[no] feature [should] be
added for which the compiler would generate code that is not as good as a
programmer would create without using the feature". This is a bold
statement.

Most programming languages called "Object Oriented" have made all objects
descend (or inherit) from a magic Object base class.  This means that to
write a container in such a language means writing a data structure that
hosts `Object` instances.  If we store an `ICMPEvent` in there, we do so as
an `Object`.

A problem with this technique is that for C++, it violates the "Zero
Overhead" principle. Storing a billion 32 bit numbers in C uses 4GB of
memory. Storing a billion `Object` instances will use no less than 16GB -
and likely more.

To adhere to its "Zero Overhead" promise, C++ implemented 'templates'.
Templates are like macros on steroids and come with tremendous power and, it
has to be said, complication. They are worth it however, as they enable the
library to provide generic containers that are as good and likely better
than what you could write - and leave open the possibility of creating
better versions too.

As a brief example:

```
template<typename T>
struct Vector
{
  void push_back(const T& t)
  {
    if(size + 1 > capacity) {
      if(capacity == 0) 
        capacity = 1;
      auto newcontents = new T[capacity *= 2];
      for(size_t a = 0 ; a < size ; ++a)
        newcontents[a]=contents[a];
      delete[] contents;
      contents=newcontents;
    }
    contents[size++] = t;
  }
  T& operator[](size_t pos)
  {
    return contents[pos];
  }       

  ~Vector()
  {
    delete[] contents;
  }

  size_t size{0}, capacity{0};
  T* contents{nullptr};
};

```

This implements a simplistic auto-growing vector of arbitrary type. It is
used like this:

```
  Vector<uint32_t> v;
  for(unsigned int n = 0 ; n < 1000000000; ++n) {
    v.push_back(n);
  }
```

When the compiler encounters the first line, it triggers the instantiation
of the `Vector` with `T` replaced by `uint32_t`. This delivers the exact
same code as if you had written it by hand. There is no overhead.

Similarly, functions can be templatized, for example:

```
// within Vector
  template<typename C>
  bool isSorted(C pred)
  {
    if(!size)
      return true;
    for(size_t n=0; n < size - 1; ++n)
      if(!pred(contents[n], contents[n+1]))
        return false;
    return true;
  }
}
```

And a sample use:

```
struct User
{
	std::string name;
	int uid;
};

Vector<User> users;
// fill users

if(users.isSorted([](const auto& a, const auto& b) {
	return a.uid < b.uid;}) 
{
	// do things
}
```

This code compiles down to be as efficient as if you had written `a.uid <
b.uid` in there yourself.  Incidentally, because templates are incredibly
generic, you could also pass function pointers or whole object to
`isSorted`, as long as it is something the compiler can call.  Incidentally,
as shown in [part 1](../c++-1/), passing a "predicate" like this is also
what makes `std::sort` faster than C `qsort`.

Based on these templates, C++ offers an [array of powerful
containers](https://en.cppreference.com/w/cpp/container) to store data in. 
Each of these containers comes with an API but also with a performance
(scaling) guarantee.  This in turn makes sure that implementors have to use
state of the art algorithms - and they do.

**Note**: This also means none of the sample code above should ever see
production -
[`std::vector`](https://en.cppreference.com/w/cpp/container/vector) and the associated
[`std::is_sorted`](https://en.cppreference.com/w/cpp/algorithm/is_sorted)
are already there and do a far better job.

You may find yourself *using* a lot of these templated containers and
associated algorithms, but very rarely writing any templated functions
yourself.  And this is pretty good news in two ways - first, writing
templated code is harder than you'd think, and it has some surprising
syntactical inconveniences.  But secondly, almost everything you'd want to
write a template for has been written already, so there rarely is a need.

A worked example
================
Our hallowed "The C Programming Language" contains example code to count the
use of C keywords in C programs. The good book rightfully spends a lot of
time creating the relevant data structures from scratch.

Within our C++ environment however, we know our journey starts with very
capable data structures already, so let's not only count words in code,
let's also index everything. What follows is a 100 line example that indexes
the entire Linux kernel source code (692MB) in 10 seconds, and performs
instant lookups.

First, let's read which files to index:
```
int main(int argc, char** argv)
{
	vector<string> filenames;

	ifstream ifs(argv[1]);
	std::string line;

	while(getline(ifs, line))
		filenames.push_back(line);

	cout<<"Have "<<filenames.size()<<" files"<<endl;
```

We read a file with filenames to index into a `std::vector`.  Earlier I
noted the C++ `iostreams` are in no way mandatory and that you can continue
to use C `stdio`, but in this case iostreams are a convenient way to read
lines of text of arbitrary length.

Next up, we read and index each file in turn:

```
	unsigned int fileno=0;
	std::string word;

	for(const auto& fname : filenames) {   // "range-based for"
		size_t offset=0;
    		SmartFP sfp(fname.c_str(), "r");
		while(getWord(sfp.d_fp, word, offset)) {
			allWords[word].push_back({fileno, offset});
    		}
		++fileno;
  	}
```

This special form of `for` loop iterates over the contents of the
`std::vector` `filenames`.  This syntax works on all standard C++
containers, and will work on yours too if you [adhere to some simple
rules](https://www.cprogramming.com/c++11/c++11-ranged-for-loop.html). 

In the next line we use `SmartFP` as defined in [part 2](../cpp-2/) of this
series.  `SmartFP` internally carries a C `FILE*`, which means we get the
raw speed of C `stdio`.

`getWord` can be found in our sample code
[GitHub](https://github.com/ahupowerdns/hello-cpp/blob/master/windex.cc),
the only special thing to note there is that `std::string word` gets passed
as a reference. This means we can keep reusing the same string instance,
which saves a ton of malloc traffic.  It also passes `offset` by reference,
and this gets updated to the offset where this word was found in the file.

The `allWords` line is where the action happens. allWords is defined like
this:

```
struct Location
{
	unsigned int fileno;
	size_t offset;
};

std::unordered_map<string, vector<Location>> allWords;
```

What this does is create an unordered associative container, keyed on a
`string`. Per entry it contains a vector of Location objects, each of which
represents a place where this word was found.

Internally, the `unordered` containers are based on the hash of the key and
by default C++ knows how to hash most primitive data types already. Compared
to an `std::map`, which is ordered, the unordered variant is twice as fast
for this usecase. 

To actually look something up, we do:

```
while(getline(cin, line)) {
	auto iter = allWords.find(line);

	if(iter == allWords.end()) {
		cout<<"Word '"<<line<<"' was not found"<<endl;
		continue;
	}

	cout<<"Word '"<<line<<"' occurred "<<iter->second.size()<<" times: "<<endl;

	for(const auto& l : iter->second) {
		cout<<"\tFile "<<filenames[l.fileno]<<", offset "<<l.offset<<endl;
	}
}
```

This introduces the concept of an iterator. Sometimes an iterator is nothing
more than a pointer, sometimes it is more complex, but it always denotes a
'place' within a container. There are two magic places, one `begin()`, one
`end()`. To denote that nothing was found, the `end()` iterator is returned,
and this is what we check against in the next line.

If we have found something, we use iterator syntax to print how many hits we
found: `iter->second.size()`.  For an associative container, an iterator has
two parts, conveniently called `first` and `second`.  The first one has the
key of the item, the other one provides access to the value found there (or,
the actual item).

In the final three lines, we again use range-based for syntax to loop over
all the `Location`s for the word we searched for, and print the details.

Now, to calibrate, we spent less than 50 unique lines on this using only
default functionality, is this any good?

```
$ find ~/linux-4.13 -name "*.c" -o -name "*.h" > linux
$ /usr/bin/time ./windex linux < /dev/null
Have 45000 files
...
45000/45000 /home/ahu/linux-4.13/tools/usb/ffs-test.c, 103302249 words, 607209 different
Read 692542148 bytes
Done indexing
9.62user 1.09system 0:10.73elapsed 99%CPU (0avgtext+0avgdata 2047712maxresident)k
```

This represents an indexing speed of around 65MB/s, covering 692MB of data. 
Lookups are instantaneous.  Memory usage, at 2GB is around 20 bytes per word
which is not too shabby, since this includes storage of the word itself.  If
we add `__attribute__((packed))` or the equivalent to our `struct Location`,
storage requirements decrease to 1.4GB or 15 bytes per word - close to the
theoretical minimum of 12. 

On [GitHub](https://github.com/ahupowerdns/hello-cpp/blob/master/windex.cc)
you can find the source code of the indexer - with the additional
functionality that it can do prefix searches as well.

Summarising
===========
C++ offers inheritance and polymorphic classes, and these are sometimes
useful, definitely better than the hand-coded equivalent, but absolutely not
mandatory to use. As an alternative to deriving everything from
`Object`, as some languages do, C++ offers templates, which provide generic
code that works on arbitrary types. This is what enables `std::sort` to be
faster than `qsort` in C.

Templates are very powerful, and underlie the useful array of C++ standard
containers like `std::map`, `std::unordered_map`, `std::vector`. Using
range-based for loops and iterators, containers can be interrogated and
modified.

In part 4 we will continue to explore C++ and containers, and if you have
any favorite things you'd like to see discussed or questions, please do hit
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



 