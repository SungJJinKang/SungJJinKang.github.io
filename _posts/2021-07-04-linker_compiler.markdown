---
layout: post
title:  "프리프로세서, 컴파일러, 링커가 하는 일 ( 번역 )"
date:   2021-07-04
categories: C++
---

The compilation of a C++ program involves three steps:

Preprocessing: the preprocessor takes a C++ source code file and deals with the #includes, #defines and other preprocessor directives. The output of this step is a "pure" C++ file without pre-processor directives.             

Compilation: the compiler takes the pre-processor's output and produces an object file from it.            

Linking: the linker takes the object files produced by the compiler and produces either a library or an executable file.             

**Preprocessing**             
The preprocessor handles the preprocessor directives, like #include and #define. It is agnostic of the syntax of C++, which is why it must be used with care.             

It works on one C++ source file at a time by replacing #include directives with the content of the respective files (which is usually just declarations), doing replacement of macros (#define), and selecting different portions of text depending of #if, #ifdef and #ifndef directives.          

The preprocessor works on a stream of preprocessing tokens. Macro substitution is defined as replacing tokens with other tokens (the operator ## enables merging two tokens when it makes sense).            

After all this, the preprocessor produces a single output that is a stream of tokens resulting from the transformations described above. It also adds some special markers that tell the compiler where each line came from so that it can use those to produce sensible error messages.         

Some errors can be produced at this stage with clever use of the #if and #error directives.        

**Compilation**           
The compilation step is performed on each output of the preprocessor. The compiler parses the pure C++ source code (now without any preprocessor directives) and converts it into assembly code. Then invokes underlying back-end(assembler in toolchain) that assembles that code into machine code producing actual binary file in some format(ELF, COFF, a.out, ...). This object file contains the compiled code (in binary form) of the symbols defined in the input. Symbols in object files are referred to by name.      

Object files can refer to symbols that are not defined. This is the case when you use a declaration, and don't provide a definition for it. The compiler doesn't mind this, and will happily produce the object file as long as the source code is well-formed.           

Compilers usually let you stop compilation at this point. This is very useful because with it you can compile each source code file separately. The advantage this provides is that you don't need to recompile everything if you only change a single file.              

The produced object files can be put in special archives called static libraries, for easier reusing later on.          

It's at this stage that "regular" compiler errors, like syntax errors or failed overload resolution errors, are reported.              

**Linking**             
The linker is what produces the final compilation output from the object files the compiler produced. This output can be either a shared (or dynamic) library (and while the name is similar, they haven't got much in common with static libraries mentioned earlier) or an executable.             

It links all the object files by replacing the references to undefined symbols with the correct addresses. Each of these symbols can be defined in other object files or in libraries. If they are defined in libraries other than the standard library, you need to tell the linker about them.       

At this stage the most common errors are missing definitions or duplicate definitions. The former means that either the definitions don't exist (i.e. they are not written), or that the object files or libraries where they reside were not given to the linker. The latter is obvious: the same symbol was defined in two different object files or libraries.           



--------------------------------------------           


The exact relationship varies somewhat. To start with, I'll consider (nearly) the simplest possible model, used by something like MS-DOS, where an executable will always be statically linked. For the sake of example, let's consider the canonical "Hello, World!" program, which we'll assume is written in C.            

**Compiler**            
The compiler will compile this into a couple of pieces. It'll take the string literal "Hello, World!", and put it into one section marked as constant data, and it'll synthesize a name for that particular string (e.g., "$L1"). It'll compile the call to printf into another section that's marked as code. In this case, it'll say the name is main (or, frequently, _main). It'll also have something to say this chunk of code is N bytes long, and (importantly) contains a call to printf at offset M in that code.              

**Linker**                
Once the compiler is done producing that, the linker will run. It's normally considered part of the development tool chain (though there are exceptions -- MS-DOS used to include a linker, though it was rarely if ever used). Although it's not normally externally visible, it will normally be passed some command-line arguments, one specifying an object file containing some startup code, and another specifying whatever file contains the C standard library.           
 
The linker will then look at the object file containing the startup code and find that it is, say, 1112 bytes long, and has a call to _main at offset 784 in that.          

Based on that, it'll start to build a symbol table. It'll have one entry saying ".startup" (or whatever name) is 1112 bytes long, and (so far) nothing refers to that name. It'll have another entry saying "printf" is a current unknown length, but it's referred to from ".startup+784".             

It'll then scan through the specified library (or libraries) to try to find definitions of the names in the symbol table that aren't currently defined -- in this case printf. It'll find the object file for printf saying that it's 4087 bytes long, and has references to other routines to do things like converting an int to a string, as well as things like putchar (or maybe fputc) to write the resulting string to the output file.            

The linker will re-scan to try to find definitions of those symbols, recursively, until it reaches one of two conclusions: it's either found definitions of all the symbols, or else there's a symbol for which it can't find a definition.            

If it's found a reference but no definition, it'll stop and give an error message typically saying something about an "undefined external XXX", and it'll be up to you to figure out what other library or object file you need to link.           

If it finds definitions of all the symbols, it moves on to the next phase: it walks through the list of places that refer to each symbol, and it'll fill in the address where that symbol got put into memory, so (for example) where the startup code calls main, it'll fill in the address 1112 as the address of main. Once it's done all that, it'll write all the code and data out to an executable file.         

There are a few other minor details that probably bear mentioning: it'll typically keep the code and data separate, and after each is complete, it'll put them all together at (more or less) consecutive addresses (e.g., all the pieces of code, then all the pieces of data). There will typically also be some rules about how to combine definitions for section/segments -- for example, if different object files all have code segments, it'll just arrange the pieces of code one after another. If two or more identical string literals (or other constants) are defined, it'll typically merge those together so all of them refer to the same place. There are also a few rules for what to do when/if it finds duplicate definitions of the same symbol. In a typical case, this will simply be an error. In a few cases, it'll have things like "weak external" symbols, that basically say: "I"m providing a definition of this symbol, but if somebody else also defines it, don't consider it an error -- just use that definition instead of this one.              

Once it has entries for all the symbols, the linker has to arrange the "pieces" and assign addresses to them. The order in which it arranges the pieces will vary somewhat -- it'll typically have some flags about the types of different pieces, so (for example) all the constant data ends up next to each other, all the pieces of code next to each other and so on. In our simple MS-DOS-like system, most of this won't matter a whole lot though.              

**Loader**              
That brings us to the next phase: the loader. the loader is typically part of the operating system, which loads the executable. In ancient versions (e.g., CP/M, MS_DOS .com files, the loader just read data from an executable file into memory, then started executing at some address. Slightly more recent loaders (e.g., for MS-DOS .exe files) will start out (more or less) the same way: read a file into memory. In this case, however, based on the entries put there by the linker, it'll "fix up" any absolute references in the executable to refer to the correct address. In the example above, our startup code referred to main at address 1112, but the executable is being loaded at a base address of (say) 4000. In this case, the loader will fix that address up to refer to 5112. In this simple of a system, however, the loader is still a pretty simple piece of code -- basically just walking through the list of relocations, and adding the base address to each.                

Now let's consider a bit more modern OS that supports something like shared object files or DLLs. This basically shifts some of the work from the linker to the loader. In particular, for a symbol that's defined in a .so/DLL, the linker will not attempt to assign an address itself.               

Instead it'll create a symbol table entry that basically says "defined in .so/DLL file XXX". When the linker writes the executable, most of these symbol table entries will basically just get copied to the executable, saying "symbol XXX is defined in file YYY". It's then up to the loader to find file YYY, and the address of symbol XXX in that file, and fill in the correct address wherever it's used in the executable. Much like in the linker this will be recursive, so DLL A may refer to symbols in DLL B, which may refer to DLL C, and so on. Although the chain from executable to all the definitions may be long, the basic idea of the process is fairly simple -- scan through the list of external references, and find a definition for each. Also note that in most cases, it will be able to share a single executable across many processes, so the OS will normally have a list of loaded modules, and when/if it gets to a module that's already loaded, it'll just fill in entries for that, and be done rather than re-loading it from the beginning.                 

Again, there are some miscellaneous bits and pieces to consider. For example, the sharing will normally only happen on a section-by-section basis, not file-by-file. If a file has some code and some (non-constant) data, for example, all processes will share the same code sections, but each will get its own copy of the data.               


프리프로세서, 컴파일러, 링커 각각의 작동 원리는 차후에 글을 쓸 예정이다.            

references : [https://stackoverflow.com/q/25826277](https://stackoverflow.com/q/25826277), [https://stackoverflow.com/q/6264249](https://stackoverflow.com/q/6264249), [https://softwareengineering.stackexchange.com/q/103673](https://softwareengineering.stackexchange.com/q/103673), [https://stackoverflow.com/q/3322911](https://stackoverflow.com/q/3322911)