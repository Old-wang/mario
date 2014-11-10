# Underscore 源码阅读

JavaScript 的函数是一级对象，可以以值的形式进行传递(函数对象可以作为参数，也可以作为返回值)。
对 JavaScript 的函数的理解是学习 JavaScript 中的一个重要的步骤。

Underscore 是一个小巧的工具类库，提供了很多有用的函数式编程工具。
对 Underscore 源码进行阅读可以更好地掌握对这个类库的使用，也能增进自己对 JavaScript 中函数式编程的理解。

本文基于 Underscore [1.7.0](./underscore-1.7.0.js) 版本。
在官网 http://underscore.org 上还有一份带注解的源码文档，感兴趣的可以参考。

## 通用初始化块

> ### 即时调用函数
> 
> 在类库的书写中，类库作者需要尽量避免变量泄露到全局变量中，以防和其它引用的 Js 代码产生冲突。
> 我们知道，在 JavaScript 只有函数的执行才能产生一个作用域，所以大量类库都采用了即时调用函数来生成作用域，
> 所有需要使用到的变量都会限制在这个作用域中。

```javascript
(function() {
  // put your code here...
})();
```

Underscore 的目标执行环境是浏览器和 NodeJs。Underscore 并不支持 AMD 模块规范。
在第 12 行，定义了根对象，在浏览器中就是 `window` 对象，在服务器端就是 `exports` 对象。

```javascript
var root = this;
```

类库会给全局对象暴露一个变量名，所以通常类库都会提供方法，给调用者提供处理命名冲突的方法。流程类似于：

```javascript
var previousUnderscore = root._;
var _ = myImpl;
root._ = _;
_.noConflict = function() {
  root._ = previousUnderscore;
  return this;
};
```

这样在加载完类库后，调用者可以调用 `var someVar = _.noConflict()` 得到这个类库对象，而原来同名的 `_` 可以继续使用。

在 18~33 行，给一些常用的原型对象和函数定义别名，便于使用。

第 36 行定义了 `_` ，它是一个构造函数，对传入的参数做了一个封装。
后续会给 `_` 构造函数的原型对象进行扩展，这样就添加了很多有用的方法，便于链式调用。

```javascript
var _ = function(obj) {
  if (obj instanceof _) return obj;
  if (!(this instanceof _)) return new _(obj);
  this._wrapped = obj;
};
```

第 45 行对 NodeJs 模块定义导出对象，对浏览器直接挂到全局对象上：

```javascript
  if (typeof exports !== 'undefined') {
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {
    root._ = _;
  }
```

第 60 行是一个内部使用的函数，根据参数个数返回一个回调函数，这个方法在内部各种迭代器中会使用到。

```javascript
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

其实际执行效果其实就是给函数绑定一个 context 上下文对象给 `this`，而且其实直接返回最后一个函数效果是一样。
但在大量测试结果中，当前浏览器引擎直接引用上述 4 个版本执行速度更快，所以使用了这个函数进行了优化。

> ### `call`、`apply`、`bind` 和 `this`
>
> 函数调用时，`this` 会动态地指向一个上下文对象。如果是对象方法调用，那么 `this` 会指向这个对象。
> 如 `a.doSomething()` 中 `this` 会指向对象 `a`。而如果函数直接调用，则 `this` 会指向全局对象。
> 如 `doSomething()` 中 `this` 会指向全局对象。
> 通过函数的 `call` 和 `apply` 方法来调用函数，我们可以把函数执行中的 `this` 指向其第一个参数。
> 在 ES5 规范中给 Function.prototype 添加了 `bind`，可用于将函数的 `this` 上下文永久地绑定到某个对象上。

第 84 行，创建了一个内部使用的用于创建迭代器的函数：

```javascript
  _.iteratee = function(value, context, argCount) {
    if (value == null) return _.identity;
    if (_.isFunction(value)) return createCallback(value, context, argCount);
    if (_.isObject(value)) return _.matches(value);
    return _.property(value);
  };
