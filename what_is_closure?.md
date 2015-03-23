Neat explanation about closure
===

###What is closure?

Closure is nothing more than a function that has access to private variable.

```
function Person(name) {
	var name = name;
}
```

The variable will be inaccessible after Person returned. So to keep a reference to *name*, we could use a closure:

```
function Person(name) {
	var _name = name;
	
	this.getName = function() {
		return _name;
	}
}
```

So when Person returned, we still have a reference to variable _name by calling getName function.

--- 
