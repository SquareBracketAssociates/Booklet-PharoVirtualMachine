## Weak Objects

References to objects are actually divided into two categories: strong and weak.

### Strong and Weak references


Strong references are visited by the garbage collector, and they are used to calculate the reachability of an object.
An object that is not reachable through a path of strong references from any of the given roots will be collected.

Weak references are not visited by the garbage collector, and they are not analyzed by the garbage collector.
An object referenced only by weak references will be garbage collected.

In the Pharo VM, a reference is a pointer.
This pointer does not encode the strong/weak information.
The reference is considered to be strong or weak depending on which object is holding it.
An object has a specific format which defines if its references are all strong or all weak \(see *@Ephemerons@* for a special case\).
We cannot mix strong and weak references in an object.
Therefore normal objects have all strong references.
We call weak objects the objects that have all weak references.

### Strong and Weak Objects


The format of an object encodes the instance specification \(_instSpec_\).
The format of an object is stored in its header (see Figure *@fig:objectheader@*).
The instSpec of a weak object is 4.
This instance specification describes a layout that contains both a fixed part \(instances variables\) and a variable part.
Every reference contained by this object will be weak.
An example of this are the instances of the WeakArray class.

While an object referenced by a weak object is reachable, the reference in the weak object will be valid, and will point to the object. 
A weak object is collected as any other.
When the garbage collector collects an object referenced by a weak object, the reference in the weak object will be set to nil.

If during garbage collection a weak reference is set to nil, a semaphore is signaled to allow the image to handle it.
This semaphore is used in the image side to implement a finalization process.
The semaphore is registered in the SpecialObjectArray *@specialObjectArray@* in the 42th position.

### Weak reference collection during scavenging


The handling of weak references is done during the execution of the garbage collector.
In this subsection we will focus on the garbage collection of the New space: the scavenge.
The scavenger copies the weak objects but does not scan the references it contains.
When we scavenge an object, only the strong references are visited.
A weak object only has weak references, so the references are not visited.
Once the weak object is copied, a reference to it is kept in a data structure that we will call the weak list.
The number of strong references in an Object is calculated from its format \(SpurMemoryManager >> #numStrongSlotsOf:format:ephemeronInactiveIf:\).

After all the Eden and the Past spaces have been scanned, the weak list is iterated.
For each of the objects in the weak list, the scavenger attempts to update each of its references.
If the reference is pointing to a object in the Eden or Past space that has been forwarded, the scavenge follows the forwarder and updates the reference to the new address in the Future space or in the Old space, because the refered object could have been tenured.
If the reference points to an object that has not been copied to the Future space nor been tenured \(ergo it's not a forwarder\), the reference in the weak object is set to nil.
If any reference was set to nil for a given weak object, it will be counted as pending finalization \(instanceVariable : StackInterpreter >> #pendingFinalizationSignals\).
This variable is checked in the `StackInterpreter >> #checkForEventsMayContextSwitch:` method and if there is pending finalization, it signals the TheFinalizationSemaphore.
Then the variable is cleared.

TODO: #diagram SpurGenerationScavenge >> #processWeakLinks.

During the copy of a weak object, this object may have been tenured.
If an object in the weak list has been tenured, it is also checked to see if it should be in the remembered set.
If a weak object's referenced object has been tenured, this weak object may be removed from the remembered set as well.

Objects in the Old space that reference objects in the new space are kept in the remembered set.
Objects in the remembered set are roots of the scavenging process.
Therefore the remembered objects in the remembered set need to update their references.
For weak objects in the remembered set, the references are updated or set to nil.

If the old object in the remembered set does not have references to objects in the New space anymore, it is removed from the remembered set.


### Weak list structure


When copying an object, the scavenger leaves a forwarder to the new location.
When copying a weak object, the scavenger leaves a corpse.
The weak list is a linked list of the corpses.
Each corpse contains the address to the new location of the weak object and a reference to the next corpse in the weak list.

A corpse is a normal forwarder.
The memory format guarantees that every single object has at least one slot.
This slot is used to hold the forwarded reference.
It also guarantees that there is a 64 bits header.
As there is no waranty that the object has more than one slot, the address of the next corpse in the weak list has to be encoded in the object's header.

All the addresses of the corpses, including the one in the weakList instance variable \(in the SpurGenerationScavenger\), are offsets from the start of the New space.
This offset is expressed in allocationUnits size \(8 bytes\).
The offset starts in 1 to detect if the list has ended, a zero value for the offset is not a valid corpse marking the end of the link list.

This offset is encoded in 27 bits of the object header \(22 bits of the hash and 5 bits of the format\).
The remaining part of the header's informations are used by the scavenger.
For example, the class index is used to know which objects are forwarders.
With this trick, we can address objects inside the new space of a maximum size of one gigabyte.
The current calculation of the VM to find the next corpse from the current one is the following:
```
allocationUnit = 8.
offset = (formatBits + (hashbit << 5)) * allocationUnit.
nextCorpseAddress = offset - 1 + newSpaceStart.
```


The corpses are added to the list in the beginning of the list. The head of the list is the last added corpse.
This allows the scavenger not to traverse the whole list to add a new corpse.

The weak list is built in the scavenging and it is discarded after it ends. 



## Ephemerons 


Let's start with an example:
An open File object uses a operating system handler.
When opening a File object, a file is opened in the operating system.
This is a limited and should be given back to the system as soon as it is not used anymore.
If the File object is collected without being closed, the system won't be able to close the file.
Therefore we need an additional mecanism to detect when a file object is collected.
One such mechanism is called Ephemerons.
