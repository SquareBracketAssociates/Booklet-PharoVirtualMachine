## Preamble


Most of the design of the Pharo Virtual Machine has been made by E. Miranda and we are very grateful for this.
On this basis, we are working on improving its design and create a new generation virtual machine. 
This book is the result of an effort to understand, document, and structure the knowledge of the internals of the Pharo VM.
Our objective is to increase the amount of people who understand and improve it in the future.
This work reused some previously existing materials such as blog posts of C. Béra.

Now, it may happen that we wrote something wrong. If you spot a mistake, please let us know.

Readers may also be interested in other booklets.
The booklet on concurrent programming in Pharo also describes some VM primitives and provide some specific information.
A booklet on the Pharo compiler is also a work in progress.

_Acknowledgements._ This work is supported by Ministry of Higher Education and Research, Nord-Pas de Calais Regional Council, CPER Nord-Pas de Calais/FEDER DATA Advanced data science and technologies 2015-2020.
The work is supported by I-Site ERC-Generator Multi project 2018-2022. We gratefully acknowledge the financial support of the Métropole Européenne de Lille.
This work is also supported by the Action Exploratoire Alamvic led by G. Polito and S. Ducasse.

<!inputFile|path=Part0-Preamble/0-RuntimeSystemOverview/runtime.md!>

# The Basics

<!inputFile|path=Part0-Preamble/1-ObjectStructure/objectStructure.md!>
<!inputFile|path=Part1-InterpreterAndBytecode/2-MethodsAndBytecode/methodsbytecode.md!>

# Bytecode Execution

<!inputFile|path=Part1-InterpreterAndBytecode/3-SemanticsByExample/basicsOnExecution.md!>
<!inputFile|path=Part1-InterpreterAndBytecode/4-Interpreter/theInterpreter.md!>

# Optimizations

<!inputFile|path=Part1-InterpreterAndBytecode/5-DeeperBytecode/methodsbytecode.md!>
<!inputFile|path=Part1-InterpreterAndBytecode/6-InterpreterOptimizations/interpreteroptimizations.md!>

# The Memory Manager

<!inputFile|path=Part3-MemoryManagement/GarbageCollector/memoryStructure.md!>
<!inputFile|path=Part3-MemoryManagement/GarbageCollector/newSpace.md!>
<!inputFile|path=Part3-MemoryManagement/GarbageCollector/oldSpace.md!>
<!inputFile|path=Part3-MemoryManagement/GarbageCollector/freeList.md!>
<!inputFile|path=Part3-MemoryManagement/GarbageCollector/ephemerons.md!>

# JIT Compilation

%<!inputFile|path=Part2-JIT/CallingConventions/CallingConventions.md!>

## Slang
@cha:Slang
Just here so that we can refer to it

# Exercises

<!inputFile|path=Part4-Tutorials/HandonsStatic/handonsstatic.md!>