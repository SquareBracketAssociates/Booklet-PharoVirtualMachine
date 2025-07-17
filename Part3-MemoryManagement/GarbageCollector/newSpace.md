### The New Space


The new space is the region where young objects are allocated.
It is the place where the generational garbage collection is taking place.
It is commonly named a _scavenger_.

#### Minimal objects: Liliputians

> **Note:** In Spur, the minimum size of an object is the size of its header and one slot.
> This means that an object without slots will contain one extra hidden slot.
> In 64 bits, the smallest object is 16 bytes long: 8 bytes of header + 8 bytes of one slot.
> In 32 bits, the smallest object is also 16 bytes long: 8 bytes of header + 8 bytes of  since it is padded on 64 bits too.


This minimum size for any object is important for some features of the VM \(GC, become\). For example, when the VM needs to move an object in memory, it ensures that there is always enough space to put a **forwarder object** at the previous object position. A **forwarder object** is a 2 words value that encodes the new position of the moved object.

### New Space Memory Layout

The new space is divided into three areas \(Eden, Future and Past\), as shown by Figure *@newSpace@*.
The Eden \(5/7 of the new space\) and two other areas \(1/7 of the new space each\) that alternatively play the role of _past_ and _future_ space.
The VM always allocates new objects into the Eden space if there is enough space.

![The NewSpace structure composed of the Eden, and two Past and Future regions.](figures/newSpace.pdf width=60&label=newSpace)

### The Scavenger


The scavenger is a copy collector responsible for managing the new space memory region.
The scavenger is periodically triggered on some events such as: the Eden is full i.e., remaining space reached a predefined threshold. There are multiple scavenger policies:
- TenureByAge: it is the default policy. It copies surviving objects either in future space or old space depending on a threshold \(cf `SpurGenerationScavenger>>shouldBeTenured:`\). Surviving objects in past space with addresses below this threshold are tenured i.e., copied to the old space instead of future space. Initially, the threshold value is 0 meaning that the scavenger will not tenure any surviving objects, they are copied to future space. At the end of the scavenge, the scavenger updates the treshold if the future space is filled at more than 90%. The next scavenge will then tenure objects.
- TenureByClass: this policy consists in tenuring objects instance of a specific class.
- TenureToShrinkRT: this policy consists in tenuring objects to shrink the remember table i.e., minimizing objects in the old space that reference objects in the new space.
- DontTenure: this policy consists in not tenuring any objects i.e., threshold is fixed at 0.
- MarkOnTenure: the full mark and compact GC of the old space calls the scavenger with this policy. The threshold will be 0.


The VM allocates new objects in the Eden space.
When a newly allocated object address in the Eden space reaches the scavenge threshold, the scavenger is triggered.
To free some space in the Eden, the scavenger does not iterate over all objects it contains.
It computes surviving objects i.e., referenced by **root objects**.
There are three kind of roots:
- objects referenced in the stack i.e., used as receivers or arguments;
- objects stored in the remembered set. This set contains all objects allocated in the old space that contains at least one reference to an object in the new space;
- special objects known by the VM such as: nil, true, false, class table, etc.


!!note Add a picture with pointers from the oldspace to the newspace \(and that are in the remembered set\)

The scavenger starts by copying roots references allocated into Eden or past spaces into the future space.
Then, it traverses these copied objects and copies their referenced objects that reside into Eden or past space into the future space.
At the same time, traversed references are fixed to correctly reference the copied objects.
Similarly, root objects references are also updated.
Finally, the future space contains all reachable objects from roots that were present into the Eden and past spaces.
Moreover, all their references are updated to correctly point to objects into the future space.
If the future space is filled during the scavenge, some objects are tenured i.e., copied into the old space.

There are multiple strategies regarding the tenuring of objects:
- By age, using the addresses in the past \(older objects have smaller addresses\).
- Tenure to shrink the remembered table, it is used when the remembered set is too big.
- Instances of a given class, it is used by the `someInstance` primitive before its execution making all instances available into the old space.


Once the scavenge is finished, the future and past spaces are switched; it just means that the future space is now considered as the past space and vice-versa.

!!note Add a figure showing the switch


### Example of a Scavenger pass


```
Roots references: A, C

B -> D
C -> A
A -> B
A -> C
E
```


- Step 1: copy roots references

```
future space: A, C
```


- Step 2: go over the first surviving object \(A\)

```
future space: A, C, B
(C was already copied, so we just update the reference)
```

- Step 3: go over the second surviving object \(C\)

```
future space: A, C, B
(C points to A, but A was already copied, so we just update the reference)
```


- Step 4: go over the next surviving object \(B\)

```
future space: A, C, B, D
```

- Step 5: go over the next surviving object \(D\)

```
Nothing to do, and nothing new in future space
```

- Step 6: exchange past and future spaces


### to be continued
