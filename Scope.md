Scope
===

programme state: the ability to store values and pull values out of variables.

scope: a set of rules for storing var and for finding those var later.

JS is in fact a complied language, but its not complied in advance. The compilation happens in many cases microseconds before the code is executed.

Any snippet of js has to be compiled before it's executed.

###var a = 2;

2 actions for assignment:

1. compiler declares a var in current scope
2. when executing, engine looks up the var in scope and assigns if found

* If an RHS look-up fails to ever find a variable, this results ReferenceError
* If a variable is found for an RHS look-up, but you try to do sth with its value that is impossible, this will result TypeError

