## JIT Vocabulary
		args
sp->	ret pc.
			arg0
			...
			argN
			caller's saved ip/this stackPage (for a base frame)
	fp->	saved fp
			method
			context (initialized to nil)
			frame flags (interpreter only)
			saved method ip (initialized to 0; interpreter only)
			receiver
			first temp
			...
	sp->	Nth temp