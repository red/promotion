Have you ever wonder about the mess that software programming stacks have become? Exploding complexity and bloated solutions are everywhere, yet few architects and programmers are really aware of the induced costs, both for creating and maintaining modern software. Red is here to fight back the complexity, that is the main reason for his creation. Yes, it is possible to still have *simple* tools and simple solutions in the modern software world. It was a well-kept secret (unfortunately), but such solution already exists since 1997 in the form of the Rebol programming language. Red is a descendant of the Rebol-language famility tree, and tries to extend it far beyond Rebol's original scope of usage. Red is a very ambitious project and is meant to be the first full-stack programming language. Here are the main features and design goals:

* Paradigm-neutral, functional/imperative/symbolic by default
* Prototype-based object support
* Homoiconic (Red is its own meta-language)
* Both statically and JIT-compiled to native code
* Concurrency and parallelism strong support (actors, parallel collections)
* Low-level system programming abilities through the built-in Red/System DSL
* High-level scripting and REPL console support
* Highly embeddable (think Lua, or better)
* Low memory footprint, garbage collected
* Low disk footprint (< 1MB)
* Single command-line executable
* Zero install, zero config
* Standalone cross-platform toolchain
* No dependencies other than what the OS provides

Red, like Rebol, relies on a different approach to programming than what the current mainstream languages are proposing. This is what Douglas Crockford, the inventor of JSON, says about Rebol:

"Rebol's a more modern language, but with some very similar ideas to Lisp, in that it's all built upon a representation of data which is then executable as programs. But it's a much richer thing syntactically. Rebol is a brilliant language, and it's a shame it's not more popular, because it deserves to be." (1)

Red and Rebol are designed to maximize expressiveness while keeping the source code very readable, quite close to a natural language in fact. A research study in 2013 that tried to rank programming languages by expressiveness, puts Rebol at third place, behind Augeas and Puppet, two DSL, showing Rebol effectively as the most expressive general-purpose language around. Both Red and Rebol are pragmatically designed, but nothing is arbitrary or "done just to be different". There is a reason behind every syntactic or semantic feature.

How does Red and Rebol achieve such level of efficiency while keeping things "simple"? One of the core reasons is that they are both a data format *and* a programming language. This feature comes from the Lisp inheritage, especially from the s-expressions concept. (3) Lisp established the meta-programming model that Red and Rebol rely on. They are all homoiconic languages, all code is represented as data. They store values in blocks (using square brackets notation: [...]) which are simple lists, as Lisp does, but they don't require parens around function calls, so the code looks more like a natural language. Functions use the prefix notation by default, however, Red and Rebol support the infix notation for math and boolean operators, for the sake of practical reading. Using this pure meta-programming approach, features like reflection, hot-patching or even live-coding come for free, no new API to learn, no special library required, you just use the basic functions for data manipulation. Here is an example using Red's REPL:

	$ red
	red>> code: ["hello"]
	== ["hello"]
	
	red>> insert code 'print
	== ["hello"]
	
	red>> code
	== [print "hello"]
	
	red>> do code
	hello
	
	red>> code/2: 123
	== 123
	
	red>> code
	== [print 123]
	
	red>> append code [+ 1]
	== [print 123 + 1]
	
	red>> do code
	124
	
	red>> length? code
	== 4
	
	red>> type? code/1
	== word!
	
	red>> foreach value code [probe type? value]
	word!
	integer!
	word!
	integer!

As you can see, the basic building elements of the language are:

* values: they represent numbers, times, strings, filenames, URL and much more: 123 45.6 10:20:30.456 192.168.1.0 "hello" http://red-lang.org

* words: they are symbols that can be used as "variables" (but not always): print block? string! map-each ?? + =

* blocks: combine multiple values, words and blocks in any order: [1 2 3] [123 "abc" print 10:20]

These are the building blocks used to represent data, build expressions, define functions, make objects, and create more complex datatypes. Symbols are first-class values, they provide a much more natural and efficient alternative to strings for a lot of use-cases, especially when lookups are involved, as symbols can be compared with each other in O(1), while strings require O(n). Symbols are case-insensitive, so print = Print = PRINT. Human language works this way too.

The source code of any Rebol or Red program usually comes as a UTF-8 input string, that gets LOAD-ed. LOAD is a native core function that transforms any string to an in-memory binary format contained in a block. Blocks are storing values into 128-bit adjacent cells. Values are mainly divided into scalar values (they are of fixed size) or series values (user can add/remove data from them). Numbers, dates, tuples, pairs are examples of scalar while pairs, blocks, strings, URLs, paths, files, tags are examples of series. Scalar values can usually fit into a memory cell, while series require an additional memory buffer. 

