# 1.程序入口(5925-5930)：

```js
function Vue(options) {
    if (!(this instanceof Vue)) {
        warn$2('Vue is a constructor and should be called with the `new` keyword');
    }
    this._init(options);
}
```

# 2.initMixin$1(4800-4854):

```js
  function initMixin$1(Vue) {
    Vue.prototype._init = function (options) {
      var vm = this; // 指向调用者Vue
      // a uid
      vm._uid = uid++;
      var startTag, endTag; // 性能测试起始和结束标记
      // 如果开启了性能记录，则打点进行记录
      // mark (1480-1502)
      if (config.performance && mark) { 
        startTag = "vue-perf-start:".concat(vm._uid);
        endTag = "vue-perf-end:".concat(vm._uid);
        mark(startTag);
      }
      // 之前已经判断过了，这里直接标记即可
      vm._isVue = true;
      // 避免vue实例被observe
      vm.__v_skip = true;
      // effect scope
      vm._scope = new EffectScope(true /* detached */);
      vm._scope._vm = true;
      // merge options
      if (options && options._isComponent) {
        // optimize internal component instantiation
        // since dynamic options merging is pretty slow, and none of the
        // internal component options needs special treatment.
        initInternalComponent(vm, options);
      } else {
        vm.$options = mergeOptions(resolveConstructorOptions(vm.constructor), options || {}, vm);
      }
      /* istanbul ignore else */
      {
        initProxy(vm);
      }
      // expose real self
      vm._self = vm;
      initLifecycle(vm);
      initEvents(vm);
      initRender(vm);
      callHook$1(vm, 'beforeCreate', undefined, false /* setContext */);
      initInjections(vm); // resolve injections before data/props
      initState(vm);
      initProvide(vm); // resolve provide after data/props
      callHook$1(vm, 'created');
      // 性能记录结束点
      if (config.performance && mark) {
        vm._name = formatComponentName(vm, false);
        mark(endTag);
        measure("vue ".concat(vm._name, " init"), startTag, endTag);
      }
      if (vm.$options.el) {
        vm.$mount(vm.$options.el);
      }
    };
  }
```

# Utils

## 1.mark、measure(1480-1502)：

判断系统是否有windows.performance如果有，使用它的mark、measure

```js
var mark;
var measure;
{
  var perf_1 = inBrowser && window.performance;
  /* istanbul ignore if */
  if (perf_1 &&
    // @ts-ignore
    perf_1.mark &&
    // @ts-ignore
    perf_1.measure &&
    // @ts-ignore
    perf_1.clearMarks &&
    // @ts-ignore
    perf_1.clearMeasures) {
    mark = function (tag) {
      return perf_1.mark(tag);
    };
    measure = function (name, startTag, endTag) {
      perf_1.measure(name, startTag, endTag);
      perf_1.clearMarks(startTag);
      perf_1.clearMarks(endTag);
      // perf.clearMeasures(name)
    };
  }
}
```

2.EffectScope、effectScope、recordEffectScope、getCurrentScope、onScopeDispose(3496-3592)

```js
var activeEffectScope;
var EffectScope = /** @class */ (function () {
    function EffectScope(detached) {
        if (detached === void 0) { detached = false; } 若未定义，则赋值false
        this.detached = detached;
        /**
         * @internal
         */
        this.active = true;
        /**
         * @internal
         */
        this.effects = [];
        /**
         * @internal
         */
        this.cleanups = [];
        this.parent = activeEffectScope;
        if (!detached && activeEffectScope) {
            this.index =
                (activeEffectScope.scopes || (activeEffectScope.scopes = [])).push(this) - 1;
        }
    }
    EffectScope.prototype.run = function (fn) {
        if (this.active) {
            var currentEffectScope = activeEffectScope;
            try {
                activeEffectScope = this;
                return fn();
            }
            finally {
                activeEffectScope = currentEffectScope;
            }
        }
        else {
            warn$2("cannot run an inactive effect scope.");
        }
    };
    /**
     * This should only be called on non-detached scopes
     * @internal
     */
    EffectScope.prototype.on = function () {
        activeEffectScope = this;
    };
    /**
     * This should only be called on non-detached scopes
     * @internal
     */
    EffectScope.prototype.off = function () {
        activeEffectScope = this.parent;
    };
    EffectScope.prototype.stop = function (fromParent) {
        if (this.active) {
            var i = void 0, l = void 0;
            for (i = 0, l = this.effects.length; i < l; i++) {
                this.effects[i].teardown();
            }
            for (i = 0, l = this.cleanups.length; i < l; i++) {
                this.cleanups[i]();
            }
            if (this.scopes) {
                for (i = 0, l = this.scopes.length; i < l; i++) {
                    this.scopes[i].stop(true);
                }
            }
            // nested scope, dereference from parent to avoid memory leaks
            if (!this.detached && this.parent && !fromParent) {
                // optimized O(1) removal
                var last = this.parent.scopes.pop();
                if (last && last !== this) {
                    this.parent.scopes[this.index] = last;
                    last.index = this.index;
                }
            }
            this.parent = undefined;
            this.active = false;
        }
    };
    return EffectScope;
}());
function effectScope(detached) {
    return new EffectScope(detached);
}
/**
 * @internal
 */
function recordEffectScope(effect, scope) {
    if (scope === void 0) { scope = activeEffectScope; }
    if (scope && scope.active) {
        scope.effects.push(effect);
    }
}
function getCurrentScope() {
    return activeEffectScope;
}
function onScopeDispose(fn) {
    if (activeEffectScope) {
        activeEffectScope.cleanups.push(fn);
    }
    else {
        warn$2("onScopeDispose() is called when there is no active effect scope" +
            " to be associated with.");
    }
}
```

# 新

## 2.window.performance

### 2.1用途监控网页性能与程序性能分析

performance.memory:内存使用情况

performance.navigation:重定向次数以及进入方式

performance.timing:各种时间

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717141446763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0ODE0MDU=,size_16,color_FFFFFF,t_70)

performance.getEntries() :所有http请求

performance.now() :距离页面初始化的时间戳，精度百万分之一，不会阻塞

performance.mark()：通过打标记计时

## 3.undefined与void 0

void是一元运算符，表示执行某个表达式但没有返回值

正常情况：undefined === void 0

由于undefined在低版本IE以及在现代浏览器局部作用域会被修改，所以由于安全问题，有时会使用void 0代替undefined

```js
//window.undefined在局部作用域中是可以被修改的
    var testUndefined = function () {
        var obj = {}
        var undefined = 'underscore'
        var window = {
            'undefined': 'bb'
        }
        console.log(window)
        console.log(undefined)
        console.log(window.undefined)
        console.log(obj.name === undefined)
        console.log(obj.name === window.undefined)
        console.log(obj.name === (void 0))
    }
    testUndefined()

```

注：使用void运算符获得的结果始终为undefined(let result = void expression中的result始终为undefined)

# vue-main源码位置解析：

## 1.核心配置文件

src/core/config.ts

```js
performance: true // 开启性能记录(window.performance)
```

# Vue2的缺陷：

1.通过Object.defineProperty去定义响应式，初始化时需要遍历每个key，造成性能损耗

2.Object.defineProperty是在初始化的时候做的，所以后续添加到data里面的新属性将不再有响应式

3.vue3解决方案(通过代理对象解决了问题1，2)

```js
new Proxy(target,{  // target 为组件的 data 返回的对象
  get(target, key){},
  set(target, key, value){}
})
```