```

如果迭代参数是 `null` 或 `undefined` 那么，返回的迭代器是标识函数，即返回参数对象本身的函数；
如果迭代参数是一个函数，则调用上面的 `createCallback` 创建一个绑定了上下文的回调；
如果迭代参数是一个对象，说明迭代器的每个属性都需要和这个对象中的属性相匹配；
其他情况，认为迭代参数是一个字符串，迭代器就是返回对象这个对应属性的函数。
其内部调用的函数在后面的代码中会有实现。

## 集合相关函数

> It is better to have 100 functions operate on one data structure than 10 functions on 10 data structures.

数组是有序的元素集合，对象是无序的键值对的集合。
在抽象的角度上看，它们都可以看做集合，我们可以遍历集合、将集合中每个元素映射为另一个集合中的元素、过滤元素等等。

程序就是将输入数据经过转换聚合等操作加工成新的数据输出的过程。
在函数式语言中，操作对象基本都是集合，因此最著名的函数式语言直接取 LISt Processing 而成 LISP。

对于集合的基本高阶函数有:

* `map` - 遍历集合，将每个元素转换成另一个元素，返回新元素组成的集合
* `reduce` - 遍历集合聚合出一个总的结果
* `filter` - 过滤集合，返回匹配指定条件的元素集合

旧版本的 JavaScript 中，数组直接面向实现，并没有这些函数式原语。
随着语言的发展，在 ES5 中对语言引入了一些新的特性，其中数组相关的有：

* `Array.prototype.forEach`
* `Array.prototype.map`
* `Array.prototype.reduce`
* `Array.prototype.reduceRight`
* `Array.prototype.filter`
* `Array.prototype.every`
* `Array.prototype.some`
* `Array.prototype.indexOf`
* `Array.prototype.lastIndexOf`
* `Array.isArray`

从抽象的角度看，其实这里很多方法不仅仅可以针对数组，对对象而言也应该适用。
所以 Underscore 这样的类库主要是给数组和对象提供统一的函数式原语，并且对低版本的 JavaScript 也兼容。

在早期版本的 Underscore 中，它会检测 `Array.prototype` 是否有原生的 `forEach` 或 `map` 等函数。
如果存在则直接调用原生函数，如果不存在则使用自身的实现。
但实际测试中，不同的浏览器实现的性能参差不齐，且原生的函数暂时并没有性能优势，所以现在版本的实现中，直接使用相应的方法实现。

```javascript
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

这里传入的 obj 应该是一个数组或对象；iteratee 是一个函数，它有三个形参，第一个是值，第二个是键，第三个是集合对象本身。
它调用了 `createCallback` 来绑定上下文。

这里使用 `length === +length` 来判断对象是否具有数字类型的 length 属性；如果有 length 属性则按照数组方式进行迭代，
否则按照对象方法迭代其键值对。这里没有用 `_.isArray` 来作判断，给迭代 *伪数组对象* 提供了空间。

```javascript
var a = {'0': 1, length: 2};
[].push.call(a, 'test');
console.log(a); // Object {0: 1, 2: "test", length: 3}
```

实际上 JavaScript 中的数组的实现比较讨巧。JavaScript 引擎本身并没有像 C 语言一样有对应硬件的数组。
JavaScript 中的数组基于对象来实现，可认为是特殊的对象。它有一个 `length` 属性来指示当前数组的长度。
除此以外，它就可以被认为是键为索引数字的对象了。
因此 `Array.prototype` 上的函数基本上可以用来操作任何带有 `length` 属性的对象。

最常见的伪数组对象是函数内部的 `arguments` 对象，它是调用函数时所有传入的参数的集合，但它并不是一个数组。
`arguments` 对象给我们实现不定数量参数函数和多态函数提供了着手点。
所以 `Function.prototype.apply` 的第二个型参其实接收伪数组对象。

最后它返回集合本身，以支持链式调用。

```javascript
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

`map` 的实现中，使用了 `_.iteratee` 来生成迭代器。
然后它也是判断 `length` 属性来确定返回的数组的长度和迭代方式。

```javascript
  var reduceError = 'Reduce of empty array with no initial value';

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

`reduce` 遍历集合聚合成一个结果。这里 `iteratee` 参数是一个函数，其第一个形参为每次迭代的中间结果，
后面的形参依次为集合元素的值、键及集合本身。
这里通过 `arguments.length < 3` 而不是 `memo == null` 来判断是否传入初始值，是因为有可能传入的初始值恰好是 null，
但 arguments.length 可以知道调用者确实传入了初始值。
迭代的过程类似于 `map` 的实现，只是返回的是单个结果。

某些特定的聚合过程和遍历集合的方向有关，或者特定实现的遍历效率需要从右往左聚合。
所以和 `reduce` 对应提供了一个 `reduceRight` 实现。

