# Vue 响应式实现的前世今生（一）：监听对象的变化

### 前言

&emsp;&emsp;响应式是 Vue 最独特的特性之一，它的底层实现主要分为两部分：

- 监听对象的变化
- 依赖收集和派发通知

&emsp;&emsp;本文就如何实现监听对象变化这一问题，一起研究一下 Vue2 以及 Vue3 的底层实现细节。

## 前世：Vue2 利用 Object.defineProperty 劫持

&emsp;&emsp;核心实现逻辑：*利用 Object.defineProperty 修改已有属性的 getter 和 setter 描述符属性*。

> Object.defineProperty 是 ES5 中一个无法 shim 的特性，所以 Vue2 不支持 IE8 以及更低版本浏览器。

```JavaScript
function defineReactive(target, key) {
    // 缓存当前属性值
    let val = target[key];
    Object.defineProperty(target, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
            console.log('[getter] 属性 key: ', key);
            return val;
        },
        set: function reactiveSetter (newVal) {
            console.log('[setter] 属性 key: ', key, newVal);
            val = newVal;
        }
    });
}

const obj = { foo: 1 };
Object.keys(obj).forEach(key => defineReactive(obj, key));

obj.foo = 100; // [setter] 属性 key:  foo 100
obj.foo; // [getter] 属性 key:  foo
```

### 1、预设描述符属性

&emsp;&emsp;使用 Object.defineProperty 方法是有前提的：*属性的描述符属性 configurable 不能为 false*。

&emsp;&emsp;当 configurable 为 false 时，直接调用 Object.defineProperty 方法会报如下错误：

```JavaScript
const obj = { foo: 1 };
Object.freeze(obj);
Object.keys(obj).forEach(key => defineReactive(obj, key));

// 报错：Uncaught TypeError: Cannot redefine property: foo
```

&emsp;&emsp;那么在调用 Object.defineProperty 方法之前，需要先判断属性的描述符是否合法，从而提高代码的健壮性：

```JavaScript
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
    return
}
```

&emsp;&emsp;另外，*原始对象可能已经通过 Object.defineProperty 定义过一次 setter 和 getter*，所以在劫持的时候应该要保持这一特性：

```JavaScript
function defineReactive(target, key, val) {
    const property = Object.getOwnPropertyDescriptor(target, key)
    if (property && property.configurable === false) {
        return
    }

    const getter = property && property.get;
    const setter = property && property.set;
    if ((!getter || setter) && arguments.length === 2) {
        val = target[key]
    }

    Object.defineProperty(target, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
            const value = getter ? getter.call(target) : val;
            return value;
        },
        set: function reactiveSetter (newVal) {
            const value = getter ? getter.call(obj) : val
            if (newVal === value || (newVal !== newVal && value !== value)) {
                return
            }
            if (getter && !setter) return
            if (setter) {
                setter.call(obj, newVal)
            } else {
                val = newVal
            }
        }
    });
}
```

### 2、深层对象属性

&emsp;&emsp;上述示例代码中，通过对对象的每一个属性执行 Object.defineProperty 方法来实现 setter 和 getter 的拦截，但是实际的开发场景中，对象属性的值也有可能是一个对象，而这种方式是无法监听到深层对象属性的变化的。

&emsp;&emsp;所以需要针对属性值进行递归处理，包含两种情况：

- *getter 中当前属性的值为对象。*
- *setter 中设置的新属性值为对象。*

```JavaScript
// src/core/observer/index.js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      ...
    },
    set: function reactiveSetter (newVal) {
      ...
      childOb = !shallow && observe(newVal)
      ...
    }
  })
}
```

&emsp;&emsp;Vue2 中的 observe 方法内部判断当前属性值是否为对象，如果属性值为对象，则调用 Observer 再对当前属性值进行拦截处理，从而解决深层对象属性的问题。*但是这种方式并不是惰性的，会影响应用初始化的性能*。

&emsp;&emsp;另外，*observe 方法内部通过判断私有属性 __ob__，来解决属性值循环引用的问题*。

### 3、数组

&emsp;&emsp;当 Object.defineProperty 方法作用于数组上时，会有如下限制：

- *无法侦测数组 length 属性变化的场景。*
- *无法侦测利用下标新增元素的场景。*
- *无法侦测通过数组方法改变元素的场景。*

&emsp;&emsp;针对第三种场景，可以通过原型继承+方法重写的方式达到拦截的目的：

```JavaScript
const arrayMethods = Object.create(Array.prototype);
function patchToArrayPushMethod () {
    const original = arrayMethods['push'];
    Object.defineProperty(arrayMethods, 'push', {
    value: function mutator (...args) {
        const result = original.apply(this, args);
        console.log('[array.push] 拦截 push 方法：', args);
        return result;
    },
    enumerable: true,
    writable: true,
    configurable: true
    });
}

patchToArrayPushMethod();

// 改变数组对象的原型链
bar.__proto__ = arrayMethods;

bar.push(90);
```

### 4、属性增删改查

&emsp;&emsp;利用 Object.defineProperty 方法来实现对象劫持的最大局限性就在于：*只能拦截已有属性的查询和更新操作*。

&emsp;&emsp;对于在已经劫持的对象上再新增属性，是无法自动侦测到的，所以在 Vue2 中需要将响应式属性提前定义到 data 中。

&emsp;&emsp;而对于属性的删除，则是超出了 Object.defineProperty 方法的能力范围。

## 今生：Vue3 利用 Proxy 代理

&emsp;&emsp;Vue3 采用了 Proxy 实现对象代理来替代对象劫持，由于 Proxy 在部分浏览器上无法完全 polyfill，所以必须调整框架整体的浏览器支持范围，因此只能在新的 Major 版本发布。

### 1、核心实现

```JavaScript
const foo = { bar: 1 };
const fooProxy = new Proxy(foo, {
    set: function (obj, key, value) {
        const result = Reflect.set(obj, key, value);
        console.log('[proxy.setter] ', key, value);
        return result;
    },
    get: function (obj, key) {
        const result = Reflect.get(obj, key);
        console.log('[proxy.getter] ', key, result);
        return result;
    },
    deleteProperty: function(obj, key) {
        const result = Reflect.deleteProperty(obj, key);
        console.log('[proxy.delete]', key);
        return result;
    }
});

fooProxy.bar = 20; // [proxy.setter]  bar 20

fooProxy.boo = 10; // [proxy.setter]  boo 10

delete fooProxy.boo; // [proxy.delete] boo
```

&emsp;&emsp;相比较 Object.defineProperty 方法，*Proxy 不需要针对对象具体的属性单独处理，从而可以节省 Object.keys 的遍历损耗*。

&emsp;&emsp;

### 数组

### 新数据结构的支持