So, blocks are the main allocation unit that the memory manager reserves and collects with a GC. Rebol uses a classic stop-the-world mark & sweep GC. Red relies on a stop-the-thread generational compacting GC (not fully implemented yet), with alternance of partial and full passes. Red will extend its GC to allow incremental collections in the future, to enable real-time applications like 60-fps arcade games.

Once a source code is loaded in memory as a block of values, it is just pure data. By default, when passing an input source code to Red or Rebol binaries, it will be evaluated. So the loaded block is interpreted in the case of Rebol, or compiled to native code in the case of Red in order to evaluate it. However, the way a block of values is interpreted is dependent on the context. It can be interpreted:

* In a functional way: arguments evaluated by a function
* As a special grammar: what we call a dialect (DSL)
* Via Parse dialect statements: matching, backtrack, and production
* With ad-hoc code: however you want to do it

The default (unpure) functional language (that we commonly refer to as "code") is an expression-oriented language with very simple semantic rules:

* No keywords
* Fixed arity functions with optional extensions called "refinements"
* Optionally typed or multi-typed function arguments and return value
* Expressions are evaluated from left to right
* Infix operators take precedence over prefix function calls

So, the functional layer on top of the language can easily be replaced by any other custom evaluator (for example, writing a Prolog-like interpreter is very simple). This makes Red and Rebol extremely maleable, they can easily be adapted to whatever need you can have. This flexibility is the foundation that allows DSL to become a natural way to solve some computing tasks, by providing a domain-specific micro-language optimized for a given task. Red and Rebol leverage that power by using embedded DSL extensively in the core language and standard library:

* VID: View Interface Dialect is a DSL for building GUIs
* Draw: 2D drawing DSL
* Parse DSL: A BNF-like grammar parsing DSL
* Security dialect: A DSL for controlling the security sandboxing features at runtime.
* Functions specification: functions prototype are specified using a DSL too.

What makes embedded DSL easy and cheap to create is:

* The Parse DSL: a powerful TDPL (4) with recursivity and backtracking support
* The ability to inline Red or Rebol code directly into the parsing rules
* A rich set of datatypes: dozens of common data literal notations are already recognized, so no need to create special rules to parse them.

A few GUI DSL examples in Rebol2:

	Open a window with a red button of size 100x100:
  	>> view layout [button "Hello" red 100x100]
  	
  	Show a button that does an action:
  	>> view layout [button "Hi" [print "Hello!"]]

	Show a button that prints a field's content:
  	>> view layout [
  			in: field 200
  			button "Print" [print in/text]
  	   ]

A Parse DSL example in Red (5):

	Extract a number from a string:
	red>> digit: charset "0123456789"
	red>> parse "hello 888 world" [
		some [copy n some digit | skip]
	]
	red>> n
	== "888"

	
What does Red bring compared to Rebol?
--------------------------------------

Red arised from the need to address several shortcomings of Rebol:

* Limited performances (close to Ruby/PHP)
* No low-level programming abilities
* Not multi-cores aware, limited concurrency support
* No support for mobile operating systems (limited Android support in 2013)
* Not open source (until December 2012 and only Rebol v3)

Red solved most of these issue using an innovative design choice: embed a low-level programming language that compiles directly to native code: Red/System. It has a Red syntax (still blocks of values), but C-like, low-level semantics. It is statically-typed and has a limited number of datatypes: integer!, float!, float32!, byte!, logic!, pointer!, struct! and function!. Pointer arithmetic is supported, giving it roughly the same power as C. The standard library is very limited also, so pure Red/System compiled code is extremely small, the following program:

	Red/System [
		Title: "Hello World app"
	]
	
	print "Hello World!"
	
typically compiles to a less than 10KB executable.

Red programs are compiled to Red/System code and linked with the Red standard library, which is written in Red/System (mainly) and in Red itself. The current uncompressed size of that library is around 200KB. We are aiming at 500KB for the 1.0 version.

Red and Red/System are closely integrated. It is possible to call or inline Red/System code in Red using different ways. The most useful one is the "routine" function type, that allows to define a Red function where the body is pure Red/System code. Red will do the boxing/unboxing of passed arguments and return value for you. With the help of Red/System, Red can address *any* abstraction level from hardware to DSL, making it effectively the first full-stack programming language. 

