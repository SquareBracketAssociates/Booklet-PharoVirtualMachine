## Polymorphic Inline Caches

### The state machine

### A Code Patching Approach

### Unlinked Callsites

### Transitioning to Monomorphic Callsites

Linking on first call.
if possible


On monomorphic callsites we send the expected class of the receiver, to compare it with the actual class of the receiver, if incompatible abort the message send by calling the abort trampoline.

### Polymorphic Callsites

Creating PIC, relinking


(In cases where the rcvr object has multiple types at runtime, the sender method in can only send a call to one method, either X>>aMesssage or Y>>aMessage, to make the choice there is a new piece of machine code introduced as indirection, so the sender is patched with an address the new code called PIC that compares receiver calss tag with all possible classtags, if none correspond to it, the pic miss trampoline is called
The pic miss trampoline is a vm c runtime function that add the new machine code method to the pic, then executes it.

There are only 6 entries in the table, or 6 conditions to be verified.
In case the receiver can be multiple classes in the same hierarchy, the pic has as many entries with the same method.  

more than 6 entires is categorized as a megamorphic call site.
  )
**Design notes: outlining PICs.**

### Megamorphic Callsites

Looking up mics.



**Design notes: sharing stubs.**