---
title: "Modern C++ for C Programmers: Part 5"
date: 2018-07-22T21:26:51+02:00
draft: false
---

**NOTE**: If you like this stuff, come work with me over at PowerDNS -
[aspiring C++ programmers
welcome](https://www.powerdns.com/careers.html#securityDeveloper)!

Welcome back!  In [part 4](../cpp-4/) we went over the nitty-gritty of
lambdas and how to store them, we explored the relation between the various
C++ algorithms and containers, plus we took a stroll through some
non-standard containers with exceptional capabilities. 

In this probably final part 5, we'll be going over some of the most powerful
stuff in modern C++: "perfect" reference counting and the concept of
`std::move`. Note that this installment introduces some pretty unfamiliar
concepts, so it may be heavier reading than earlier parts.

Memory management
=================
Memory is frequently the most important factor determining a program's speed
and reliability.  CPUs these days tend to be tremendously faster than their
attached RAM, so preventing needless copies and fragmented memory access may
deliver whole orders of magnitude improvements in speed.

The authors of C++ were well aware of this and have delivered functionality
that makes it possible to pass around or construct objects "in place", thus
saving a lot of memory bandwidth.

Most of these techniques require some thinking, but we're going to start
with one that is nearly invisible, and explains why some C code becomes
faster simply by being recompiled as C++.

Copy elision or Return Value Optimization
=========================================
Ponder the following code:

```
struct Object
{
	// many fields
};

Object getObject(size_t i)
{
	Object obj;
	// retrieve object #i
	return obj;
}

int main()
{
	Object o = getObject(271828);
}
```

Seasoned C programmers will have been exhorted to never do this, starting
from gloried pages of K&R. "Passing structs over the stack leads to needless
copying". Instead, we'd be passing a pointer to an `Object`, meticulously
reset it to its default state and then fill it out.

In C, the compiler is not allowed to optimize the code as written above, and
the `struct Object` gets copied on the return from `getObject`.  In C++
however, not only is the compiler allowed to optimize, all of them actually do
it. In effect, `Object o` gets constructed on the caller's stack and filled
out there, with no copying going on. 

This enables code like this to be efficient:

```
Vector<Object> getAll(); // "Vector" from part 3
..
auto all = getAll(); // returns millions of Objects
```

Later we will see how C++ offers explicit opportunities to transfer
ownership without copying.

Smart Pointers
==============
In [part 2](../cpp-2) we touched on smart pointers.  We also noted that
memory leaks are the bane of every project, and probably the most vexing
thing about writing in pure C.  Every C (and C++) programmer I know has lost
at least one solid week on chasing an obscure memory leak.

These problems are so large that most modern programming languages decided
to incur a huge amount of overhead to implement garbage collection (GC). GC
is amazing when it works, and especially lately, the overhead is now at
least manageable. But as of 2018, all environments still struggle with
hick-ups caused by GC runs, which always tend to happen exactly when you
don't want them to. And to be fair, this is a very difficult problem to
solve, especially when many threads are involved.

C++ has therefore not implemented garbage collection. Instead, there is a
judicious selection of smart pointers that perform their own cleanup. In
[part 2](../cpp-2) we described `std::shared_ptr` as "the most do what I
mean" smart pointer available, and this is true.

Such magic however does not come for free however.  If we look 'inside' a
`std::shared_ptr`, it turns out it carries a lot of administration.  First
there is of course the actual pointer to the object contained.  Then there
is the reference count, which needs to be updated and checked **atomically**
at all times.  Next up, there may also be a custom deleter.  For good
reasons, this metadata is itself allocated dynamically (on the heap).  So
while the `sizeof` of a `std::shared_ptr` may only be 16 bytes (on a 64 bit
system), effectively it uses much more memory.  In [one specific
test](https://github.com/ahupowerdns/hello-cpp/blob/master/move.cc), a
`std::shared_ptr<uint32_t>` ended up using 47 bytes of memory on average.

The question of course is: can we do better?

Introducing: std::unique_ptr
============================
The overhead of generic reference counted pointer was well known when C++
took its initial standardized form.  Back then, a quirky smart pointer
called [`std::auto_ptr`](https://en.cppreference.com/w/cpp/memory/auto_ptr)
was defined, but it turned out that within the C++ of 1998 it was not
possible to create something useful.  Making "the perfect smart pointer"
required features that only became available in C++ 2011.

First, let us try some simple things ([source on
GitHub](https://github.com/ahupowerdns/hello-cpp/blob/master/move.cc)):

```
  std::unique_ptr<uint32_t> testUnique;
  uint32_t* testRaw;
  std::shared_ptr<uint32_t> testShared;

  cout << "sizeof(testUnique):\t" << sizeof(testUnique) << endl;
  cout << "sizeof(testRaw):\t" << sizeof(testRaw) << endl;
  cout << "sizeof(testShared):\t" << sizeof(testShared) << endl;
```

Rather amazingly this outputs:

```
sizeof(testUnique): 8
sizeof(testRaw):    8
sizeof(testShared): 16
```

You saw that right. A `std::unique_ptr` has no overhead over a 'raw'
pointer. And in fact, with some judicious casting, you can find out it in
contains nothing other than the pointer you put in there. That's zero
overhead.

Here's how to use it:

```
void function()
{
  auto uptr = std::make_unique<uint32_t>(42);

  cout << *uptr << endl;
} // uptr contents get freed here
```

The first line is a shorter (and better) way to achieve:

```
std::unique_ptr<uint32_t> uptr = std::unique_ptr<uint32_t>(new uint32_t(42));
```

> In general, always prefer the `std::make_` form for smart pointers.
> For `std::shared_ptr` it turns two allocations into one, which is a huge
> win both in CPU cycles and memory consumed.

It should be noted that `std::unique_ptr` may be a smart pointer,
but it is not a generic reference counted pointer.  Or, to put it more
precisely, there is always exactly one place that owns a `std::unique_ptr`. 
This is the magic of why there is no overhead: there is no reference count
to store, it is always '1'.

`std::unique_ptr` cleans up only when it goes out of scope, or when it is
reset or replaced.

To access the contents of a smart pointer, either `deference` it (with `*`
or `->`), or use the `get()` method if you need the actual pointer inside. 
Smart pointers can also be unset, and in that case evaluate as 'false':

```
  std::unique_ptr<int> iptr;

  auto p = [](const auto& a) {
    cout << "pointer is " << (a ? "" : "not ") << "set\n";
  };

  p(iptr);
  cout << (void*) iptr.get() << endl;  

  iptr = std::make_unique<int>(12);
  p(iptr);

  iptr.reset();
  p(iptr);
```

This prints:
```
pointer is not set
0
pointer is set
pointer is not set
```

If we attempt to copy a `std::unique_ptr`, the compiler stops us. It does
allow us however to 'move' it:

```
  std::unique_ptr<uint32_t> uptr2;

  uptr2 = uptr; // error about 'deleted constructor'

  uptr2 = std::move(uptr); // works
```

The reason a simple copy is not allowed is that this would lead to a
non-unique pointer: both uptr and uptr would refer to the same `uint32_t`
instance. 

So what is this `std::move` thing, and why does that work?

std::move
=========
In [part 2](../cpp-2) we defined a `SmartFP` class as an example of Resource
Acquisition Is Initialization (RAII). It's intended to be used like this:

```
int main()
try
{
	string line;
	SmartFP sfp("/etc/passwd", r");
	stringfgets(line, sfp.d_fp); // simple wrapper
	// do stuff
}
catch(std::exception& e) 
{
	cerr << "Fatal error: " << e.what() << endl;
}
```

`SmartFP` underneath is nothing but a wrapper for `fopen` and `fclose`.  It
also turns `fopen` errors into an explanatory exception.  The nice thing
about RAII that it guarantees the filedescriptor won't ever leak, even in
the face of error conditions.

In [part 2](../cpp-2) we also noted that as defined, `SmartFP` had a
problem.  It performs an `fclose` when it goes out of scope, but what if
someone copied our `SmartFP` instance?  We would then close the same `FILE`
pointer twice, which is very bad.  Enter the **move constructor**:

```
struct SmartFP
{
  SmartFP(const char* fname, const char* mode)
  {
    d_fp = fopen(fname, mode);
    if(!d_fp)
      throw std::runtime_error("Can't open file: " + stringerror());
  }

  SmartFP(SmartFP&& src) // move constructor. Note "&&"
  {
    d_fp = src.d_fp;
    src.d_fp = 0;
  }

  ~SmartFP()
  {
    if(d_fp)
      fclose(d_fp);
  }
  FILE* d_fp{0};
};
```

The `move constructor` is the important bit. Its presence tells C++ that
this class can not be copied, only moved. The semantics of a `move` are a true
transfer of ownership. 

The following may help:

```
SmartFP sfp("/etc/passwd", "ro");
cout << (void*) sfp.d_fp << endl;  // prints a pointer

SmartFP sfp2 = sfp;                // error
SmartFP sfp2 = std::move(sfp);     // transfer!

cout << (void*) sfp.d_fp << endl;  // prints 0
cout << (void*) sfp2.d_fp << endl; // prints same pointer
```

When this code runs, the `FILE` pointer we created on the first line gets
`fclose`d exactly once. This is because during the move, the 'donor' `FILE*`
was set to zero, and in the destructor we make sure not to `fclose` a 0.

This `move` is performed automatically on return:
```
SmartFP getTmpFP()
{
	// get tmp name
	return SmartFP(tmp, "w");
}

...

SmartFP fp = getTmpFP();
```

Additionally, the C++ standard containers are all `move` aware, with a
special syntax to construct elements 'in place':

```
  vector<SmartFP> vec;
  vec.emplace_back("move.cc", "r");
```

All the parameters to `emplace_back` get forwarded to the `SmartFP`
constructor, which constructs the instance straight into the `std::vector` -
all without a single copy. When filling large containers, this can make a
huge difference.

Note that if we want to, a class can have both move constructors and regular
constructors. A good example of this are all the C++ standard containers,
including `std::string`. This gives you a choice between making a real copy
or transferring ownership.

Smart pointers and polymorphism
===============================
A main reason we store things as pointers is to benefit from polymorphism.
The downside of pointers is of course memory management, so it would be
great if smart pointers were to interoperate with base and derived classes.
The wonderful news is that they do.

Based on our `Event` class from [part 3](../cpp-3):

```
  std::deque<std::unique_ptr<Event>> eventQueue;

  eventQueue.push_back(std::make_unique<PortScanEvent>("1.2.3.4"));
  eventQueue.push_back(std::make_unique<ICMPEvent>());

  for(const auto& e : eventQueue) {
    cout << e->getDescription() << endl;
  }
```

This all works as expected, and the contents of `eventQueue` get cleaned up
when the container goes out of scope. 

> When using polymorphic classes, make sure there either is no ~destructor,
> or that it is declared as virtual. Otherwise `std::unique_ptr` will call
> the base class destructor. See [part 3](../cpp-3) for more details, plus
> this [stackoverflow
> post](https://stackoverflow.com/questions/461203/when-to-use-virtual-destructors)

Placement new
=============
As noted earlier, C makes it possible to 'live on the edge', or as some of
the Node.JS people said, to ['be close to the
metal'](https://twitter.com/shit_hn_says/status/234856345579446272). The
good thing is that C++ offers you that same ability, should you need it, and
more.

When we do the following:

```
auto ptr = new SmartFP("/etc/passwd", "ro");
```

This does two things:

 1. Allocate memory to store a `SmartFP` instance
 2. Call the `SmartFP` constructor using that memory

Generally this is what we need.  However, sometimes our memory arrives from
elsewhere but we'd still like to construct objects on there.  Enter
`placement new`.

Here is an actual usecase from the PowerDNS `dumresp` utility:

```
  std::atomic<uint64_t>* g_counter;

  auto ptr = mmap(NULL, sizeof(std::atomic<uint64_t>), PROT_READ | PROT_WRITE,
		  MAP_SHARED | MAP_ANONYMOUS, -1, 0);

  g_counter = new(ptr) std::atomic<uint64_t>();
  
  for(int i = 1; i < atoi(argv[3]); ++i) {
    if(!fork())
      break;
  }
```

This uses `mmap` to allocate memory that will be shared with any child
processes, and then uses fancy `placement new` syntax to construct a
`std::atomic<uint64_t>` instance in that shared memory.

The code then forks the number of processed described in `argv[3]`. Within
all these processes, a simple `++(*g_counter)` works, and all update the
same counter.

Based on techniques like these, it is possible to create highly efficient and
easy to use interprocess communications libraries, like for example [Boost
Interprocess](https://www.boost.org/doc/libs/1_67_0/doc/html/interprocess/quick_guide.html).

Some general advice
===================
Many modern C++ projects will only have a handful of explicit calls to `new`
or `delete` (or `malloc/free`). It is easy to audit those few calls.
Restrict manual memory allocation to the cases where you really have to.

For the rest, use `std::unique_ptr` if you can get away with it, and
`std::shared_ptr` when you can't.  Note that you can convert a
`std::unique_ptr` into a `std::shared_ptr` efficiently, so you can change
your mind:

```
auto unique = std::make_unique<std::string>("test");
std::shared_ptr<std::string> shared = std::move(unique);
```

In addition, a `std::unique_ptr` can also `release()` the pointer it owns,
which means it will not get `delete`d automatically. 

The easy ability to cheaply convert a `std::unique_ptr` into a
`std::shared_ptr` or a raw pointer means that functions can return a
`std::unique_ptr` and keep everyone happy.

On the `move constructor`, it pays to understand this somewhat unfamiliar
construct.  Classes that represent resources (like sockets, file
descriptors, database connections) are naturals for having a `move
constructor`, since this makes their semantics closely match how these
resources work: should be opened and closed exactly once, and exactly when
we want them to.


Summarising
===========
Memory allocation is hard and various smart pointers provided by C++ make
it easier. `std::shared_ptr` is luxurious but comes with baggage,
`std::unique_ptr` is frequently good enough and carries no overhead at all. 

C++ tries hard to prevent needless copying of objects and adding a `move
constructor` makes this explicit. By using `std::move` it is possible to
store `std::unique_ptr` instances in containers, which is both safe and
fast.

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



 


