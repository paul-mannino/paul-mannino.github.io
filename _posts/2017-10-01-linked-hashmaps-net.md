---
layout: post
title:  "Linked Dictionaries in C#"
date:   2017-10-01 12:44:57 -0500
categories: C# data-structures
---
.NET dictionaries can be kind of sneaky, because they get you used to the idea that key/value order doesn't change.
```csharp
var dictionary = new Dictionary<int, char>();
for (int i = 65; i < 70; i++)
{
    dictionary.Add(i, (char)i);
}

foreach (var key in dictionary.Keys)
{
    Console.Write("[{0}, {1}], ", key, dictionary[key]);
}
// prints "[65, A], [66, B], [67, C], [68, D], [69, E]"
```
As long as you're only adding keys, the order in which you've added keys is preserved for iteration. But once you start deleting keys, all bets are off:
```csharp
dictionary.Remove(67);
dictionary.Add(85, (char)85);

foreach (var key in dictionary.Keys)
{
    Console.Write("[{0}, {1}], ", key, dictionary[key]);
}
// prints "[65, A], [66, B], [85, U], [68, D], [69, E]"
```
Internally, the .NET dictionary manages dictionary entries in an array. When key-value pairs are deleted, the dictionary manages a linked list of slots in the array that get freed up. On subsequent adds, the dictionary first looks for space in this linked list of free slots. This helps keep the array of entries compact and reduces the need for costly array resizes. Since the Dictionary's Enumerator iterates over this entries array in order, key-value pairs that get added into freed up slots will be potentially hit sooner during iteration, as seen above.

#### __LinkedHashMap__
What happens if you want the order of iteration to be based solely on the order in which the key or value was added? Well, Java has a LinkedHashMap for this. The keys and values are managed with a linked list in conjunction with the usual dictionary internals. All the standard operations are made slightly more costly, but you get predictable FIFO iteration out of it.  

Why doesn't .NET have a LinkedHashMap/LinkedDictionary? Guessing it wasn't important enough to be part of the fairly lean core collections library. Nevertheless, I've wanted something like this frequently enough that it was worthwhile for me to implement it in C#.

Here's a sketch of my implementation:
![LinkedDictionary implementation]({{ site.url }}/assets/linkeddictionary.png)

The linked list in purple keeps track of the order of the entries as they are added. I used a cyclical double-linked list because it lets you easily implement an iterator that goes in the reverse direction.

The LinkedDictionary class works pretty much the same as a regular Dictionary, except all mutations will appear in FIFO order during iteration:
```csharp
var dictionary = new LinkedDictionary<int, char>();
for (int i = 65; i < 70; i++)
{
    dictionary.Add(i, (char)i);
}

dictionary.Remove(67);
dictionary.Add(85, (char)85);
dictionary[66] = 'Q';

foreach (var key in dictionary.Keys)
{
    Console.Write("[{0}, {1}], ", key, dictionary[key]);
}
// prints "[65, A], [68, D], [69, E], [85, U], [66, Q]"
```
If you want to check out the source code, [you can check it out here](https://github.com/paul-mannino/DataStructures).
