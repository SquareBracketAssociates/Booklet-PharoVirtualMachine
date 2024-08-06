## Object Representation and Memory Access


Before delving inside the internals of the VM execution, it is important to understand the data it manipulates, in our case, Pharo objects.
This chapter presents in-depth how objects are represented in memory.
This will allow us to understand in further chapters how objects are created and mutated, how dynamic type checks are performed, and the different memory optimizations employed.

Since a couple of years, the Pharo VM uses the Spur memory model, designed and implemented by Eliot Miranda for the OpenSmalltalk VM {!citation|ref=Mira18a!}.
This model greatly improved the Garbage Collector (GC) and the complexity of JIT-compiled machine code.


### Background

This section defines some terminology necessary to understand this chapter, such as _word_, _nibble_, and _alignment_.
It is important to set up a common vocabulary because some of these terms are used differently by different technologies, and the Pharo VM terminology is no exception.

#### Data Units: Words and Bytes

Objects are stored in memory, thus it is important to understand the basics of memory organization.
Such an organization depends on the chosen computer architecture, which encompasses the memory and the processor.
One trait that characterizes a computer architecture and strongly influences the memory organization is its _bit width_ _i.e.,_ the number of bits used to represent the main processing unit in a processor.
For example, 64-bit machines are machines that have a bit width of 64.
Since nowadays the most common machines are 64-bit machines, we will focus our presentation on them.
However, the Pharo VM also supports 32-bit machines for compatibility with smaller devices.

![Word size and alignment on 32-bit and 64-bit architectures.](figures/architecture32vs64.pdf width=100&label=fig:32vs64Architectures)

Memory is conceptually divided into cells of 1-byte length, each byte using 8 bits.
Data is manipulated in units that group many bytes together.
A _word_ is a fixed unit of data with as many bits as the processor bit width.
This means a word is 64 bits long --or 8 bytes long-- in 64-bit processors, and 32 bits long --or 4 bytes long-- in 32-bit processors.
Figure *@fig:32vs64Architectures@* shows the memory layout on both 32 and 64-bit architectures.
The main difference between these two architectures is their word size.

Each 1-byte memory cell has an address, a unique identifier that can be used to read and write into that memory cell.
Addresses are typically restricted to fit in a word.
Memory addresses form a sequence ranging from 0 to the highest integer that can be represented in a word.
For example, on 64-bit processers the maximum address is 2^64, thus it can address 2^64 different bytes.
Memory cells are said to be contiguous if their addresses are contiguous.
Note that Figure *@fig:32vs64Architectures@* only shows addresses that are a multiple of a word, although each 1-byte cell also has an address.
Representing memory addresses as data is what is commonly referred to as pointers.

Usually, processors also define concepts such as _half-word_ and _double-word_.
We believe that such notations are confusing because they require context (Am I in a 32-bit architecture? 64-bit?), and thus we will not use them in this book.
Instead, when referring to a sub-word unit, we will use the exact number of bits.
For example, we will use _16-bit integer_ instead of short or half-word.

Moreover, when useful, we will use the term _nibble_ to refer to half a byte, or 4 bits.
A byte's high nibble and low nibble are the most and least representative halves of a byte respectively.
Interestingly, when a byte is written in hexadecimal it gets a two-digit representation, where each digit represents the value of each nibble.
For example, the byte 193 is represented as 16rC1 in hexadecimal.
Its high nibble has a value of 16rC (12 in decimal).
The low nibble has a value of 1.

#### Alignment

A processor's Instruction Set Architecture (ISA) generally provides instructions that read and write data from an address with different granularity.
For example, there are instructions to read/write individual bytes or entire words.
Although ISAs give a lot of freedom to developers, modern architectures and micro-architectures (how CPUs are implemented internally) behave better when data access is consistent and predictable.
Particularly, address alignment, _i.e.,_ its relative position in memory, is a property exploited by micro-architectures, compilers and programming languages for optimization.

