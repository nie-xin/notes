#读Underscore，学习JavaScript

>为什么选Underscore？除了目前的项目正在用之外，也因为我认为Underscore是一个小而美的库。作为我认真读的第一个JS库源码，小可以保证难度不会过高，美可以保证质量不低。


整个Underscore库被包含在一个IIFE中：

```
(function() {
   //具体实现...
}.call(this))
```

IIFE是JavaScript中常用的实现命名空间的方法。JS中没有原生的命名空间，为保证不污染global全局空间，一般都会将代码用一个IIFE包含起来，形成一个函数空间。也请记得JS中只有函数作用域，没有块作用域。

call(this)在这里给整个类库传递一个全局环境变量，在浏览器下就是window，在服务器上（如node上）就是node的全局环境。

---

###基础设置

```
  var root = this;

  var previousUnderscore = root._;
```
将全局环境变量保存为root，并为避免覆盖已存在的_的变量，将其缓存。

---

```
var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

  var
    push             = ArrayProto.push,
    slice            = ArrayProto.slice,
    concat           = ArrayProto.concat,
    toString         = ObjProto.toString,
    hasOwnProperty   = ObjProto.hasOwnProperty;

  var
    nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind;
```

这里给各种原生的原型和功能函数建立一个缓存快捷访问。JS中每次.访问都会造成一个look-up动作，大量使用的方法来说会造成一定的性能影响。将常用方法缓存是常用的优化技巧。

---

```
var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
  };
```

这是一个有趣的函数，将obj转换为Underscore的实例。该函数保证在任何时候，只有一个_包含一个obj。例如：

```
//new _ instance wrapping an array. Straightforward.
var _withNew = new _([]);

//automatically calls `new` for you and returns that, resulting in same as above
var _withoutNew = _([]);

//just gives you _withoutNew back since it's already a proper _ instance
var _doubleWrapped = _(_withoutNew);

```

该函数提供一种类似OOP风格的接口，比如：

```
_(val).method(…);
// instead of the equal
_.method(val, …);
```

