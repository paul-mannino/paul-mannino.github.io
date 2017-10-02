---
layout: post
title:  "Diving into .NET Interface Performance"
date:   2017-09-10 12:44:57 -0500
categories: C# JIT Performance
---
I was trying to prove a point about C# interfaces to a coworker and came across a performance quirk I couldn't explain. Here's an example that gives the gist of it<sup>[1](#fn1)</sup>:
```csharp
public interface IFoo
{
  void F();
}

public class Foo : IFoo
{
  int x;
  public void F()
  {
    x = 1;
  }
}

class Program
{
  static void Main()
  {
    Foo a = new Foo();
    IFoo b = new Foo();
    const int m = 1000000000;
    var sw = new Stopwatch();
    sw.Start();
    for (int i = 0; i < m; i++)
    {
      a.F();
    }
    sw.Stop();
    Console.WriteLine("Class, {0} ms", sw.ElapsedMilliseconds);

    sw.Restart();
    for (int j = 0; j < m; j++)
    {
      b.F();
    }
    sw.Stop();
    Console.WriteLine("Interface, {0} ms", sw.ElapsedMilliseconds);

    // Output:
    // Class, 36 ms
    // Interface, 240 ms
  }
}
```
Invoking the class method was nearly 7 times faster than invoking the interface method. Huh.

It's pretty obvious that something like this would be ~~unlikely to ever make a practical difference in anything I write~~ extremely interesting and enriching to learn more about! So I decided to dig a little more.

I had some notion that when code is JIT compiled, lots of magical optimizations happen. So maybe it's related to that. Sure enough, running the code in Debug mode and checking _Options > Debugging > General > "Suppress JIT optimization on module load"_ makes this speed difference vanish. Still, I curious why JIT compilation was handling one case so much better than the other.

#### __Getting familiar with native code__
First thing I did was take a look at the native code (Ctrl + Alt + D in Visual Studio while debugging). Here's code for the loop containing the ```IFoo::F``` invocation:
 ```nasm
02AF2E76  xor         esi,esi  
02AF2E78  mov         ecx,edi  
02AF2E7A  call        dword ptr ds:[1120024h]  // a.F() call
02AF2E80  inc         esi  
02AF2E81  cmp         esi,2FAF080h  
02AF2E87  jl          02AF2E78  
...
02AF2EB0  mov         dword ptr [ecx+4],1  // stepping into a.F() call
02AF2EB7  ret
 ```
In case you've never seen assembly before, I'll try to give a very brief description of what you're looking at. Every line consists of a line number in hex, followed by an operation, followed by any operands that operation requires (which can be registers, addresses in memory, or constants).

The following snippet is an instruction that xor's the esi register with itself, and stores the result (0) in the esi register:
```nasm
02AF2E76  xor         esi,esi
```
This is how the program initializes j at 0 for the second loop. Most assembly instructions are (sort of) intuitively named: ```mov x y``` sets the value of ```x``` to ```y```, ```inc x``` increments ```x```, ```cmp x y``` compares the values of ```x``` and ```y``` and can be coupled with jump (```jmp```) instructions to establish looping behavior. If you squint a little bit, you can make out the code for a ```for``` loop. [This](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html) is a pretty nice resource if you want to learn a little more about assembly.

#### __Comparing IFoo and Foo function calls__
In the IFoo::F loop on line ```02AF2E7A```, you can see a call operation to execute code that resides at the specified piece of memory. You'd have no way of knowing this from looking at it, but since I stepped through the code, I can tell you that you'll end up at line ```02AF2EB0```. Take another look at the instruction at that line:
```nasm
02AF2EB0  mov         dword ptr [ecx+4],1  
```
It's doing pretty much what you'd expect: it sets the value of x in a Foo instance to 1 (```ecx``` in this case holds the address for the Foo instance, and x is stored at a 4 byte offset).

Here's code for the ```Foo::F``` loop:
```nasm
02AF2E65  xor         eax,eax  
02AF2E67  mov         dword ptr [esi+4],1  
02AF2E6E  inc         eax  
02AF2E6F  cmp         eax,2FAF080h  
02AF2E74  jl          02AF2E67  
```
The key difference to notice is that Foo::F loop doesn't contain a call instruction at all. The code the sets the value of x to 1 is executed directly in the loop code. This looks like one of the main reasons the ```Foo::F``` loop executes so much faster: it can (sometimes) take small functions inject the code directly at the call site. This is called __inlining__. Curiously, the compiler only seems to be able to inline non-interface function calls, even though the executing code is otherwise the same.

#### __When does inlining fail?__
Unfortunately, it doesn't seem like there's a ton of official documentation about when and why the JIT does the things that it does. I did a little more digging and found this [MSDN blog post](https://blogs.msdn.microsoft.com/davidnotario/2004/11/01/jit-optimizations-inlining-ii/) from 2004 explaining why JIT can fail to inline methods:
>Virtual calls: We don't inline across virtual calls. The reason for not doing this is that we don't know the final target of the call. We could potentially do better here (for example, if 99% of calls end up in the same target, you can generate code that does a check on the method table of the object the virtual call is going to execute on, if it's not the 99% case, you do a call, else you just execute the inlined code), but unlike the J language, most of the calls in the primary languages we support, are not virtual, so we're not forced to be so aggressive about optimizing this case.

I'm guessing a decent amount has changed since that post was made, but it jives with present-day reality and is probably the best answer I'm going to get. Since the actual object behind the variable could change at any time in the execution of the code, inlining would effectively be impossible in many cases. Even though it seems like it might be occasionally possible to make this optimization, that's not a choice developers chose to make.

That doesn't totally clear things up... some of you might know that interface implementations always treated as virtual. But compare the IL code generated if you mark ```Foo::F``` as virtual:
```
.method public hidebysig newslot virtual
        instance void  F() cil managed
```
with the code you get if you don't:
```
.method public hidebysig newslot virtual final
        instance void  F() cil managed
```

If you don't explicitly use the ```virtual``` keyword, interface implementations are marked ```virtual``` and ```final```. My best guess is that the presence of ```final``` is enough to tip off the JIT compiler that inlining is possible since you can assume the target of the call won't change. And as expected, if you do decide to explicitly mark ```Foo::F``` as virtual in the code, the JIT compiler does not inline anything and performance comes much closer to parity.

<br/><br/>
#### __Footnotes__
<a name="fn1">1</a>: Benchmarking code is simplified a bit, but results stay close to the same when you control for warm-up and ordering.
