# 原型和原型链

### 理解原型设计模式以及 JavaScript 中的原型规则

#### 什么是原型模式

**原型模式(Prototype pattern)**:`通俗点讲就是创建一个共享的原型,并通过拷贝这些原型创建新的对象。 用于创建重复的对象,这种类型的设计模式属于创建型模式,它提供了一种创建对象得比错选择。`

#### 实现原型模式

我们可以通过 JavaScript 特有的原型集成特性去实现原型模式,也就是创建一个对象作为另一个对象的 prototype 属性值,我们可以通过 Object.create(prototype,optionalDescriptorObjects)来实现原型继承。

```
var fatherPrototype = {
  init: function(money) {
    this.selfMoney = money;
  },
  getMoney: function() {
    console.log("取钱" + this.selfMoney);
  }
};

function sonMoney(Smoney) {
  function F() {}
  F.prototype = fatherPrototype;
  var f = new F();
  f.init(Smoney);
  return f;
}

var sonGetMoney = sonMoney("20");
sonGetMoney.getMoney();
//儿子取钱取得是他爹的
```

**原型模式,就是创建一个共享的原型,通过拷贝这个原型来创建新的类,用于创建重复的对象,带来性能上的提升。**

### instanceof 的底层实现原理,手动实现一个 \_instanceof

先来看个例子

```
function Person(name, age) {
  this.name = name;
  this.age = age;
}

function Student(score) {
  this.score = score;
}

Student.prototype = new Person("李明", 22);
var s = new Student(99);

console.log(s instanceof Student); //true
console.log(s instanceof Person); //true
console.log(s instanceof Object); //true
```

检测对象 A 是不是另一个对象 B 的实例的原理：**查看对象 B 的 prototype 属性指向的原型对象是否在对象 A 的原型上,若在则返回 true,若不在则返回 false。**

```
function _instanceof(ins, cons) {
  while (true) {
    if (ins.__proto__ === null) {
      return false;
    }
    if (ins.__proto__ === cons.prototype) {
      return true;
    }
    ins = ins.__proto__;
  }
}
```

\_instanceof 实际上是根据原型链向上追溯,子类找不到找父类,直到 ins.\_\_proto\_\_指向的就是 cons.prototype,否则,当 ins 已经追溯到 Object.prototype,由于 Object.prototype 是没有\_\_proto\_\_的,所以当 ins.**proto** === null 时,就说明我们的循环到最后也没有成功使 ins.**proto** === cons.prototype,那么 ins 就不是 cons 或 cons 的子类实例化出来的对象,函数返回 false。

### 实现继承的几种方式以及他们的优缺点

* 构造继承
* 原型继承
* 实例继承
* 拷贝继承

下面谈一下前三种的继承方式

```
function Father(){}
function Son{}

//方法一
Son.prototype = new Father();

//方法二
Son.prototype = Father.prototype;

//方法三
Son.prototype = Object.create(Father.prototype);
```

***

* 方法一：
  * 优点：正确设置原型链实现继承
  * 优点：父类实例属性得到继承,原型链查找效率提高,也能为一些属性提供合理的默认值。
  * 缺点：父类实例属性为引用类型时,不恰当的修改会导致所有子类被修改。
  * 缺点：创建父类实例作为子类原型时,可能无法正确构造函数需要的合理参数,这样提供的参数继承给子类没有实际的意义,当子类需要这些参数时应该在构造函数中进行初始化设置。
  * **总结**：继承应该是继承方法而不是继承属性,为子类设置父类实例属性应该是通过在子类构造函数中调用父类构造函数进行初始化。

***

* 方法二：
  * 优点：正确设置原型链实现继承；
  * 缺点：父类构造函数原型与子类相同。修改子类原型添加方法会修改父类。

***

* 方法三：
  * 优点：正确设置原型链且避免方法 1-2 中的缺点
  * 缺点：ES5 方法需要注意兼容性。

***

*   改进

    * 所有三种方法应该在子类构造函数中调用父类构造函数实现实例属性初始化。

    ```
    function Son() {
      Father.call(this);
    }
    ```

    * 用新创建的对象替代子类默认原型,设置`Son.prototype.constructor = Son;`保证一致性；
    * 第三种方法的延伸版本

    ```
    function create(obj) {
      if (Object.create) {
        return Object.create(obj);
      }

      function f() {}
      f.prototype = obj;
      return new f();
    }
    ```

### new 一个对象的详细过程,手动实现一个 new 操作符

**`new`运算符**创建一个用户定义的对象类型的实例。 使用`new`命令时,它后面的函数依次执行下面的步骤。

* 创建一个空对象,作为将要返回的对象实例。
* 将这个空对象的原型,指向构造函数的`prototype`属性。
* 将这个空对象赋值给函数内部的`this`关键字。
* 开始执行构造函数内部的代码。

也就是说,构造函数内部,`this`指的是一个新生成得空对象,所有针对`this`的操作,都会发生在这个空对象上。构造函数之所以叫”构造函数”,就是说这个函数的目的,就是操作一个空对象(即`this`对象),将其”构造”为需要的样子。

`new`命令简化的内部流程：

```
/**
 *
 * @param {*} constructor 构造函数
 * @param {*} params 构造函数参数
 */

function _new(constructor, params) {
  // 将arguments对象转化为数组
  var args = [].slice.call(arguments);
  // 取出构造函数
  var constructor = args.shift();
  // 创建一个空对象,继承构造函数的prototype属性
  var context = Object.create(constructor.prototype);
  // 执行构造函数
  var result = constructor.apply(context, args);
  // 如果返回结果是对象,就直接返回,否则返回context对象
  return typeof result === "object" && result != null ? result : context;
}

var actor = _new(Person, "张三", 28);
```
