---
layout: post
title:  "Investigating Linux I/O. Or: stupid kernel tracing tricks."
date:   2015-12-18 13:16:00
published: false
categories: systems wankery
---

If you define a hobby as something that you put way too much effort into to be
justified by the payoff received, then understanding the performance of programs
in Linux is *definitely* a hobby of mine.

I've slowly picked up the sysadmin's stable of tricks and arcana, which boils down to
running htop, and profiling whatever seems to be the obvious bottleneck
(memory, CPU, etc.). Then, if none of that yields hope, I generally aimlessly read selected works of
[Brendan Gregg](http://www.brendangregg.com/linuxperf.html) and hope to absorb some knowledge
from osmosis.

However, one area that I've never been happy with is debugging I/O bottlenecks. There are good
tools for profiling CPU bottlenecks depending on your language. Likewise, memory blowup is
generally reproducible in a heavily instrumented environment (e.g. in valgrind). But I/O 
performance is the member of the resource triumvirate that I've always felt like I'm
casting chicken bones at tea leaves by juggling `iostat`, `iotop`, RAID specific utilities,
and ancillary utilities like `vmstat`. It requires a lot of crazy theorizing about what the
program is actually doing behind the scenes from incomplete data, and I prefer other hobbies for
[crazy theorizing](http://darthjarjar.com/).

So let's see if we can't get some more insight into what's going on with a program's
I/O usage in a more systematic manner.

### Picking our battles

One challenge is that other than perhaps virtual memory, the I/O subsystem has the biggest
disconnect between what a program thinks is going on and what the machine is actually doing.

Seriously. There are more lies in the I/O subsystem paths than there are uncles who totally work at Nintendo
in the average elementary recess cohort. Language libraries
lie to programs about what's been flushed to the operating system, the operating system lies
to user space about what's been sent to the filesystem, the operating system lies to the filesystem
about what's been sent to the block device driver, the block device driver lies to the operating system
about what's been sent to the actual block device hardware, and the block device hardware lies to
the operating system about what's been synced to the actual disk as opposed to a cache.

So we need to pick some layer to believe or else become German nihilists. The closer we get to the
application, the easier it is to walk back to the application code that's causing the problems. Besides,
most of the existing I/O tooling is low-level (e.g. `iostat` is at the block device level), so this'll
be something different.



{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}