[参考1](http://stackoverflow.com/questions/16378628/what-is-the-underscore-function-for)
[参考2](http://stackoverflow.com/questions/18792622/what-is-this-underscore-js-safe-reference-code-doing?rq=1)

---

```
var createCallback = function(func, context, argCount) {
    if (context === void 0) return func;
    switch (argCount == null ? 3 : argCount) {
      case 1: return function(value) {
        return func.call(context, value);
      };
      case 2: return function(value, other) {
        return func.call(context, value, other);
      };
      case 3: return function(value, index, collection) {
        return func.call(context, value, index, collection);
      };
      case 4: return function(accumulator, value, index, collection) {
        return func.call(context, accumulator, value, index, collection);
      };
    }
    return function() {
      return func.apply(context, arguments);
    };
  };
```

这是个有趣的函数。从注释上来看，该函数的功能就是返回一个更高效的callback给Underscore的其他函数调用。从形式上来看，该函数特别的地方就是明确了参数的个数，并根据参数的个数来确认callback的调用形式。我始终搞不明白为什么这么处理就会高效了？到底有什么必要这么处理呢？

直到我在stackoverflow上看到一个[有趣的讨论](http://stackoverflow.com/questions/12662497/performance-penalty-for-undefined-arguments)后，才明白这么处理的高明之处。讨论的起因是有人提出，在他的实验中，带有可选参数的函数会有出乎意料的性能下降。但如果将可选参数明确定义为undefined，却不会有性能损失。于是下面有人回答，这其实是和JS的优化策略有关。

当参数固定时，优化器可以准确预测到函数的返回值，于是可以将该函数调用优化为内联函数（类似C++中的inline函数，将预测的返回值插入到每个函数调用的位置）。但如果参数的数目不确定，则必须执行函数调用以得到返回值，因此必然造成性能下降了。

[参考1](http://stackoverflow.com/questions/26759412/underscore-js-wrapped-callbacks-efficiency)
[参考2](http://stackoverflow.com/questions/12662497/performance-penalty-for-undefined-arguments)

---

```
 _.iteratee = function(value, context, argCount) {
    if (value == null) return _.identity;
    if (_.isFunction(value)) return createCallback(value, context, argCount);
    if (_.isObject(value)) return _.matches(value);
    return _.property(value);
  };
```

这里涉及到几个函数：

- _.identity: 该函数直接返回传入的参数，类似数学中f(x) = x
- _.isFunction(value): 判断是否是函数
- _.isObject(value): 判断是否是对象
- _.matches(value): 返回一个判断函数，用于判断传递的对象是否包含value
- _.property(value): 返回一个函数，该函数返回传入对象的value属性

这几个函数的详细情况到后面遇到的时候再分析。从已知的情况和官方文档分析，_.iteratee主要用于遍历collection，它根据参数类型返回对应的函数处理collection中的model。

官方文档中有这么一个例子：

```
var stooges = [{name: 'curly', age: 25}, {name: 'moe', age: 21}, {name: 'larry', age: 23}];
_.map(stooges, _.iteratee('age'));
//=> [25, 21, 23];
```

结合源码来看，_.iteratee('age')对应的是最后一种情况，既return _.property(value)，返回对象对象的属性值。所以map的结果就是返回stooges中每个对象的age对应的值。

---

###处理collection

>用过backbone.js的同学对collection应该不陌生，Underscore中的collection更抽象一些，可近似认为是同质对象的集合。

```
_.each = _.forEach = function(obj, iteratee, context) {
    if (obj == null) return obj;
    iteratee = createCallback(iteratee, context);
    var i, length = obj.length;
    if (length === +length) {
      for (i = 0; i < length; i++) {
        iteratee(obj[i], i, obj);
      }
    } else {
      var keys = _.keys(obj);
      for (i = 0, length = keys.length; i < length; i++) {
        iteratee(obj[keys[i]], keys[i], obj);
      }
    }
    return obj;
  };
```

这里有个小技巧：

```
if (length === +length)
```

+length尝试将length转换为数字（如+"123" === 123），如果不能转换则返回NaN。于是该if语句可以判断length是不是数字。

如果length对象是数字，则表示obj是类似array的对象，可以按obj[i]的形式进行遍历。如果length不是数字，则表示obj是一个一般的JS对象，于是取其key的集合，按key进行遍历。在遍历的同时，对每个遍历的对象调用iteratee进行处理。

---

```
 _.map = _.collect = function(obj, iteratee, context) {
    if (obj == null) return [];
    iteratee = _.iteratee(iteratee, context);
    var keys = obj.length !== +obj.length && _.keys(obj),
        length = (keys || obj).length,
        results = Array(length),
        currentKey;
    for (var index = 0; index < length; index++) {
      currentKey = keys ? keys[index] : index;
      results[index] = iteratee(obj[currentKey], currentKey, obj);
    }
    return results;
  };
```

看过each，map也比较容易懂了。创建遍历函数，然后依次对传入的对象或数组的元素调用该函数。不同的是将该函数得到的结果存入另一个数组在遍历结束后返回。

---

```
_.reduce = _.foldl = _.inject = function(obj, iteratee, memo, context) {
    if (obj == null) obj = [];
    iteratee = createCallback(iteratee, context, 4);
    var keys = obj.length !== +obj.length && _.keys(obj),
        length = (keys || obj).length,
        index = 0, currentKey;
    if (arguments.length < 3) {
      if (!length) throw new TypeError(reduceError);
      memo = obj[keys ? keys[index++] : index++];
    }
    for (; index < length; index++) {
      currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
    return memo;
  };
```

和原生reduce功能类似，将数组逐步推导为一个最终结果。结合文档来看：

```
_.reduce(list, iteratee, [memo], [context]) 
```

于是源代码中得这一句就清楚了：

```
    if (arguments.length < 3) {
      if (!length) throw new TypeError(reduceError);
      memo = obj[keys ? keys[index++] : index++];
    }
```

memo是初始值，如果传递的参数不足三个，则忽略初始值，于是将初始值设置为数组的第一个元素并将指针后移一位：

```
memo = obj[keys ? keys[index++] : index++];
```

index++可是先引用后增值哦。继续看下去：

```
    for (; index < length; index++) {
      currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
```

遍历数组，逐次调用迭代函数。注意每次迭代都将memo传入，得到新的结果在存入memo中。于是就将数组归并为一个单一结果了。

---

```
  _.reduceRight = _.foldr = function(obj, iteratee, memo, context) {
    if (obj == null) obj = [];
    iteratee = createCallback(iteratee, context, 4);
    var keys = obj.length !== + obj.length && _.keys(obj),
        index = (keys || obj).length,
        currentKey;
    if (arguments.length < 3) {
      if (!index) throw new TypeError(reduceError);
      memo = obj[keys ? keys[--index] : --index];
    }
    while (index--) {
      currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
    return memo;
  };
```

和reduce思想类似，这次从右边归并。不同的几个地方主要是：

```
    if (arguments.length < 3) {
      if (!index) throw new TypeError(reduceError);
      memo = obj[keys ? keys[--index] : --index];
    }
```

若无传递初值，则初值设置为数组最后一个元素。这里--index是先减再引用，index初始为length，即最后一个元素的再后一个位置，--之后就指向数组最后一个元素了。

这里的迭代过程也有些不同：

```
    while (index--) {
      currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
```

从数组尾部开始迭代，迭代函数的思想跟前面reduce一致。

---

```
 _.find = _.detect = function(obj, predicate, context) {
    var result;
    predicate = _.iteratee(predicate, context);
    _.some(obj, function(value, index, list) {
      if (predicate(value, index, list)) {
        result = value;
        return true;
      }
    });
    return result;
  };
```

逐个检查obj中的对象，返回第一个符合predicate为true的对象。在some的迭代中，如果predicate返回true，就直接return退出迭代，保证只返回第一个符合的对象。

--- 

```
_.filter = _.select = function(obj, predicate, context) {
    var results = [];
    if (obj == null) return results;
    predicate = _.iteratee(predicate, context);
    _.each(obj, function(value, index, list) {
      if (predicate(value, index, list)) results.push(value);
    });
    return results;
  };
```

和原生filter类似，返回一个数组，数组中的元素是符合predicate的obj中的成员。和上面的find结构类似，不同的就是这里用了each，即对每一个obj成员迭代，并在符合predicate后也不返回。

---

```
 _.reject = function(obj, predicate, context) {
    return _.filter(obj, _.negate(_.iteratee(predicate)), context);
  };
```

一个很有用的函数，原生JS没有。功能于filter相反，返回所以不通过predicate的obj成员。结构就很简单了，用一个negate来完成不通过这个概念。我们也来看一下negate：

```
_.negate = function(predicate) {
    return function() {
      return !predicate.apply(this, arguments);
    };
  };
```

就是对predicate的结果取反。

--- 

```
_.every = _.all = function(obj, predicate, context) {
    if (obj == null) return true;
    predicate = _.iteratee(predicate, context);
    var keys = obj.length !== +obj.length && _.keys(obj),
        length = (keys || obj).length,
        index, currentKey;
    for (index = 0; index < length; index++) {
      currentKey = keys ? keys[index] : index;
      if (!predicate(obj[currentKey], currentKey, obj)) return false;
    }
    return true;
  };
```

原生接口也有，如果obj的所有成员都符合predicate，就返回true，任何一个成员不符合就返回false。关键就在这里了：

```
if (!predicate(obj[currentKey], currentKey, obj))
  return false;
```

迭代中只要有一个不符合predicate的成员就立即返回false。

--- 

```
 _.some = _.any = function(obj, predicate, context) {
    if (obj == null) return false;
    predicate = _.iteratee(predicate, context);
    var keys = obj.length !== +obj.length && _.keys(obj),
        length = (keys || obj).length,
        index, currentKey;
    for (index = 0; index < length; index++) {
      currentKey = keys ? keys[index] : index;
      if (predicate(obj[currentKey], currentKey, obj)) return true;
    }
    return false;
  };
```

检测集合中是否存在符合条件的元素，只要有一个符合的元素就返回true。基本结构也和every类似，只是返回条件不一样而已。

---

```
 _.contains = _.include = function(obj, target) {
    if (obj == null) return false;
    if (obj.length !== +obj.length) obj = _.values(obj);
    return _.indexOf(obj, target) >= 0;
  };
```

检测集合是否包含制定的元素。这里引入了连个underscore的函数values和index，我们分别来看看。

```
_.values = function(obj) {
    var keys = _.keys(obj);
    var length = keys.length;
    var values = Array(length);
    for (var i = 0; i < length; i++) {
      values[i] = obj[keys[i]];
    }
    return values;
  };
```

该函数返回一个对象的所有值组成的数组。先用keys取出对象所有键值，然后新建一个同长度的结果数组。最后一次循环对象属性，将其键值存入数组。

```
_.indexOf = function(array, item, isSorted) {
    if (array == null) return -1;
    var i = 0, length = array.length;
    if (isSorted) {
      if (typeof isSorted == 'number') {
        i = isSorted < 0 ? Math.max(0, length + isSorted) : isSorted;
      } else {
        i = _.sortedIndex(array, item);
        return array[i] === item ? i : -1;
      }
    }
    for (; i < length; i++) if (array[i] === item) return i;
    return -1;
  };
```

返回对象在数组中的下标。最后一个参数isSorted可用来指出是否需要对数组排序，如果该参数是数字的话，则指出返回下标的下限。这里又引用了另一个函数sortedIndex，用来返回元素在数组中的排序下标，而数组并未被排序。

```
_.sortedIndex = function(array, obj, iteratee, context) {
    iteratee = _.iteratee(iteratee, context, 1);
    var value = iteratee(obj);
    var low = 0, high = array.length;
    while (low < high) {
      var mid = low + high >>> 1;
      if (iteratee(array[mid]) < value) low = mid + 1; else high = mid;
    }
    return low;
  };
```

这是一个标准的二分查找，唯一比较疑惑的是var mid = low + high >>> 1; 我们来仔细看看>>>的用法。

'>>>'是无符号右移，即右移后，左边用0填充空位。那么这里用这个干嘛呢？我们来看几个例子：

```
8 >>> 1;   //4
3 >>> 1;   //1
```

相当于除2了，于是就得到了二分法的中间位。这里用位移而不直接用除法，是从效率上考虑。位移的速度要比除法速度快很多，是一种高效的求中间位下标的方法。这里复习一下，左移等同于乘2，右移等同于除2。

--- 

```
 _.invoke = function(obj, method) {
    var args = slice.call(arguments, 2);
    var isFunc = _.isFunction(method);
    return _.map(obj, function(value) {
      return (isFunc ? method : value[method]).apply(value, args);
    });
  };
```

在迭代集合的同时，对集合的每个元素应用method。可传入额外的参数以便在调用method的时候传递。内部实际上是一个map，method可以是实际的函数，也可以是函数名。所以这里先用isFunction来判断method的类型，是函数就直接调用，否则就视为对象的方法调用（value[method]）。

---

```
 _.pluck = function(obj, key) {
    return _.map(obj, _.property(key));
  };
```

一个类似map的对对象数组使用的函数，可提取数组中对象的某个特定属性。从本质上看还是一个条件为制定key的map函数。

--- 

```
  _.where = function(obj, attrs) {
    return _.filter(obj, _.matches(attrs));
  };
```

和原生的where类似，不过范围更广泛，可以作用在对象数组上。内部依然是一个filter根据特定的attrs过滤。我们顺便来看看matches吧：

```
_.matches = function(attrs) {
    var pairs = _.pairs(attrs), length = pairs.length;
    return function(obj) {
      if (obj == null) return !length;
      obj = new Object(obj);
      for (var i = 0; i < length; i++) {
        var pair = pairs[i], key = pair[0];
        if (pair[1] !== obj[key] || !(key in obj)) return false;
      }
      return true;
    };
  };
```

返回一个判定方程，判断对象是否包含指定的attrs。这里必须先看懂pairs：

```
  _.pairs = function(obj) {
    var keys = _.keys(obj);
    var length = keys.length;
    var pairs = Array(length);
    for (var i = 0; i < length; i++) {
      pairs[i] = [keys[i], obj[keys[i]]];
    }
    return pairs;
  };
```

将对象转换为一个包含[key, value]形式的数组。还是很容易懂的，就是按对象的key依次遍历，构造指定形式的数组。在回过头看matches，关键的是这里：

```
      for (var i = 0; i < length; i++) {
        var pair = pairs[i], key = pair[0];
        if (pair[1] !== obj[key] || !(key in obj)) return false;
      }
```

遍历所有给定属性集合对，判断对象中是否有这些属性对。if语句判断如果同名属性但值不等，或者根本不存在该属性，就返回false。

---

```
_.findWhere = function(obj, attrs) {
    return _.find(obj, _.matches(attrs));
  };
```

返回第一个包含attrs的对象。内部就是个find，只不过将判断条件设置成了matches以应对对象数组。

---

```
 _.max = function(obj, iteratee, context) {
    var result = -Infinity, lastComputed = -Infinity,
        value, computed;
    if (iteratee == null && obj != null) {
      obj = obj.length === +obj.length ? obj : _.values(obj);
      for (var i = 0, length = obj.length; i < length; i++) {
        value = obj[i];
        if (value > result) {
          result = value;
        }
      }
    } else {
      iteratee = _.iteratee(iteratee, context);
      _.each(obj, function(value, index, list) {
        computed = iteratee(value, index, list);
        if (computed > lastComputed || computed === -Infinity && result === -Infinity) {
          result = value;
          lastComputed = computed;
        }
      });
    }
    return result;
  };
```

返回列表中最大值，可以用于数组，也可用于对象数组(可能需要用iteratee方程判断大小)。第一个if判断是否有iteratee函数，若没有则按数组处理（对象用values抽出值数组），然后就是常规的依次遍历找最大值了。如果有iteratee，则需要依次调用该函数计算当前对象的值，然后和上一次计算值比较，以找出最大值。[Infinity](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Infinity)是JS内置的一个表示极限的值。

--- 

```
 _.extend = function(obj) {
    if (!_.isObject(obj)) return obj;
    var source, prop;
    for (var i = 1, length = arguments.length; i < length; i++) {
      source = arguments[i];
      for (prop in source) {
        if (hasOwnProperty.call(source, prop)) {
            obj[prop] = source[prop];
        }
      }
    }
    return obj;
  };
```

今天跳一下顺序，先来看extend这个函数。extend是一个很常用的函数，在JQuery中也有，功能是合并多个对象，类似数组中得concat函数。在该函数定义中我们只看到一个obj对象，就是目标对象，其他对象的属性将被归并到这个对象中。
关键功能是下面的一个双重for，遍历各个对象和其所有属性。有一点需要注意的地方， obj[prop] = source[prop];表示不加判断地将source的属性复制到obj中，如果值钱obj已经有同名属性，该属性会被后来的属性覆盖。既last in always win。

--- 

接着max继续看下去吧：

```
  _.min = function(obj, iteratee, context) {
    var result = Infinity, lastComputed = Infinity,
        value, computed;
    if (iteratee == null && obj != null) {
      obj = obj.length === +obj.length ? obj : _.values(obj);
      for (var i = 0, length = obj.length; i < length; i++) {
        value = obj[i];
        if (value < result) {
          result = value;
        }
      }
    } else {
      iteratee = _.iteratee(iteratee, context);
      _.each(obj, function(value, index, list) {
        computed = iteratee(value, index, list);
        if (computed < lastComputed || computed === Infinity && result === Infinity) {
          result = value;
          lastComputed = computed;
        }
      });
    }
    return result;
  };
```

和max基本类似，如果有迭代函数iteratee，则使用该迭代函数判断大小。否则就按标准算法依次找最小值。

--- 

```
  _.shuffle = function(obj) {
    var set = obj && obj.length === +obj.length ? obj : _.values(obj);
    var length = set.length;
    var shuffled = Array(length);
    for (var index = 0, rand; index < length; index++) {
      rand = _.random(0, index);
      if (rand !== index) shuffled[index] = shuffled[rand];
      shuffled[rand] = set[index];
    }
    return shuffled;
  };
```

洗牌，意思就是对obj做一个随机全排列。[这里](http://en.wikipedia.org/wiki/Fisher–Yates_shuffle)有详细的介绍，我就只简单说一下了：

设有数字1到N，则

1. 取一个从1到未排序序列长度的随机数字K
2. 取出第K个数，放入已排序集
3. 重复以上步骤，直到所有的数字都放入已排序集

所以函数一开始就分别建立未排序序列和已排序序列：

```
 var set = obj && obj.length === +obj.length ? obj : _.values(obj);
    var length = set.length;
    var shuffled = Array(length);  //已排序序列，初始为长度一致的空数组
```

这里有一个random函数要先介绍一下：

```
_.random(min, max) ： 返回一个介于min和max之间的整数，可视为原生random的一个扩展。
```

下面可以来看全排列的过程了：

```
 for (var index = 0, rand; index < length; index++) {
      rand = _.random(0, index);
      if (rand !== index) shuffled[index] = shuffled[rand];
      shuffled[rand] = set[index];
    }
```

0到index之间的是已排序序列，index指示的时当前需要排列的对象下标，每次迭代就往已排序序列中增加一个元素。为了确定当新加入的index元素位置，需要在已排序序列中随机取一个数，和index元素交换。于是新加入元素也完成随机排序。当下标增加至index-1的时候排列就完成了。

---

```
_.sample = function(obj, n, guard) {
    if (n == null || guard) {
      if (obj.length !== +obj.length) obj = _.values(obj);
      return obj[_.random(obj.length - 1)];
    }
    return _.shuffle(obj).slice(0, Math.max(0, n));
  };
```

从集合中随机抽样，可指定样本数量n。第一个if语言确定没有指定样本数量，于是只随机抽一个样（使用random函数随机选取一个集合内元素）。guard是一个内部参数，稍后遇到的时候再详解。若有样本数量，则用shuffle做一次随机排列，然后取开头的n个元素即可。

--- 

```
 _.sortBy = function(obj, iteratee, context) {
    iteratee = _.iteratee(iteratee, context);
    return _.pluck(_.map(obj, function(value, index, list) {
      return {
        value: value,
        index: index,
        criteria: iteratee(value, index, list)
      };
    }).sort(function(left, right) {
      var a = left.criteria;
      var b = right.criteria;
      if (a !== b) {
        if (a > b || a === void 0) return 1;
        if (a < b || b === void 0) return -1;
      }
      return left.index - right.index;
    }), 'value');
  };
```

这个sortBy和带排序函数的原生sort作用一致，都是根据提供的排序函数对序列进行排序。先用map将序列组合成形式为{value, index, criteria}对象的集合，注意这里的criteria是调用排序函数的结果，用于稍后的排序。对这样的序列调用标准sort，通过比较criteria进行排序。对象集合排序完成后，用pluck抽取出value组成排序序列。

--- 

```
var group = function(behavior) {
    return function(obj, iteratee, context) {
      var result = {};
      iteratee = _.iteratee(iteratee, context);
      _.each(obj, function(value, index) {
        var key = iteratee(value, index, obj);
        behavior(result, value, key);
      });
      return result;
    };
  };

_.groupBy = group(function(result, value, key) {
    if (_.has(result, key)) result[key].push(value); else result[key] = [value];
  });
```

这两个放到一起看，他们实现了将一个集合按给定方法分组的功能。先从group看起吧，这是一个currying结构的函数，先接收一个behavior函数，再返回一个接收三个参数的函数。在返回的函数中，按迭代函数遍历集合，得到key（集合中元素可能是对象）和对应的value。然后调用前面接收的behavior函数来进行分组，分组结果保存在对象result中。

再来看groupBy函数，其实是group的一个包装函数，提供了前面说过的behavoir函数。在这里，behavior其实就是按key分组。先看结果对象中是否已经有对应的key，如果有就表明该分组已存在，将找到的value归入改组。如果key不存在，则说明这是一个不同的类别，于是在结果对象中新建一个分类，分类指向一个同类对象的数组。

---

```
 _.indexBy = group(function(result, value, key) {
    result[key] = value;
  });
```

同是一个使用group的函数。和groupBy不同的地方就在于传递给group的behavior不同。刚才是将得到的结果按key分组，这里是按key分类。

官方文档上有这么一个例子：

```
var stooges = [{name: 'moe', age: 40}, {name: 'larry', age: 50}, {name: 'curly', age: 60}];
_.indexBy(stooges, 'age');
=> {
  "40": {name: 'moe', age: 40},
  "50": {name: 'larry', age: 50},
  "60": {name: 'curly', age: 60}
}
```

可见indexBy按给定的age作为分类的对象。这里再复习一下iteratee这个创建迭代函数的函数：

```
  _.iteratee = function(value, context, argCount) {
    if (value == null) return _.identity;
    if (_.isFunction(value)) return createCallback(value, context, argCount);
    if (_.isObject(value)) return _.matches(value);
    return _.property(value);
  };
```

上列是return的情况，于是迭代函数就是一个访问对象属性的函数。结合上面的代码，分类也就很好理解了。

--- 

```
_.countBy = group(function(result, value, key) {
    if (_.has(result, key)) result[key]++; else result[key] = 1;
  });
```

又是一个依赖group的函数，统计分类结果。当group得出分类值后，如果该分类已经存在，就在该分类统计上加一。如果分类不存在，则建立该分类并初始为一。

---

```
_.sortedIndex = function(array, obj, iteratee, context) {
    iteratee = _.iteratee(iteratee, context, 1);
    var value = iteratee(obj);
    var low = 0, high = array.length;
    while (low < high) {
      var mid = low + high >>> 1;
      if (iteratee(array[mid]) < value) low = mid + 1; else high = mid;
    }
    return low;
  };
```

找出元素在集合中的排序位置。这里还是可以利用iteratee作为排序函数，求出代表值value。然后就是标准的二分排序了，>>>的作用我们前面讲过，就是快速求中间数，速度比直接除要快。

--- 

```
_.toArray = function(obj) {
    if (!obj) return [];
    if (_.isArray(obj)) return slice.call(obj);
    if (obj.length === +obj.length) return _.map(obj, _.identity);
    return _.values(obj);
  };
```

将任何可遍历的对象转化为数组。这个函数在JS中挺有用的，比如arguments这样的特殊数组，就可以用这个函数转换。基本思想还是根据参数的类型做相应的处理，是数组就调用slice做安全转换。如果是对象数组，则返回对象identity（一般就是对象本身）的数组。最后如果是对象，就返回属性值得数组。

--- 

```
 _.size = function(obj) {
    if (obj == null) return 0;
    return obj.length === +obj.length ? obj.length : _.keys(obj).length;
  };
```

返回对象中成员的数量。如果是集合类对象，比如数组，就返回其长度。如果是一般对象，就返回其键值的数量。

--- 

```
_.partition = function(obj, predicate, context) {
    predicate = _.iteratee(predicate, context);
    var pass = [], fail = [];
    _.each(obj, function(value, key, obj) {
      (predicate(value, key, obj) ? pass : fail).push(value);
    });
    return [pass, fail];
  };
```

按predicate分割集合，集合分成满足predicate的部分和不满足的部分。首先自然是将predicate转换成可迭代的函数，然后遍历集合，对各元素依次调用predicate。结果为true的压入pass数组，为fail压入value。最后返回两个结果的数组。

Array相关函数
==

```
 _.first = _.head = _.take = function(array, n, guard) {
    if (array == null) return void 0;
    if (n == null || guard) return array[0];
    if (n < 0) return [];
    return slice.call(array, 0, n);
  };
```

返回数组中头n个元素，若不指定n，如第二个if的情况if (n == null || guard) return array[0];就返回第一个元素。返回头n个元素可以用原生的slice来实现。

--- 

```
_.initial = function(array, n, guard) {
    return slice.call(array, 0, Math.max(0, array.length - (n == null || guard ? 1 : n)));
  };
```

返回除倒数n个元素外的所有元素，不知道n则返回除最后一个元素外的其他元素。还是利用slice实现的，这里稍微注意下slice得结束位置：

```
  Math.max(0, array.length - (n == null || guard ? 1 : n))
```

这里需要排除负数，所以max的下限为0；上限是array.length - n，也就排除了最后的n个元素。若没有设置n，则array.length - 1。

--- 

```
 _.last = function(array, n, guard) {
    if (array == null) return void 0;
    if (n == null || guard) return array[array.length - 1];
    return slice.call(array, Math.max(array.length - n, 0));
  };
```

和first功能相反，返回最后n个元素。不指定n则返回最后一个元素。结构和前面的initial有点类似，用slice分割数组，这里传给slice的只有一个开始下标（下标为length - n），于是slice默认切割到数组末尾。 

--- 

```
_.rest = _.tail = _.drop = function(array, n, guard) {
    return slice.call(array, n == null || guard ? 1 : n);
  };
```

返回从n开始的余下数组元素，基本方法都类似，还是slice实现。

---

```
_.compact = function(array) {
    return _.filter(array, _.identity);
  };
```

清除数组中值为false的元素（包括false，null，0和"", underfined和NaN）。用一个filter来过滤identity，也就是元素值。

---

下面这个flatten是重点，也是面试时容易出问题。

```
 var flatten = function(input, shallow, strict, output) {
    if (shallow && _.every(input, _.isArray)) {
      return concat.apply(output, input);
    }
    for (var i = 0, length = input.length; i < length; i++) {
      var value = input[i];
      if (!_.isArray(value) && !_.isArguments(value)) {
        if (!strict) output.push(value);
      } else if (shallow) {
        push.apply(output, value);
      } else {
        flatten(value, shallow, strict, output);
      }
    }
    return output;
  };
```

将嵌套数组归纳为一个数组。可传入一个shallow参数以限制嵌套层数为1。当只需归纳一层，并且所有需要归纳的对象都是数组的时候，解法就很简单了，用concat依次连接各数组即可。

当需要处理多层嵌套的时候，就需要使用递归了。遍历每个归纳对象，如果对象不是数组，则处理结束，压入结果数组。如果对象是数组，但要求shallow，也可以压入结果数组。但如果是数组且无shallow要求，则递归这个数组，往该数组下层进行处理，直到递归到以上两种简单情况。

我个人在codewar上已经预见好几次这个题目了，有一题还要求控制递归层数，有机会再复习一下。

下面这个就只是上面flatten的包装函数了，定义了strict为false，output初始值是[]。

```
 _.flatten = function(array, shallow) {
    return flatten(array, shallow, false, []);
  };
```

--- 

```
_.without = function(array) {
    return _.difference(array, slice.call(arguments, 1));
  };
```

返回一个不包括指定值的数组部分。这里得先看difference函数：

```
  _.difference = function(array) {
    var rest = flatten(slice.call(arguments, 1), true, true, []);
    return _.filter(array, function(value){
      return !_.contains(rest, value);
    });
  };
```

返回不出现在另一数组中的数值集合。核心是一个filter函数，针对array（第一个参数）过滤，返回不包含在第二个参数rest中（利用contain检查）的数值集合。这里rest使用了flatten方法，因为第二个参数是个笼统的说法，我们可以传递多个数组作为第二个参数。

without实际上只是调用了difference，利用difference去掉了slice.call(arguments, 1)，也就是除第一个参数外的其他参数。

--- 

```
  _.uniq = _.unique = function(array, isSorted, iteratee, context) {
    if (array == null) return [];
    if (!_.isBoolean(isSorted)) {
      context = iteratee;
      iteratee = isSorted;
      isSorted = false;
    }
    if (iteratee != null) iteratee = _.iteratee(iteratee, context);
    var result = [];
    var seen = [];
    for (var i = 0, length = array.length; i < length; i++) {
      var value = array[i];
      if (isSorted) {
        if (!i || seen !== value) result.push(value);
        seen = value;
      } else if (iteratee) {
        var computed = iteratee(value, i, array);
        if (_.indexOf(seen, computed) < 0) {
          seen.push(computed);
          result.push(value);
        }
      } else if (_.indexOf(result, value) < 0) {
        result.push(value);
      }
    }
    return result;
  };
```

产生一个无重复值的集合。可传递isSorted来使用快排算法。若集合已排序，去除重复值只需比较前一个数值就可以了：

```
if (isSorted) {
        if (!i || seen !== value) result.push(value);
        seen = value;
```

这里!i是为了检验数组首个元素。seen == value则是保存当前数值，以便下次遍历做比较。

如果提供了比较函数，则使用比较函数计算比较值，然后检查该数值是否未出现在seen数组中。

最后一种情况就是数组未排序，也没有比较函数。将遍历的值直接压入result数组，每次遍历检查当前数值是否已存在result中，若不存在继续压入。

---

```
 _.union = function() {
    return _.uniq(flatten(arguments, true, true, []));
  };
```

返回集合的并集。在内部先用flatten将参数归并为一个数组，再利用uniq过滤掉其中重复的部分，于是便得到并集了。

--- 

```
 _.intersection = function(array) {
    if (array == null) return [];
    var result = [];
    var argsLength = arguments.length;
    for (var i = 0, length = array.length; i < length; i++) {
      var item = array[i];
      if (_.contains(result, item)) continue;
      for (var j = 1; j < argsLength; j++) {
        if (!_.contains(arguments[j], item)) break;
      }
      if (j === argsLength) result.push(item);
    }
    return result;
  };
```

取交集，即各集合共有的部分。这里估计取了第一个集合array做参数引用，因为是交集，所以只需遍历其中一个集合，并找出在该集合，也在其他集合中得元素就可以了。于是我们看到了一个双重循环。第一个for遍历array，对它的每个元素检查是否已存在结果中，若已存在则直接跳过。若不存在，则进入第二个for，检查该元素是否在其他集合中（if (!_.contains(arguments[j], item)) break;）。如果这里没有执行break，则是一个共同元素，将它存入结果中。这里使用双重循环效率是比较低的，但因为第二个循环一般数字不大，也可以接收。

---

```
 _.difference = function(array) {
    var rest = flatten(slice.call(arguments, 1), true, true, []);
    return _.filter(array, function(value){
      return !_.contains(rest, value);
    });
  };
```

于intersection相反，返回不存在其他集合中的元素。前面已经介绍过了，这里再复习一下。现将除第一个集合外的其他集合flatten成一个大集合，再对第一个集合的元素逐个过滤，只返回那些不包含在大集合中的元素。

---

```
 _.zip = function(array) {
    if (array == null) return [];
    var length = _.max(arguments, 'length').length;
    var results = Array(length);
    for (var i = 0; i < length; i++) {
      results[i] = _.pluck(arguments, i);
    }
    return results;
  };
```

将多个数组中同一下标的元素归并到一个数组。先找出最终结果数组的长度：var length = _.max(arguments, 'length').length;，该长度等于所有参数数组中长度最大的那一个。结果数组由此确定长度：var results = Array(length);。然后就可以利用pluck函数依次提取各参数数组的元素。我们得先看下pluck：

```
_.pluck = function(obj, key) {
    return _.map(obj, _.property(key));
  };
```

内部其实就是一个map，取出key表示的obj属性。于是_.pluck(arguments, i);就是取出各个参数中下标i代表的元素了。

--- 

```
 _.unzip = function(array) {
    var length = array && _.max(array, 'length').length || 0;
    var result = Array(length);

    for (var index = 0; index < length; index++) {
      result[index] = _.pluck(array, index);
    }
    return result;
  };
```
  
和zip正相反，将各数组按下标归并。即各数组的第一个元素归并为一个数组，第二个元素归并为第二个数组，等等。先按各参数数组的最大长度确立结果数组的长度，然后依次遍历参数数组，将各下标元素存入对应的结果数组中。经过上面的几个函数分析，这个函数也挺容易理解的。

--- 

```
 _.object = function(list, values) {
    var result = {};
    for (var i = 0, length = list && list.length; i < length; i++) {
      if (values) {
        result[list[i]] = values[i];
      } else {
        result[list[i][0]] = list[i][1];
      }
    }
    return result;
  };
```

将数组转换为对象，可以传入key/value对，也可以传入一个可以数组，一个value数组。

这里依然是一个遍历。但因为传入的参数可以不同，在每次遍历有一个if判断是否有第二个value数组。如果有，则list是key数组，将result的key值设置为对应下标的value。如果没有values，则是key/value对，于是list[i][0]是key，而list[i][1]是value。

--- 

```
 _.indexOf = function(array, item, isSorted) {
    var i = 0, length = array && array.length;
    if (typeof isSorted == 'number') {
      i = isSorted < 0 ? Math.max(0, length + isSorted) : isSorted;
    } else if (isSorted && length) {
      i = _.sortedIndex(array, item);
      return array[i] === item ? i : -1;
    }
    if (item !== item) {
      return _.findIndex(slice.call(array, i), _.isNaN);
    }
    for (; i < length; i++) if (array[i] === item) return i;
    return -1;
  };
```

和原生的indexOf类似，返回数组中第一个item的下标。不同的是，如果数组已排序，可传入第三个参数true，以便采用二分查找（_.sortedIndex(array, item)）。


```
  if (item !== item) {
      return _.findIndex(slice.call(array, i), _.isNaN);
    }
```

我一度不太理解item !== item这一句，后来从github更新记录中发现，原来是为了处理array包含NaN的情况，比如：

```
trictEqual(_.indexOf([1, 2, NaN, NaN], NaN), 2, 'Expected [1, 2, NaN] to contain NaN');
```

我们知道，JS诡异的地方在于NaN !== NaN居然是是true。于是这里先检测NaN的情况，然后按isNaN的情况查找。
 

如果第三个参数是数字，则表示需要从该数字下标找起。于是return _.findIndex(slice.call(array, i), _.isNaN);将数组从i下标分割后查找。

最后，如果都找不到则返回-1。

---

上面如果懂了的话，下面这个几乎是一样，只不过从尾部查找而已：


```
_.lastIndexOf = function(array, item, from) {
    var idx = array ? array.length : 0;
    if (typeof from == 'number') {
      idx = from < 0 ? idx + from + 1 : Math.min(idx, from + 1);
    }
    if (item !== item) {
      return _.findLastIndex(slice.call(array, 0, idx), _.isNaN);
    }
    while (--idx >= 0) if (array[idx] === item) return idx;
    return -1;
  };
```

---

下面这个是最近新加入的函数：


```
function createIndexFinder(dir) {
    return function(array, predicate, context) {
      predicate = cb(predicate, context);
      var length = array != null && array.length;
      var index = dir > 0 ? 0 : length - 1;
      for (; index >= 0 && index < length; index += dir) {
        if (predicate(array[index], index, array)) return index;
      }
      return -1;
    };
  }
```

主要是为findIndex和findLastIndex准备的，也就是返回第一个满足条件的元素的下标。这里的dir用于区分从前找起还是从后找起。

确定了查找方向后，返回的函数接收三个参数，一个对象数组，一个检测条件和可选的环境绑定。

这里的cb时一个内部函数，用于产生callback：

```
 var cb = function(value, context, argCount) {
    if (value == null) return _.identity;
    if (_.isFunction(value)) return optimizeCb(value, context, argCount);
    if (_.isObject(value)) return _.matcher(value);
    return _.property(value);
  };
```

在这里我们遇到的时isFunction的情况，结果是调用optimizeCb:

```
var optimizeCb = function(func, context, argCount) {
    if (context === void 0) return func;
    switch (argCount == null ? 3 : argCount) {
      case 1: return function(value) {
        return func.call(context, value);
      };
      case 2: return function(value, other) {
        return func.call(context, value, other);
      };
      case 3: return function(value, index, collection) {
        return func.call(context, value, index, collection);
      };
      case 4: return function(accumulator, value, index, collection) {
        return func.call(context, accumulator, value, index, collection);
      };
    }
    return function() {
      return func.apply(context, arguments);
    };
  };
```

我们并没有传入argCount参数，于是转到case 3，也就是简单的函数call而已了。

回到createIndexFinder。通过dir确定index的值，然后遍历数组，依次调用刚才得到的callback，只要为真就立刻返回该下标index。

于是findIndex和findLastIndex都搞清楚了。

--- 

```
_.sortedIndex = function(array, obj, iteratee, context) {
    iteratee = cb(iteratee, context, 1);
    var value = iteratee(obj);
    var low = 0, high = array.length;
    while (low < high) {
      var mid = Math.floor((low + high) / 2);
      if (iteratee(array[mid]) < value) low = mid + 1; else high = mid;
    }
    return low;
  };  
```

找出对象在数组中的插入位置，可提供一个比较函数来计算位置。开始还是跟前面差不多，利用cb来创建callback。这次的argCount是1，于是得到的时return func.call(context, value);这样一个callback，只绑定环境和一个参数。

下面就是标准的二分查找了，这里不再详述。

--- 

下面是array系列的最后一个函数了：


```
 _.range = function(start, stop, step) {
    if (arguments.length <= 1) {
      stop = start || 0;
      start = 0;
    }
    step = step || 1;

    var length = Math.max(Math.ceil((stop - start) / step), 0);
    var range = Array(length);

    for (var idx = 0; idx < length; idx++, start += step) {
      range[idx] = start;
    }

    return range;
  };
````

用于产生一个序列的整数，类似很多语言中的[0...10]这样一个语法。

start可省略，默认为0. step默认为1；当step为负数时，可取负数序列。

这里先计算整个数组的长度(stop - start) / step，然后就依照长度和step依次叠加产生数列了。



##Functions

上来先是几个help function：

```
var baseCreate = function(prototype) {
    if (!_.isObject(prototype)) return {};
    if (nativeCreate) return nativeCreate(prototype);
    Ctor.prototype = prototype;
    var result = new Ctor;
    Ctor.prototype = null;
    return result;
  };
```

创建一个继承prototype参数的对象。


这里有几个新版加入的快键方式：

```
var
    nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind,
    nativeCreate       = Object.create;
```

好，现在看起来就容易懂了。第一句判断参数是否正确，prototype不是对象的话，无法创建有继承关系的新对象，就直接返回一个空对象了。

然后判断当前环境是否支持Object.create（至少ES5）。支持的话就直接用Object.create建立关系。使用这个方法的原因参考you don't know JS系列吧。

如果不支持Object.create，就只能采用new来建立关系了。这里Ctor是一个空函数：

```
 var Ctor = function(){};
```

先将这个函数的prototye连接到参数上，在利用Ctor做constructor建立有连接关系的对象。这里需要连接new的原理，还是建议读读You don't know JS系列。简单说一下，new创建的对象的prototype，会连接到Ctor的prototype上。

--- 

```
 _.bind = function(func, context) {
    if (nativeBind && func.bind === nativeBind) return nativeBind.apply(func, slice.call(arguments, 1));
    if (!_.isFunction(func)) throw new TypeError('Bind must be called on a function');
    var args = slice.call(arguments, 2);
    var bound = function() {
      return executeBound(func, bound, context, this, args.concat(slice.call(arguments)));
    };
    return bound;
  };
```

原生bind的一个封包，绑定函数和其执行对象。

第一句判断知否支持原生bind，支持就直接使用原生bind。

如果不支持原生bind，就调用内部的executeBound：

```
return executeBound(func, bound, context, this, args.concat(slice.call(arguments)));
```


```
  var executeBound = function(sourceFunc, boundFunc, context, callingContext, args) {
    if (!(callingContext instanceof boundFunc)) return sourceFunc.apply(context, args);
    var self = baseCreate(sourceFunc.prototype);
    var result = sourceFunc.apply(self, args);
    if (_.isObject(result)) return result;
    return self;
  };
```

这个函数主要判断是将函数作为constructor执行，还是作为普通函数执行。如果this是包装函数bound的实例，就按普通函数apply调用。

如果不是，则创建一个继承func的对象self。用self作为绑定对象调用sourceFunc。


--- 

