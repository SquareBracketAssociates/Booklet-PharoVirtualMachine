## Basics on Execution: The interpreter point of view
	"Answer the width of the receiver."
	^ corner x - origin x
>>> #[1 126 0 126 97 92]
	25 <01> pushRcvr: 1 
	26 <7E> send: x 
	27 <00> pushRcvr: 0 
	28 <7E> send: x 
	29 <61> send: - 
	30 <5C> returnTop
	corner := origin + aPoint
>>> #[0 64 96 201 88]
	"Return an adjustment of the receiver that fits within aRectangle by reducing its size, not by changing its origin."

	^ origin corner: (corner min: aRectangle bottomRight)
>>> #[0 1 64 128 145 146 92]
>>> #(#+ #- #< #> #'<=' #'>=' #= #'~=' #* #/ #'\\' #@ #bitShift: #'//' #bitAnd: #bitOr: #at: #at:put: #size #next #nextPut: #atEnd #'==' nil "class" 
#'~~' #value #value: #do: #new #new: #x #y)
	^self right @ self center y
	"Answer the position of the receiver's right vertical line."
	^ corner x 
	"Answer whether aPoint is within the receiver."
	^origin <= aPoint and: [aPoint < corner]
	<primitive: 1>
	^super + addend