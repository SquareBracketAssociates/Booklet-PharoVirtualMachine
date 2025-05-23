### The Free List


When a new object should be allocated, the VM checks first if there is an object that was already allocated and garbage collected of the given size that can be reused. Else it allocates a new object by "cutting" and splitting a large free object.
The management of such used _cells_ is managed by the _Free List_ and its companion the _Free Table_.
The Free List manages chunks of memory below numFreeLists \(i.e., 63 in 64 bits architecture\).
The Free Table manages larger chunks of memory.



### Free cells in memory


The free list is a structure that keeps information about the memory that can be used to allocate objects.
It refers to free cells \(which are special object tagged as Free class\).
The minimum size of a free cell is two words: one for the object header and the next. 
Such minimal cells are used to allocate an object with one single slots \(because one slot is for the header and the second one is for the slot\).

The free cells stay in the memory where they are allocated as shown in Figure *@freeListMemory@*.
In Figure *@freeListMemory@*, there are two  objects of size 5 that are free. Such objects are structured in a linked list of objects of the same size. A free cell is an element of a linked list.
The first object then points to the next free object of the same size. The second object is the last one of this size so its next slot is empty. In addition when the size allows it, the free object has a previous pointer in addition to the next one \(implementing a double linked list\).

![Free cells of size 5 in the memory.](figures/freeListMemory2.pdf width=100&label=freeListMemory)


### Free list


The free list is a structure that keeps tracks of free cell objects based on their size.
Figure *@freeListMemory@* shows that the free list linked lists are built using the free objects.

On 64 bits, the free list has 63 slots to keep a free cell linked list per number of slots of the object.
The first element of the free list is a pointer to the free tree \(a tree structure that manages chunks of memory larger than 63 words\).

% +Free list first view.>figures/freeListMemory2.pdf|width=100|label=freeListMemory2+

The free list first element is a pointer to the tree table, the next elements are pointers to linked-list of free objects, one linked-list per object size.

![Free list view is a table of linked-lists of free objects.](figures/freeListBasic.pdf width=100&label=freeListBasic)

### Free Table