```javascript
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

`find` 方法找到第一个满足条件的元素。它内部调用了后面定义的 `some` 函数。`find` 应该是一种常见的使用模式而提取出来。
在函数式编程中最基础的高阶函数就是 `map`、`reduce`、`filter`，其他函数是在它们的基础上实现的。

```javascript
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

在 Underscore 的实现中，`each` 给其他高阶函数提供了基础。`filter` 的实现就很简单，如果满足条件就把值放到结果集中。
`reject` 恰好和 `filter` 相反，把满足条件的值从集合中移除。

```javascript
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

`every` 在集合中的元素都满足指定条件时返回 `true`，否则返回 `false`。所以这里在遇到第一个不满足的元素时其实就可以跳出循环。
在低版本的 Underscore 里，`every` 是调用 `each` 来迭代集合，所以以前 `each` 定义了一个 `breaker` 私有空对象来跳出循环。
在这里直接使用 for 循环来迭代集合，所以可以直接通过在循环中 `return false;` 来跳出循环，但选择迭代方式等需自行定义。
这里弃用 `each` 来迭代主要是性能因素。

`some` 方法和 `every` 相反，只有集合中有某个元素满足条件就返回 `true`，否则返回 `false`。这里它的实现和 `every` 雷同。

```javascript
  _.contains = _.include = function(obj, target) {
    if (obj == null) return false;
    if (obj.length !== +obj.length) obj = _.values(obj);
    return _.indexOf(obj, target) >= 0;
  };
```

`contains` 判断集合的值中是否包含某个元素。它通过 `===` 来判断相等性。

```javascript
  _.invoke = function(obj, method) {
    var args = slice.call(arguments, 2);
    var isFunc = _.isFunction(method);
    return _.map(obj, function(value) {
      return (isFunc ? method : value[method]).apply(value, args);
    });
  };
```

`invoke` 会给集合中的每个元素应用第二个参数所表示的函数，如果有三个或以上的参数，则后面的这些参数都作为这个函数的参数传递进去。
返回的结果是每个元素应用函数的结果所组成的数组。

```javascript
  _.pluck = function(obj, key) {
    return _.map(obj, _.property(key));
  };
```

`pluck` 是 `map` 的一种常见使用模式，我们仅需要从集合的元素中抽取某个属性。

```javascript
  _.where = function(obj, attrs) {
    return _.filter(obj, _.matches(attrs));
  };
```

`where` 和 `findWhere` 分别是 `filter` 和 `find` 的常见使用场景，过滤(或找到第一个)满足所有指定键值对的元素。

```javascript
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

`max` 和 `min` 代码雷同，分别是找到集合中最大(或最小)的元素。可以传递一个用于计算比较值的函数。
如果没有这个函数，那么出于性能因素，这里直接使用 `for` 循环进行迭代。
如果有该函数，则通过一个 `result` 用于存放最大(或最小)元素，一个 `lastComputed` 存放最大(或最小)比较值，
然后通过 `each` 迭代找到集合中的最大(或最小)元素。

实际上在 Underscore 的高版本的代码中处处可以看到这种因为性能因素而做出的在代码复杂度上的妥协。

```javascript
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

`shuffle` 用于将集合中的元素进行洗牌。在很多排序算法中，为了避免应元素初始有序而出现最差情况，通过一次洗牌将元素随机打乱，
可使元素处于平均情况。
这里的洗牌算法采用 Fisher-Yates 洗牌算法，其思路是从左到右调整元素位置，每次调整的时候随机选择左侧的一个元素，
将它和当前需调整元素进行换位。
该算法实现简单，而且可以原地洗牌。这里因为面向函数式编程，所以没有进行原地洗牌，而是返回一个新的洗牌后的集合。
这也是函数式编程的一个重要思想，尽量限制函数的副作用。

```javascript
  _.sample = function(obj, n, guard) {
    if (n == null || guard) {
      if (obj.length !== +obj.length) obj = _.values(obj);
      return obj[_.random(obj.length - 1)];
    }
    return _.shuffle(obj).slice(0, Math.max(0, n));
  };
