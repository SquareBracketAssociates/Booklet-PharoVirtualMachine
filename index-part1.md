## Preamble

A virtual machine is a complex system. It is often based on several abstractions, such as an interpreter and a native code compiler. In addition, memory management introduces extra complexity as it represents an important concern for modern managed runtime for languages such as Pharo. The Pharo virtual machine is based on a bytecode interpreter and a baseline JIT compiled. It has two different garbage collectors.  Most of the design of the Pharo Virtual Machine has been made by E. Miranda.

This book is an effort to document and structure the knowledge of the internals of the Pharo VM.
Our objective is to make the design of the Pharo VM more accessible. 
Doing so it will help increase the amount of people who understand and improve it.
On this basis, we are working on improving its design and creating a new generation virtual machine. 

Caveats. it may happen that we wrote something unclear. If you spot something you think can be improved, please let us know.

Readers may also be interested in other booklets. The booklet on concurrent programming in Pharo also describes some VM primitives and provides some specific information. A booklet on the Pharo compiler is also a work in progress.


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
<!inputFile|path=Part1-InterpreterAndBytecode/5-DeeperBytecode/methodsbytecode.md!>

%# Optimizations

%<!inputFile|path=Part1-InterpreterAndBytecode/6-InterpreterOptimizations/interpreteroptimizations.md!>