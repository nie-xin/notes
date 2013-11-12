JS的怪癖
===
忽略var声明等同于声明全局变量。

声明变量时，变量会被升起至functional scope的开始处，并被赋值undefined，直到变量被赋值为止。因此无论在函数的任何位置声明的变量在整个函数中都是可见的，虽然其值有可能是undefined。

	function prison () {
		console.log(prisoner);	//prisoner已可见，但值为undefined
		var prisoner = 'Now I am defined!';	//此刻开始prisoner被赋值
		console.log(prisoner); //此时prisoner已经是Now I am defined!
	}
	prison();
	
因为变量声明总是会被提升至最高处，所以一个最佳经验就是将变量声明放在函数的开始处，并只采用一个var声明。