```

`sample` 用于从集合中随机取 n 个元素的样本。它调用了上面的洗牌算法，然后取洗牌后的前 n 个元素。
在常规的 `_.sample` 使用中，我们通常仅会有两个参数。
在作为 `_.map` 的迭代器使用时，我们知道其第一个参数是元素值，第二个参数是元素索引(键)，第三个参数是集合对象。
这个时候，我们通常只希望每个元素仅随机抽取一个值；而且我们希望代码仅可能简单，如：

```javascript
_.map([[1, 2, 3], [4, 5]], _.sample);
```

而不是：

```javascript
_.map([[1, 2, 3], [4, 5]], function(arr, i, coll) {
	return _.sample(arr, null);
});
```

因此这个时候，实际上我们传入了第三个额外的参数，如果它存在的话，我们仅取样一个元素。
所以这种 `guard` 参数的特殊用法仅是为了给 `_.map` 作为迭代器提供便利。
在后续的数组相关函数中，我们还可以看到多个类似 `guard` 参数的使用。

```javascript
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

`sortBy` 对集合中的元素进行排序。
这里的排序条件调用了 `_.iteratee`，可以记得上面它支持没有参数、函数参数、对象参数和字符串参数四种类型的参数来生成对应的迭代器。
这里的流程是先通过 `map` 将集合映射成带有 `value`、`index` 和 `criteria` 三个属性的对象的数组；
然后这个数组对象通过通用排序函数进行排序；最后通过 `pluck` 取出排序后数组的 `value` 属性组成排序结果。

```javascript
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

  _.indexBy = group(function(result, value, key) {
    result[key] = value;
  });

  _.countBy = group(function(result, value, key) {
    if (_.has(result, key)) result[key]++; else result[key] = 1;
  });
```

`groupBy`、`indexBy` 和 `countBy` 都是将集合聚合成一个 `map`，它的键是通过函数计算出来的键，
值分别是对应键的元素数组、元素值或元素数量。
在常见的函数式编程里，它们分别的实现可能都是使用 `reduce` 函数来实现。但考虑到聚合结果都是一个 `map`，且生成键的逻辑其实是一致的，
所以这里提取出一个统一的 `group` 内部实现，它仅在具体如何将对象的元素聚合到 `map` 上的行为有差异。

```javascript
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

`sortedIndex` 找到某个元素在一个以排序数组中的插入位置，这里使用了二分查找。

```javascript
  _.toArray = function(obj) {
    if (!obj) return [];
    if (_.isArray(obj)) return slice.call(obj);
    if (obj.length === +obj.length) return _.map(obj, _.identity);
    return _.values(obj);
  };
```

`toArray` 将一个集合转换成数组。最常见的是将一个伪数组对象转成数组。
这里如果是数组对象，就调用 `slice` 方法来获取数组的一个完整拷贝；如果是伪数组对象，通过 `map` 将其映射为新的数组；
其他情况返回对象的值所组成的数组。
实际上对伪数组对象像数组一样调用 `slice` 方法也是可行的。但在旧版本的 IE 中 `slice` 的实现可能不支持伪数组对象，
因此这里是通过 `map` 进行映射。

```javascript
  _.partition = function(obj, predicate, context) {
    predicate = _.iteratee(predicate, context);
    var pass = [], fail = [];
    _.each(obj, function(value, key, obj) {
      (predicate(value, key, obj) ? pass : fail).push(value);
    });
    return [pass, fail];
  };
```

`partition` 根据条件将集合中的元素分成满足条件和不满足条件的两部分。

至此，集合相关的函数告一段落。

我们可以看一下集合函数中第二个参数 `iteratee` 或 `predicate` 函数。
在 `each` 中，我们通常需要给每个元素执行某些操作，不传这个参数或者传入对象或字符串对这个方法而言并没有显而易见的意义，
所以这个方法中通过 `createCallback` 来创建迭代器；
在 `reduce` 和 `reduceRight` 中，迭代器需传入聚合中间结果和迭代元素信息，然后返回一个聚合的结果，
所以也不能使用 `_.iteratee` 来创建迭代器。
对其他方法而言，迭代器一般仅和当前元素相关，且通常是计算出一个值，因此通过 `_.iteratee` 来创建的迭代器较容易理解。

## 数组相关函数

数组是有序元素集合，所以存在和数组特性相关的一些函数。
它们在调用时需注意传入的参数需为数组。

```javascript
  _.first = _.head = _.take = function(array, n, guard) {
    if (array == null) return void 0;
    if (n == null || guard) return array[0];
    if (n < 0) return [];
    return slice.call(array, 0, n);
  };
  
  _.rest = _.tail = _.drop = function(array, n, guard) {
    return slice.call(array, n == null || guard ? 1 : n);
  };