A read from and write to an address are said to be aligned when the address is a multiple of the accessed element size, in bytes.
1-byte reads are always aligned because all addresses are multiples of 1.
8-byte reads are aligned when the address is multiple of 8.

We will see later in this chapter how the Pharo VM exploits alignment to implement tagged pointers, and optimize the reading of object header meta data.

#### Most and Least Significant Bytes and Bits

We mentioned before that memory cells are ordered.
Moreover, the individual bits within a single cell are ordered too.

We say that the _most significant_ bit in a byte is the bit that has the most value in a byte, and conversely for the _least significant_ bit.
We can define the most and least significant bytes in a word in the same way.
For example, the bit string 00000101 represents the number 5 in binary, and its least significant bit is represented by the rightest bit with value 1.

While we usually represent bytes from left to right (most to least significant), this is only the case with some architectures.
The order in which individual bytes of a word are stored in memory is again a trait of the computer architecture: the endianness.
An architecture is said to be _little endian_ if data bytes are stored from the least significant to the most significant, and _big endian_ otherwise.
Understanding endianness is important when reading and writing values smaller than a word.

Nowadays, the most popular architectures out there are little-endian.
This means that a 64-bit word with the value `16rFEDCBA987654321` is stored backward.
Let's imagine that the word is stored at address a.
- The lowest address -a- contains the least significant byte -16r21-.
- The highest address -a+7- contains the most significant byte -16r0F- (see Fig. *@fig:LittleEndian@*).
If we wanted to read the bytes in most-to-least significance order, then we need to iterate it backward: from a+7 to a.

![16rFEDCBA987654321 in 64-bits Little and Big-Endian.](figures/LittleBigEndian.drawio.pdf width=95&label=fig:LittleEndian)

### Object Layout

Pharo programs are made of objects which are, for the most part, allocated in memory and occupy space.
We call these objects _heap-allocated_ because they reside in a memory region managed by the VM called the _heap_, that we will explore in later chapters.

#### Object Formats
@sec:layout
@sec:formats

Pharo objects contain _slots_ that store the object's data.
Objects come in different kinds, determining the number of slots they contain and how their slot contents is interpreted.
The following table summarizes the most common types of objects and their variations.

| Type/Format    | # slots | Slot type       | Slot size    | Variations |
| -------------- | --------------- | --------------- | --------------- | ---------- |
| Fixed          | fixed           | reference       | word            | Ephemeron |
| Variable       | variable        | reference       | word            | Weak       |
| Byte indexable | variable        | byte            | 1/2/4/8 bytes   | -          |
| CompiledMethod | variable        | reference+byte  | word + 1 byte   | -          |

**Fixed and variable slots.** The number of slots in an object is either fixed, variable or a combination of both.
Fixed slots are those decided statically. For example, an instance of class `Point` declaring variables `x` and `y` has two fixed slots.
Variable slots are those determined at allocation time. The simplest example of variable slots are arrays, whose number of slots is specified as argument of the method `new:`.
Some objects may contain a combination of fixed and variable slots, as it is the case of the instances of `Context`.

**Slot type.** Slots contain either object references or plain data bytes.
Object references are pointers that reference other objects forming a graph, further explained in *@sec:references@*.
Plain data is stored as raw bytes in a slot, typically representing low-level data types such as integers or floats.

**Slot size.** The different slots in an object have a size that limits their contents.
Reference slots store an address, and thus are a word long.
Byte slots store a sequence of bytes, and thus element size can be 1, 2, 4 or 8 bytes.
All fixed slots in an object are of type reference.
All variable slots in an object are of the same type and are defined by its class.
For example, instances of the class `ByteArray` have 1-byte slots, and instances of `FloatArray` have 8-byte slots containing IEEE-754 double-precision floating point numbers.

