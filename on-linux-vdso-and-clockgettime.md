---
title: "On Linux Vdso and Clockgettime"
date: 2017-03-24T20:22:18+02:00
draft: false
---
### On Linux vDSO and clock_gettime sometimes being slow

Like the previous post on this somewhat dormant blog, I want to share an oddity I discovered that no search engine could really find for me - even though once I found what the problem was, it turns out I was by no means the first person to discover this.  

Some system calls that are used extremely frequently in Linux can be speeded up by a mechanism called vDSO: a virtual dynamically linked shared object. In this way, the kernel can publish selected functions that can run straight in userspace. This means a regular program dynamically links in bits of kernel supplied code, which in turn means that there is no overhead to "jump into the kernel" to execute code. All good.  

One way you notice your system call has received the vDSO treatment is that "strace" and friends no longer see it, since there actually is no system call anymore.  

Of specific interest are time related calls, like gettimeofday and clock_gettime. Many programs make a ton of these calls, and little can be done to prevent it. You might want to cache the current time perhaps, but to do so, you'd need to know the time. So quite some code relies on time related system calls being really really fast.  

This explains why the recent discovery that [the AWS platform does not vDSO gettimeofday](https://blog.packagecloud.io/eng/2017/03/08/system-calls-are-much-slower-on-ec2/) was such a big deal.  

Within PowerDNS software (dnsdist), we use clock_gettime() in hopes of getting the kind of timer we want, and also one that is fast and cheap for the kernel to provide. While doing "million QPS" scale benchmarking of dnsdist today, we did a strace to find out what [dnsdist](http://dnsdist.org/) was doing, and lo, within there we found millions and millions of system calls to clock_gettime(). Help!  

My first thought was that the platform we were on might perhaps not actually support clock_gettime as vDSO. To figure out what is actually in the kernel supplied vDSO, I used a program called dump-vdso.c that can be found [strewn across the web](https://kernel.googlesource.com/pub/scm/linux/kernel/git/luto/misc-tests/+/5655bd41ffedc002af69e3a8d1b0a168c22f2549/dump-vdso.c). This emits the library on stdout, and we can then run the regular objdump tool on it to get:  

```
$ ./dump-vdso > vdso.so  
$ objdump -T vdso.so   

vdso.so:     file format elf64-x86-64  

DYNAMIC SYMBOL TABLE:  
0000000000000418 l    d  .rodata 0000000000000000              .rodata  
0000000000000a20  w   DF .text 0000000000000305  LINUX_2.6   clock_gettime  
0000000000000000 g    DO *ABS* 0000000000000000  LINUX_2.6   LINUX_2.6  
0000000000000d30 g    DF .text 00000000000001b1  LINUX_2.6   __vdso_gettimeofday  
0000000000000f10 g    DF .text 0000000000000029  LINUX_2.6   __vdso_getcpu  
0000000000000d30  w   DF .text 00000000000001b1  LINUX_2.6   gettimeofday  
0000000000000ef0  w   DF .text 0000000000000015  LINUX_2.6   time  
0000000000000f10  w   DF .text 0000000000000029  LINUX_2.6   getcpu  
0000000000000a20 g    DF .text 0000000000000305  LINUX_2.6   __vdso_clock_gettime  
0000000000000ef0 g    DF .text 0000000000000015  LINUX_2.6   __vdso_time  
```

From this we see that clock_gettime is in fact in there. So why was it not getting used? I donned the protective gear and the spelunking equipment and entered the caves of glibc, where I found several nested files, each #including a file from a parent directory, in an impressive attempt to abstract out per CPU, per OS and C library logic. I stared at that code for what felt like a long time, but it appeared to check lots of things, to eventually always end up calling __vdso_clock_gettime(). Weird.  

I then headed to __vdso_clock_gettime() in the Linux kernel where things finally became clear. It turns out the vdso code ITSELF will generate an actual system call for many timers you can request. In fact, this happens for all cases except CLOCK_REALTIME, CLOCK_MONOTONIC, CLOCK_REALTIME_COARSE and CLOCK_MONOTONIC_COARSE (as of Linux 3.13 up to 4.11-rc3).  

So that solved the mystery: the vDSO stuff was working, but it was itself causing an old fashioned system call. Perhaps the other timers are too difficult (or perhaps even impossible) to supply from the userspace context.  

Now that I knew what the problems was, I found lots of other places noting  issues with clock_gettime() performance, for example [here](https://github.com/Netflix/rend/issues/96#issuecomment-245816569) and [there](https://bugs.openjdk.java.net/browse/JDK-8006942), and other people have written [some harsh words about CLOCK_MONOTONIC_RAW](http://btorpey.github.io/blog/2014/02/18/clock-sources-in-linux/) that we attempted to use.  

It is my hope that the next person to run into this will find this blogpost before spending half a day learning about vDSO. Good luck!

