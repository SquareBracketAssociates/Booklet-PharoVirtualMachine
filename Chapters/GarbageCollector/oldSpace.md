### The Old Space


At startup, the VM allocates a memory map as depicted on Figure *@memoryMap@*.
Initially the OldSpace has only one segment but it can then vary dynamically by allocating and freeing segments.
Figure *@oldSpace@* shows an example of an old space with two segments.

![The Old Space with two segments.](figures/oldSpace.pdf width=100&label=oldSpace)

The first segment of the old space stores at least these objects:

- nil
- true
- false 
- classTable i.e. an array of all the classes in the image; in its header, an object does not directly store a reference to its class but the index of its class in this table.
- remember set
- freeTree
- freeLists
- bridge byte array