**Weak and Ephemeron.** Weak and Ephemeron object formats are variations of the types described above, extending them with special semantics for the memory manager.
Weak objects are objects whose variable slots work as weak references (in contrast with strong references). That is, they don't prevent the garbage collection of the referenced object.
Ephemeron objects are fixed objects representing a key-value mapping whose value is referenced strongly as long as the key is referenced by objects other than the ephemeron.
These special objects will be further discussed in the chapters about memory management.

**CompiledMethod.** Compiled methods are variable objects that do not follow the conventions above.
They contain a word sized variable part storing object literals, followed by a 1-byte variable part storing _bytecode_ instructions.

#### Representing Objects in Memory

Objects in Pharo are represented as a contiguous memory region with space for a header and data slots.
The header contains meta data used for decoding the object internals, such as the size, its type and its class.
The data slots contain the object slots.
Figure *@fig:objectLayout@* illustrates the layout of a 3-slot object in both 32-bit and 64-bit architectures.

![Object layout and alignment on 32-bit and 64-bit architectures.](figures/objectLayout.pdf width=90&label=fig:objectLayout)

Each object has a mandatory base header that contains common information such as its class, its size and mutable bits for the Garbage Collector.
When objects are more than 254 words long, they are considered large, and their actual size is stored in an overflow header that precedes the base header.
The base and overflow headers have each a fixed size of 8 bytes \(64 bits\).
Headers are discussed in-depth in Section *@sec:header@*.

Data slots contain the different slots in an object.
However, there is not a one-to-one mapping between an object's slots and its underlying data slots.
Data slots are always 1 word long and their number is chosen to accommodate all the slots of the object.
Each reference slot occupies one data slot.
Byte slots, however, may occupy less than a data slot.
For example, in a 64-bit system, a data slot can accommodate 8 1-byte-long slots, 4 2-byte-long slots, 2 4-byte-long slots or 1 8-byte-long slots.

A special case arises when byte slots do not entirely fill an object's data slots.
For example, a 3-slot byte array occupies 3 bytes in a word-long data slot.
In such a case, the Pharo VM introduces padding _i.e.,_ filling space.
Such unused filler is used to guarantee that the next object is aligned to the 8-byte boundary, a property that can be exploited for both performance and the representation of immediate objects explained in Section *@sec:alignment@*

#### References and Ordinary Object Pointers
@sec:references

Objects reference each other, forming a directed graph.
Nodes in the graph are the objects themselves, edges in the graph are usually called _object references_.
In the Pharo VM, object references are called _ordinary object pointers_, or _oop_s for short.
There are two kinds of oops: object pointers to other objects and immediate objects.
Pointers work as normal pointers in low-level languages.
Immediate objects are objects encoded in invalid object pointers using a technique called tagged pointers that takes advantage of pointer alignment.

Every object in Pharo has an address in memory, which is the memory address of its base header.
An object `A` references an object `B` with an absolute pointer to `B`'s base header stored in one of `A`'s reference slots.
Figure *@references@* shows two objects forming a cycle. Each object has a single reference slot pointing to the other.
References point to the object base header.