So, Red relies on Red/System toolchain to generate executables. The currently supported targets for the compiler and linker are:

* CPU: IA-32, ARMv5 and limited AVR-8 support (Atmel328P)
* File formats: PE, ELF, Mach-o, .dll, .so, APK, Windows driver

Other planned formats to be supported:

* Backends: IA-64, ARMv7, asm.js, JVM, CLR, Dalvik, LLVM
* File formats: .lib, .a, .ipa, drivers formats, war (web archive)

In addition to that, Red relies on bridging technologies for interfacing with VM like Java, especially for Android support. So, a Red/Java bridge is already available using JNI to remote control the Java and Android API from Red programs and get back events. A Red/Obj-c bridge is also planned later this year to provide access to iOS and Cocoa APIs.

Given the wide range of possible combinations of these backends and file formats, Red uses a short configuration file for defining the options for each target. This allows a very simple cross-compilation support, for example:

   Generate a Windows executable from Linux or Mac:
   $ red -t Windows hello.red

   Generate a RaspberryPi executable from Windows:
   $ red -t Linux-ARM hello.red
   
Here are how these targets are defined;

	Windows [
		OS:			'Windows
		format: 	'PE
		target:		'IA-32
		type:		'exe
		sub-system: 'GUI
	]
	
	Linux-ARM [
		OS:			'Linux
		format:		'ELF
		target:		'ARM
		type:		'exe
		base-address: 32768			; 8000h
		dynamic-linker: "/lib/ld-linux.so.3"
	]


Red is still under heavy work and not completed yet. It is currently bootstrapped using Rebol2 for the toolchain part (Red and Red/System compilers + linker). As we need to support JIT-compilation of Red/System code generated dynamically, Red needs to be self-hosted, so all the toolchain needs to be rewritten in Red after v1.0. The new toolchain will enable the real full-featured Red and bring the optimizing layers for both compilers, generating even smaller and faster code. Currently, the performances of Red/System were measured around 4-6 times slower than C, which is already good for non-optimized native code.

The performances will also increase with the upcoming support for concurrency. Red will provide a fully async I/O interface later this year, coupled with a M:N threading model (M light threads dispatched over N OS threads). The current plan for the higher-level abstraction is to use the Actor model, in order to allow simple, but efficient shared state synchronization. From the user perspective, the Actor will just be a first class datatype that will be used mostly like any other object, with minor or no syntactic overhead. However, given the huge popularity of the goroutine model from Go language, we are still reviewing the options to decide on the best model to use for Red.

Red is an open source project under BSD license. As the language author, I work on it full-time since 3 years now. The project funding is done by direct donations from users and followers. The amount of support I have received since the first announce in 2011 is incredible. Programmers are very excited by what Red can offer, especially Rebol users that know what Rebol is already capable of. Red is like a dream tool coming true to most of them, including myself. Red is bringing back the fun again to programming, by keeping complexity away and letting the programmer feel in control again, like we used to be in the good old 8-bit times.

During this year of the Horse, Red will be reaching 1.0 during fall of this year and shortly after a company will be formed to support Red commercially and help it grow further. As Red will start with a strong Android support and is particulary suited to newcomers to programming, we want it to be where the biggest Android market is: China. The number of potential programers in China is huge, one of the most talented Red contributor is a Chinese programmer from Shanghai, Xie Qingtian! So Red is moving to China this year, to be based in Beijing for starting. Red is the color of China afterall. ;-)

You can download a suitable Red binary from http://www.red-lang.org/p/download.html. Red comes as a single half-megabyte executable. It contains the whole toolchain, both Red and Red/System compilers, an interpreter with an optional console and the whole standard library with currently around 30 datatypes. Alternatively, you can also play with the older, but complete Rebol2 interpreter, especially to test the GUI DSL, you can download one of the interpreters from http://www.rebol.com/download-view.html.


(1) "The making of JSON by Douglas Crockford" (2009) http://www.dzone.com/links/the_making_of_json_by_douglas_crockford_an_influe.html

(2) "Programming languages ranked by expressiveness" (2013) http://redmonk.com/dberkholz/2013/03/25/programming-languages-ranked-by-expressiveness/

(3) S-expression, Wikipedia http://en.wikipedia.org/wiki/S-expression

(4) TDPL, Wikipedia http://en.wikipedia.org/wiki/Top-down_parsing_language

(5) More Parse DSL examples at http://www.red-lang.org/2013/11/041-introducing-parse.html
