## Architecture



```
Pharo --slang --> C

Interpreter 		JIT Compiler

```

## Interpreter

Each bytecode is a method
Bytecode table
	#(
		byte1
		byte2)
	
```	
while (true){
	b =fetch (byte)
	table[b]()
}
```	

Threaded interpreter with goto between branches


## JIT
#(
	genByte1
	genByte2
	...
)
250 different bytecode

genByte1 
- generates IR abstract interpretation of the byte code 
- code generation
- generated code is a code cache 
- there is a reference counting algorithm of the code cache

Primitives


### Stack 

The native stack is split into stack.
Each thread on pages on the native stack.

Each of the thread 

### Spur the memory scavenger

- scavenger old
	mark and compact 
- copy collector for young object 
	future / past / nursery (eden), .... old 
	
### memory model

- header
- immediate
- opaque (bytearray)


### Frame to context


### Frames

Frames are not the same in interpreter or in jit.
The PC is different.

### Calling conventions






### Cog vs. CogIt

- Cog VM 
- CogIt compiler inside the VM


	
	
