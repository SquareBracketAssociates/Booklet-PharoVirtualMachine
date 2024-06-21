### Contexts


Push, store, and jump bytecodes require only small changes to the state of the interpreter. 
Objects may be moved to or from the stack, and the instruction pointer is always changed; but most of the state remains the same. Send and return bytecodes require much larger changes to the interpreter's state. 

When a new message is sent \(for example message `x` or `-` above\), all five parts of the interpreter's state may have to be changed to execute a different CompiledMethod in response to this new message. The interpreter's old state must be remembered because the bytecodes after the send must be executed after the value of the message is returned. 

The interpreter saves its state in objects called _contexts_. There will be many contexts in the system at any one time. The context that represents the current state of the interpreter is called the _active_ context. When a send bytecode in the active context's CompiledMethod requires a new compiled method to be executed, the active context becomes _suspended_ and a new context is created and made active. The suspended context retains the state associated with the original compiled method until that context becomes active again. A context must remember the context that it suspended so that the suspended context can be resumed when a result is returned. The suspended context is called the new context's _sender_. 



We extend the form used to show the interpreter's state. The active context will be indicated by the word **Active** in its top delimiter. Suspended contexts will say **Suspended**. 
