## The Spur Memory Manager Overview
  allocateSlots: numberOfSlots
  format: instanceSpecification
  classIndex: classIndex
memoryManager newSpaceLimit.

SpurMemoryManager >> isYoung: oop
	<api>
	"Answer if oop is young."
	^(self isNonImmediate: oop)
	  and: [self oop: oop isLessThan: newSpaceLimit]