![References to heap-allocated objects are pointers to an object's base header.](figures/references.pdf width=90&label=references)


### Immediate Objects

The object representation presented so far imposes a non-negligible overhead on small objects, because of the space taken by its header.
This problem becomes more visible with types that tend to have a large number of instances.
Integers, for example, are in theory infinite and are used very often by even the simplest programs to drive the execution of loops.
Representing integer objects as heap allocated objects becomes a bottleneck in an application very fast. Instead, Pharo uses a common optimization called _tagged pointers_ to represent integers and other common simple-valued immutable objects of the like.

#### Alignment and Padding
@sec:alignment

Pointer tagging exploits the alignment property of pointers.
In the Pharo VM, all objects are stored in memory aligned to 8 bytes.
That is, an object address, and thus its header, is stored always at the 8-byte boundary, regardless of the architecture.
Note that since an object header is always 8 bytes long, this means that the first data slot of an object is also always aligned.
Further, since all data slots are a word long, each subsequent data slot is aligned too.

To guarantee that objects are always aligned to the 8-byte boundary, the allocator inserts a padding at the end of an object filling it up to the next 8-byte boundary.
This happens in two cases: byte objects in general, and potentially all objects in 32-bit architectures.
Byte objects may contain a number of slots that does not entirely fill a data slot, as shown in Section *@sec:formats@*, thus requiring padding to fill a data slot.
Moreover, data slots in 32-bit architectures are 4 bytes long, and thus an odd number of data slots requires 4 bytes of padding.
Figure *@fig:objectLayout@* shows an example of how an object with 1 header and 3 slots is laid out both in 32-bit and 64-bit architectures.
- in 64-bit architectures, the object occupies 4 words, for a total of 32 bytes: 1 base header of 8 bytes, and 3 slots of 1 word each. The next free address \(32 in Figure *@fig:objectLayout@*\) is aligned and thus an object can start there.
- in 32-bit architectures, the object occupies 5 words, for a total of 20 bytes: 1 base header of 2 words \(8 bytes / 4 bytes per word\), and 3 slots of 1 word each. The next free address \(20 in Figure *@fig:objectLayout@*\) is not aligned. In this case, the allocator inserts a 4 byte padding to align the following object to the 8-byte boundary.

Padding represents wasted memory and could be avoided in 32-bit architectures by requiring an alignment to a word.
However, enforcing the same alignment in all architectures allows an overall code simplification by unifying the 64-bit and 32-bit implementations.

#### Pointer Tagging
@sec:tagging

Pointer tagging is a technique to represent a set of values without the need to perform an allocation.
Pointer tagging works by encoding (and disguising) such values within the pointer.
Such a technique is possible thanks to object alignment.

Since all Pharo objects are aligned to 8 bytes, all object references will be multiple of 8 and have the form `xxx...xxx000` in binary, where its 3 lower bits are zero.
Pointer tagging exploits this property by encoding data within the least significant bits of a pointer.
With these three bits, we can encode up to 7 different tags (111, 110, ... 001) that tell us how to interpret the most significant bits.

The main advantage of this technique is to save storage space for common objects, which indirectly improves on data locality and CPU cache behavior.
Its main drawback is the overhead incurred by runtime type checks: we need to verify if a pointer is tagged or not before operating on data.

Another consideration for tagged pointers is that they significantly reduce the number of values that can be represented.
For example, tagged integers in the described schema have a maximum precision of 61 bits (+ 3 bits of tag = 64 bits).
In the following sections we explain variable-sized tags and boxed values.
Variable-sized tags help us mitigate this problem in 32-bit architectures, in which the loss is 10% (3 bits out of 32).
Moreover, a combination of pointer tagging and boxed values allows us to represent larger numbers by assuming that such larger numbers will be less common.

Currently, Pharo supports integers, characters and floating point numbers as immediate objects.
In 64-bit architectures, they use the tags `001`, `010` and `100` respectively, as shown in Figure *@fig:64bitsimm@*.
In 32-bit architectures, floats are not represented as immediate objects, integers present a 1-bit tag `1`, while characters are represented with the 2-bit tag `10`, as shown in Figure *@fig:32bitsimm@*.

![64-bit immediate objects.](figures/64bitsImmediate.pdf width=100&label=fig:64bitsimm)
![32-bit immediate objects.](figures/32bitsImmediate.pdf width=100&label=fig:32bitsimm)

The following code shows how we can tag, untag and verify if a value is taggable.
The next sections explain the specific designs for integers, characters and floats, for 64 and 32 bits.

```caption=Testing Small integers.
"Convert native values into objects and vice-versa"
integerValue := objectMemory integerValueOf: aSmallIntegerObject.
aSmallIntegerObject := objectMemory integerObjectOf: integerValue.

characterValue := objectMemory characterValueOf: anImmediateCharacterObject.
anImmediateCharacterObject := objectMemory characterObjectOf: characterValue.

floatValue := objectMemory smallFloatValueOf: anImmediateFloatObject.
anImmediateFloatObject := objectMemory smallFloatObjectOf: floatValue.

"Check if an OOP is a tagged integer, character or float"
objectMemory isImmediate: anOop.
objectMemory isIntegerObject: anOop.
objectMemory isCharacterObject: anOop.
objectMemory isImmediateFloat: anOop.

"Check if a native value can be encoded as a tagged value"
objectMemory isIntegerValue: aValue.
objectMemory isSmallFloatValue: aValue.
objectMemory isCharacterValue: aValue.
```

#### Immediate Characters and Integers in 64 bits

In 64 bits, all immediate objects are represented as 61 bits of value and 3 bits of tag.
In Pharo, immediate integers are instances of the class `SmallInteger`, and immediate characters are instances of the class `Character`. `SmallInteger` immediate objects range is between \[-2^60,2^60-1\] and are represented in two's complement.
For example, the binary value `2r1010001` represents the untagged value `2r1010` which has the decimal value `10`.

```caption=On 64 bits, SmallInteger instances are encoded in 61 bits.
SmallInteger minVal == (2**60) negated
>>> true

SmallInteger maxVal == (2**60-1)
>>> true
```

Immediate characters encode their Unicode codepoint in the 61 value bits.
This is, so far, enough to represent all Unicode codepoints: the maximum valid codepoint nowadays is 16r10FFFF, which requires only 21 bits.

#### 32-bit Immediate Integers and Variable Tags

In 32-bit architectures using 3 bits of tag would leave 29 bits left to represent integers.
Instead of choosing this fixed tag representation, the 32-bit VM uses variable tagging.
That is, different values use a different number of bits for tagging.
Thus, tagging is carefully designed to avoid conflicts and ambiguities.

Immediate integers are tagged with a single bit and use the remaining 31 bits to encode a signed integer in two's complement.
The range of immediate integers is \[-2^30,2^30-1\]. For example: `2r10101` represents untagged binary number `2r1010` which has the decimal value `10`.

Figure *@fig:32bitsimm@* illustrates the entire 32-bit tagging schema, with tag bits in gray:
- `00` is an aligned address and therefore it is an object pointer in the heap.
- `*1` is the tag for immediate integers.
- `10` is the tag for immediate characters.

#### 64-bit Immediate Floats

In 64-bit architectures, the Pharo VM represents floats as immediate objects with the tag `100`.
The tagged value is an IEEE-754 64-bit double-precision floating point number accommodated in 61 bits.
However, to accommodate the 64 bits into 61 bits, immediate floats give up 3 bits in the exponent offset, storing only 8 out of 11 bits of exponent.
The VM verifies that only immediate floats that do not lose information in this format are encoded as immediates.
For floats that do not satisfy this constraint, floats use a boxed representation as explained in Section *@sec:boxing@*.

![64 bits `SmallFloat` immediate.](figures/64bitsFloatImmediate.pdf width=100&label=fig:64bitsfloatimm)

Figure *@fig:64bitsfloatimm@* shows the structure of a `SmallFloat`.
The sign bit is moved to the lowest bit of the tagged value, and the highest 3 bits of the exponent are lost.

#### Boxed Native Objects
@sec:boxing

Numbers that cannot be encoded as _immediates_ need to either gracefully fail or implement a fallback mechanism.
After arithmetics, if a number does not fit in the 61 bits of a tagged pointer, the runtime creates a boxed object with the result.
Boxed numbers are byte objects that contain the native number encoded in their byte slots.

In Pharo, boxed numbers include large integers (instances of `LargePositiveInteger` and `LargeNegativeInteger`) and boxed floats (instances of `BoxedFloat64`).
Large integers implement variable-sized integers and represent arbitrary large integers as a string of bytes.
Boxed floats are instead of fixed size: they represent IEEE-754 double-precision floating point numbers, and store the corresponding float in 8 bytes.

### Object Header
@sec:header

Each _heap-allocated_ object has a header that describes it.
The header consists of a mandatory base header, and in the case of large objects, an overflow header that includes the object size.
This section explains the overall design of the object header and each of its fields in detail.

#### Base Object Header

The base object header contains meta data that the Virtual Machine uses for several purposes such as decoding an object's contents, maintaining garbage collection state, or even doing runtime type checks.
Regardless of the architecture, the base header is 64 bits length, which means it is 2 words in 32 bits and 1 word in 64 bits as shown in Figure *@fig:objectheader@*.

![Base Object Header.](figures/ObjectHeader.pdf width=100&label=fig:objectheader)

This header is composed of several fields, marked with different colors in the figure.
From the most significant to the least significant bits, the fields are as follows:

- **Object size (s).** This field contains the number of data slots in the object, padding included. For example, the object size of a byte array of 14 byte slots is 2 data slots. Padding is computed from the **object format** field below.  Objects with more than 254 slots are considered large, are given 255 as object size and an overflow header as explained in Section *@sec:large_objects@*.
- **Object hash (h).** This field contains 22 bits representing the identity hash of the object.
- **Object format. (o)** This field contains an enumerated value that identifies the format of the object as described previously in Section *@sec:formats@*. The exact values of this field are explained in Section *@sec:format_encoding@*.
- **Class index (c).** This field contains the index at which the object's class is found in the class table.
- **Miscellaneous (x).** The remaining 7 bits illustrated with a green `X` are reserved for different reasons:
  - 1 bit is reserved for immutability.
  - 1 bit is reserved to mark the object as pinned. Basically, a pinned object is an object that cannot be moved in memory by the GC.
  - 3 bits are reserved for the GC: isGray (for tri-color marking), `isRemembered` (for the remembered table from old space to young space) and `isMarked` (for the GC mark phase).
  - 2 bits are not used.

Notice that the fields of the header are not all contiguous: miscellaneous bits are interleaved in between them.
The header has been designed so that commonly accessed fields are aligned to a byte or 2-byte boundary.
This design largely simplifies the decoding of the header, which boils down to a `load` and a `bit-and` instruction sequence.
This simplifies the JIT compiler and generates better-quality machine code.

#### Large Objects and the Overflow Header
@sec:large_objects

The object size field is 8 bits long and cannot store values larger than 255.
It is, however, desirable to have large arrays or strings with thousands of elements.
For this purpose, large objects contain an extra header, namely the overflow header, preceding the base header _i.e.,_ it is placed contiguous to the base header but in a lower address.


!!note The address of an object is always the one of its base header regardless if it has an extra header or not.


The overflow header is 8 bytes long and contains the object size. It allows for very large objects with sizes of up to 2^64 words, which is largely sufficient.

SD: Would be nice to have a diagram

When an object has an overflow header, the object size field in the base header is marked with the value 255.
The pseudocode in Listing *@list:numSlots@* shows how to obtain the number of data slots of an object.

```caption=Extracting the number of data slots in an object&label=list:numSlots
numSlotsOf: objOop
	numSlots := self baseNumSlotsOf: objOop.
	^numSlots = 255
		ifTrue:  [ self readOverflowHeaderOf: objOop]
		ifFalse: [ numSlots ]
```


#### Class References and Class Table

Each object includes a reference to its class in its header.
However, for space reasons, an object does not store the absolute address.
An arbitrary address requires an entire word, which would add a non-negligible memory overhead.
Instead, classes are stored in a table, and objects store the index of the class in the class table.
The only exception to this is immediate objects that do not contain a header: the object tag is used as its class index.

Since programs are expected to have a low number of classes, class indexes are limited to 22 bits.
22 bits of class index support a maximum of 4 million of classes, which will be largely sufficient for most applications for a long time.

The class table is organized into 4096 pages of 1024 elements.
The 12 most significant bits in the class index indicate the page index. The 10 least significant bits in the class index indicate the index of the class within the page.

Each class stores its own index as its hash.
This allows the VM to get the index of a class without iterating the entire class table, and to guarantee a unique identity hash per class.

![Finding a class in the class table using its index.](figures/classtable.pdf label=classtable)

#### Encoding of the Object Format Field
@sec:format_encoding

The object format field contains 5 bits that are used to identify the object's format explained in Section *@sec:layout@*.
An object's format is encoded as a 5-bit integer:

- `0`: 0 sized object _e.g.,_ `nil`, `true`, `false`
- `1`: fixed size object _e.g.,_ a `Point`
- `2`: variable-sized object with no instance variables _e.g.,_ an `Array`
- `3`: variable-sized object with instance variables _e.g.,_ a `Context`
- `4`: weak variable sized object with instance variables  _e.g.,_ a `WeakArray`
- `5`: ephemeron object
- `7`: forwarder object
- `9` : 64 bits indexable
- `10 - 11`: 32 bits indexable
- `12 - 15`: 16 bits indexable
- `16 - 23`: 8 bits indexable
- `24 - 31`: compiled methods
- `6` and `8` : unused

From the list above notice that zero-sized objects, instances of classes that define no instance variables, have their own format identifier.
Variable objects with instance variables are marked separately from those without instance variables.

Special attention needs to be given to byte objects, where format `9` identifies byte objects with 64-bit slots, formats `10` and `11` identify byte objects with 32-bit slots, and so on.
All byte objects having slots smaller than a word encode their padding in the format: the first format in each category (`10`, `12`, `16`, `24`) is used for objects that require no padding.
Subtracting the base format from the format returns the number of padding bytes.
For example, a byte array with format 21 has 21-16 = 5 bytes of padding.

The object format and its padding are necessary to compute the number of actual slots in the object.
For example, given a byte array with format 21 and slot size 10, we can compute its size as 10 data slots * 8 bytes - 5 padding bytes = 75 bytes.

CompiledMethods are similar to byte arrays in terms of padding.
However, they use a different format so the runtime can differentiate them from normal byte arrays.

### Classes

HERE!

### Conclusion

In this chapter, we explored how Pharo objects are represented in memory using the Spur memory model supporting both 32 and 64 bits.
From a user's perspective, objects have a set of fixed and variable slots.
Slots contain references to objects or plain byte data.

Under the hood, most objects are allocated in the heap and possess a header with meta data.
Object slots are stored in word-large data slots, and padding is inserted so that all objects are aligned to the 8-byte boundary.
We exploit object alignment to implement integers, floats, and characters as tagged pointers.
Tagged pointers use the least significant bits of a pointer to encode a type, and the most significant bits to encode a value.
Thus, tagged pointers help us represent objects without the overhead of a memory header.

### References
@sec:references

- Spur [http://www.mirandabanda.org/cogblog/2013/09/05/a-spur-gear-for-cog/](http://www.mirandabanda.org/cogblog/2013/09/05/a-spur-gear-for-cog/)
- [https://clementbera.wordpress.com/category/spur/](https://clementbera.wordpress.com/category/spur/)
- [https://clementbera.wordpress.com/2018/11/09/64-bits-immediate-floats/](https://clementbera.wordpress.com/2018/11/09/64-bits-immediate-floats/)
- [https://clementbera.wordpress.com/2014/01/16/spurs-new-object-format/](https://clementbera.wordpress.com/2014/01/16/spurs-new-object-format/)
- [https://clementbera.wordpress.com/2014/02/06/7-points-summary-of-the-spur-memory-manager/](https://clementbera.wordpress.com/2014/02/06/7-points-summary-of-the-spur-memory-manager/)
- [http://www.mirandabanda.org/cogblog/category/spur/page/3/](http://www.mirandabanda.org/cogblog/category/spur/page/3/)
