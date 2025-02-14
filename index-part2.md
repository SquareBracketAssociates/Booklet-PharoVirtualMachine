## Preamble


Most of the design of the Pharo Virtual Machine has been made by E. Miranda and we are very grateful for this.
On this basis, we are working on improving its design and create new generation virtual machine. 
This book is the result of an effort to understand, document, and structure the knowledge of the internals of the Pharo VM.
Our objective is to increase the amount of people who understand and improve it in the future.
This work re-used some previously existing materials such as blog posts of C. Béra.

Now, it may happen that we wrote something wrong. If you spot something wrong, please let us know.

Readers may also be interested in other booklets.
The booklet on concurrent programming in Pharo also describes some VM primitives and provide some specific information.
A booklet on the Pharo compiler is also a work in progress.

_Acknowledgements._ This work is supported by Ministry of Higher Education and Research, Nord-Pas de Calais Regional Council, CPER Nord-Pas de Calais/FEDER DATA Advanced data science and technologies 2015-2020.
The work is supported by I-Site ERC-Generator Multi project 2018-2022. We gratefully acknowledge the financial support of the Métropole Européenne de Lille.
This work is also supported by the Action Exploratoire Alamvic led by G. Polito and S. Ducasse.

<!inputFile|path=Part0-Preamble/0-RuntimeSystemOverview/runtime.md!>

# JIT Compilation

<!inputFile|path=Part2-JIT/1-CogitOverview/chapter.md!>
<!inputFile|path=Part2-JIT/2-CodeLayout/CodeLayout.md!>
<!inputFile|path=Part2-JIT/3-Trampolines/chapter.md!>
<!inputFile|path=Part2-JIT/4-PICs/chapter.md!>
<!inputFile|path=Part2-JIT/5-AbstractInterpretation/chapter.md!>
<!inputFile|path=Part2-JIT/5-CodeCache/chapter.md!>

<!inputFile|path=Part4-Tutorials/HandonsStatic/handonsstatic.md!>