```

`first` 取数组的前 n 个元素；`rest` 取数组前 n 个元素之后的元素；`initial` 取数组除最后 n 个元素之外的元素；`last` 取数组最后 n 个元素。
在实现上都是通过计算数组需要 `slice` 的两个索引值来实现。
注意这几个函数都有 `guard` 参数，它们都是为了上面说的为 `_.map` 提供便利。

```javascript
  _.compact = function(array) {
    return _.filter(array, _.identity);
  };
```

`compact` 去除所有逻辑 false 的值。

```javascript
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

  _.flatten = function(array, shallow) {
    return flatten(array, shallow, false, []);
  };
```

`flatten` 将一个嵌套的数组展平。

```javascript
_.flatten([1, [2], [3, [[4]]]]); //=> [1, 2, 3, 4];
_.flatten([1, [2], [3, [[4]]]], true); //=> [1, 2, 3, [[4]]]; 
```

可以传入第二个参数，是否浅度展平：如果是的话仅展平一层；否则会递归整个结构进行展平。

```javascript
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

这里 `isSorted` 参数是用于指示原数组是否已是有序的，如果是有序的，可以通过前一个值是否等于当前值来判断是否需将当前值加入结果集中；
否则已有比较值是一个数组，根据迭代器算出比较值后，查看它在已有比较值中是否存在来判断它是否需加入结果集。
这里实际上 `isSorted`、`iteratee` 及 `context` 都是可选参数，且 `isSorted` 在其位置上并非必填。
所以这里通过限制 `isSorted` 为布尔型，判断它是否存在。

```javascript
  _.union = function() {
    return _.uniq(flatten(arguments, true, true, []));
  };
```

`union`、`intersection` 和 `difference` 是集合论中的概念。集(set) 是无序元素的集合，所以这里的数组被当做集(set) 来看待。
`union` 就是给定多个集的并集；`intersection` 是给定多个集的交集；`difference` 是属于第一个集，但不属于其他集的元素的集。
`union` 的实现中直接将给定参数展平，然后给上面的 `_.uniq` 获取所有唯一元素的集。

```javascript
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

`intersection` 的实现中以第一个数组作为基础，遍历每个元素查看在其他数组中是否存在。

```javascript
  _.difference = function(array) {
    var rest = flatten(slice.call(arguments, 1), true, true, []);
    return _.filter(array, function(value){
      return !_.contains(rest, value);
    });
  };
```

`difference` 的实现中将其他数组展平，然后过滤出第一个数组中不属于展平数组的元素。

```javascript
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

`zip` 即拉链，即每个数组中第一个元素依次排列到一起，然后排列第二个元素，依次类推。
在一般的函数式语言中，`zip` 函数得到的是一个展平的列表，且到数组中较短的长度为止：

```clojure
(zip [1 2 3] [4 5 6]) ;=> [1 4 2 5 3 6]
(zip [1 2 3] [4 5]) ;=> [1 4 2 5]
```

但这里 `zip` 直接得到的是一个数组的数组，实现上比较方便。

```javascript
  _.object = function(list, values) {
    if (list == null) return {};
    var result = {};
    for (var i = 0, length = list.length; i < length; i++) {
      if (values) {
        result[list[i]] = values[i];
      } else {
        result[list[i][0]] = list[i][1];
      }
    }
    return result;
  };
```

`object` 将两个键和值的数组或一个键值对数组转换成一个对象。

```javascript
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

  _.lastIndexOf = function(array, item, from) {
    if (array == null) return -1;
    var idx = array.length;
    if (typeof from == 'number') {
      idx = from < 0 ? idx + from + 1 : Math.min(idx, from + 1);
    }
    while (--idx >= 0) if (array[idx] === item) return idx;
    return -1;
  };
```

前面讲了 ES5 中引入了 `Array.prototype.indexOf` 和 `Array.prototype.lastIndexOf` 用于查找元素在数组中的第一个索引值和最后一个索引值。
因为在旧版本的 IE 中并没有这两个方法，所以 Underscore 提供了这两个方法。
在 `indexOf` 的实现中，第三个参数是查找的起始索引。这里如果起始索引存在且不是数字的话，说明是给了已有数组是已排序数组的提示。
这样可以调用上面的 `_.sortedIndex` 进行二分查找。

```javascript
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
```

`range` 函数可以指定起止值和步长，生成一个范围内的值的数组。如果只传入一个参数，则认为这个参数是结束值，步长默认为 1。

## 函数相关函数
## 对象相关函数
## 工具函数
## 链式调用