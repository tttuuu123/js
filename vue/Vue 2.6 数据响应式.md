在一个Vue实例的初始化过程中（也就是new Vue()的过程中），经过init->initState->initData，并将响应式数据data代理到实例上后，最终执行了observe方法。

observe方法内部会为对象类型且（非对象直接return）未做响应式处理的数据创建一个Observer实例。

在Observer初始化的过程中，会先为这个对象伴生一个dep订阅器实例，并将实例本身代理到对象的_ob_属性上（也就是说每个观察者对象有个_ob_属性可以获取自身，也就意味着可以通过_ob_.dep拿到自己的订阅器以及订阅器内部可能收集的订阅者）。

接着就是对数据做响应式处理，数组通过覆盖原链并且遍历再次observe每个元素；纯对象对属性的集合遍历调用defineReactive方法并再次observe这个属性的值。

最终目的是为每个对象创建一个Observer实例，这就意味着每个对象都会伴生一个dep，并且有一个_ob_的属性指向Observer实例，同时对纯对象的每个key也伴生一个dep，并对每个key用Object.defineProperty做代理拦截。

initData到这里就结束了，后面执行一些方法后，最终要执行vue的$mount方法，$mount方法内部会走到mountComponent。

mountComponent方法内部核心是：

```javascript
/* /core/instance/lifecycle.js */
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```
这也说明Vue2是为每个Vue实例创建一个Watcher（对Vue1的改进，Vue1是为响应式数据每个属性创建一个Watcher，所以在某个属性变化后能精确知道dom的哪部分要做变动，优点是不需要虚拟dom和patch比较，缺点是随着响应式数据量大后，Watcher越来越多，然后程序崩了），所以才需要虚拟dom patch比较来找出新旧节点变化，复用未变化部分，变化部分执行相应dom操作。

在new Watcher的过程中要获取当前Watcher监听的数据的值，也就是执行get方法：
```javascript
/* core/observer/watcher.js */
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

get方法内部先将Dep.target指向了这个Watcher实例，然后调用getter方法，也就是传入的updateComponent方法

```javascript
/* core/observer/watcher.js */
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

其内部实际上是先通过vm._render()获取vdom，在调用vm._update()最终生成真实dom,这部分涉及到Vue渲染，patch，这里暂且不提。</br>
在这个过程中，会去获取所有render函数内用到的变量的值，如果是一个响应式数据，就会触发响应式数据的get:

```javascript
/* core/observer/index.js */
get: function reactiveGetter() {
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
},
```

因为Dep.target指向了这个Watcher实例，所以执行了dep.depend()，要注意这个方法内部实际实行的是Watcher实例的addDep：
```javascript
/* core/observer/watcher.js */
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```
最终会在Watcher实例内存一个dep实例，同时在dep实例内存入当前的Watcher，也就是说这一步是双向收集的。

如果当前这个key的值也是个对象，即<br />
```javascript
/* core/observer/index.js */
let childOb = !shallow && observe(val);
```
这一步，childOb有值，那么会为当前这个key的值，也就是子对象的的dep实例内收集当前的Watcher。<br />

以上就是初始响应式部分的完整的逻辑，同时也可以看出Vue初始化会为所有响应式数据做好处理。。

当后续改变响应式数据时，通常会触发set方法
```javascript
/* core/observer/index.js */
val = newVal
childOb = !shallow && observe(newVal)
dep.notify()
```
先赋值，修改后的值newVal也可能是个对象，所以调用observe方法，并更新childOb的值，然后调用当前key伴生的dep去通知Watcher更新。</br>
dep的notify方法内部是调用该dep收集到的每个watcher实例的update方法，通常情况下是触发queueWatcher(watcher)，这涉及到Vue批量异步更新策略。


下面说一些零碎的：
- 在数据劫持的get方法中childOb.dep.depend()的作用：<br />
  1、响应式数据的响应式逻辑实现：<br />
    由于数组是对象，上文说过Vue的响应式处理会为每个对象创建一个Observer实例，所以可以从数组上拿到他自身的_ob_，<br />
    调用数组的7种方法，最终执行的就是_ob_.dep.notify()，而这个ob实例dep内的Watcher就是在childOb.dep.depend()这一步收集的。<br />

  2、Vue提供 修改/删除 纯对象属性/数组元素 的 set/delete方法：<br />
    如果传入的target是响应式数据，那就会调用defineReactive(target._ob_.value, key, val)，<br />
    然后执行target._ob_.dep.notify()去通知Watcher更新。<br />

- Vue 响应式的设计会生成多少个Observer实例，多少个dep实例：<br />
  只要调用了对数据调用了observe方法，并且该数据是对象，就会生成一个Observer实例，并且伴生一个dep实例。<br />
  同样的，在defineReactive方法中，会为每一个key也伴生一个dep实例。<br />
  举个粟子:<br />
  ```javascript
  data: {
    a: {
      b: 1,
      c: [],
    },
  };
  observe(data);
  ```
  首先要分清楚
  key是
  ```javascript
  data
  ```
  value是
  ```javascript
  {
    a: {
      b: 1,
      c: [],
    },
  }
  ```
  Observer构造函数内执行的是：
  ```javascript
  /* core/observer/index.js */
  this.dep = new Dep()
  if (Array.isArray(value)) {
    if (hasProto) {
      protoAugment(value, arrayMethods)
    } else {
      copyAugment(value, arrayMethods, arrayKeys)
    }
    this.observeArray(value)
  } else {
    this.walk(value)
  }
  ```
  因为这个值是对象，所以Vue会为这个值创建一个Observer实例，并且伴生一个dep实例，然后因为这个值不是数组，是个纯对象，所以Observer会对值内每个key调用defineReactive方法。<br />
  在defineReactive方法有两行核心代码是：
  ```javascript
  /* core/observer/index.js */
  const dep = new Dep();
  let childOb = !shallow && observe(val)
  ```
  说明了Vue会为纯对象值的每个key都伴生一个dep实例，然后检测这个key对应的val是不是对象，如果是，则会为这个对象创建一个Observer实例，然后继续重复上述流程。</br>
  在这个递归过程中会在defineReactive方法中碰到一个key是c的数组，数组也是对象，那么Vue为其调用observe方法后也会为其创建一个Observer实例，</br>
  在Observer的构造器中检测是数组，就先做原型替换（如果检测到不支持_proto_，就执行copyAugment，简单粗暴的在这个数组上定义改写后的7种method属性），然后执行了observeArray方法：
  ```javascript
  /* core/observer/index.js */
  observeArray(items: Array < any > ) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
  ```
  该方法即使为数组中每个对象元素继续建立响应式。</br>
  以上就是数据响应式完整的创建步骤。</br>
  综上，这个例子中，Observer实例数量有data的值，a的值和c的值合计3个；dep的实例有3个Observer伴生的加上a、b、c3个key合计6个。</br>
  当然在实际场景中，自2.4.0版本起，Vue新增了两个实例属性$attrs和$listeners，Vue的响应式中会首先对这2个key调用defineReactiv方法(这2个key的值默认是个空对象)，</br>
  所以实际上的Observer实例和dep实例还是有更多额外因素的，比如要考虑父亲节点传入的属性，以及实例本身的inheritAttrs选项的值等。

- computed是怎么实现的：</br>
  通常情况下我们是以配置对象的形式初始化Vue实例的，也就是new Vue(options)，在这个options中计算属性computed也是一个对象，</br>
  如果在实例中打印this，那么可以看到this.$options.computed中就是我们定义的计算属性。</br>
  在Vue初始化过程中会执行一步叫做initState(vm)的方法，这个方法内就是做了组件数据初始化，包括props/data/methods/computed/watch。</br>
  ```javascript
  /* core/instance/state.js */
  export function initState (vm: Component) {
    vm._watchers = []
    const opts = vm.$options
    if (opts.props) initProps(vm, opts.props)
    if (opts.methods) initMethods(vm, opts.methods)
    if (opts.data) {
      initData(vm)
    } else {
      observe(vm._data = {}, true /* asRootData */)
    }
    if (opts.computed) initComputed(vm, opts.computed)
    if (opts.watch && opts.watch !== nativeWatch) {
      initWatch(vm, opts.watch)
    }
  }
  ```
  很明显的要去看iniComputed方法：
  ```javascript
  /* core/instance/state.js 有删减 */
  const computedWatcherOptions = { lazy: true }
  function initComputed (vm: Component, computed: Object) {
    const watchers = vm._computedWatchers = Object.create(null)
    for (const key in computed) {
      const userDef = computed[key]
      const getter = typeof userDef === 'function' ? userDef : userDef.get
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
  ```
  initComputed方法中会为每个computed属性创建一个Watcher实例并保存在实例的_computedWatchers对象中，</br>
  实例参数有getter(根据计算属性本身的类型而传入函数或者对象的get方法)，还要注意计算属性的Watcher配置对象里lazy置为true。</br>
  在Watcher实例的构造函数中：
  ```javascript
  /* core/observer/watcher.js */
  this.dirty = this.lazy // for lazy watchers
  this.value = this.lazy
      ? undefined
      : this.get()
  ```
  因为lazy是true，所以Watcher实例的dirty初始化也赋值为true，并且计算属性不会在构造函数内调用get方法求值（get方法上文有讲）。</br>
  然后在defineComputed（一般Vue源码中的define开头的方法的都是把属性代理到实例上，后续就可以直接用vm.xxx来访问了）方法中核心是调用了createComputedGetter方法，</br>
  并将这个方法的返回值（也就是下文的computedGetter方法）代理到了当前实例上。
  ```javascript
  /* core/instance/state.js 有删减 */
  function createComputedGetter (key) {
    return function computedGetter () {
      const watcher = this._computedWatchers && this._computedWatchers[key]
      if (watcher) {
        if (watcher.dirty) {
          watcher.evaluate()
        }
        if (Dep.target) {
          watcher.depend()
        }
        return watcher.value
      }
    }
  }
  ```
  执行该方法实际就是执行computedGetter方法，其内部先获取了为当前计算属性创建的Watcher实例，dirty属性是初始化过程中赋值为true的，所以首先执行了evaluate方法：
  ```javascript
  /* core/observer/watcher.js */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }
  ```
  这里就是计算属性第一次求值，上文提过Watcher的get方法就是执行其传入的getter方法，</br>
  在这个过程中，会获取计算属性内每个变量的值，如果变量是做过响应式处理的，那么就会触发这个变量的get，并在这个变量伴生的dep实例内收集当前计算属性的Watcher。</br>
  执行完这一步后将dirty置为了false。</br>
  而computedGetter方法的第二步判断Dep.target是处理一些其他场景（涉及到Dep.target，则必然和pushTarget/popTarget这两个的dep.js中的方法有关，可以直接搜索这两个方法调用的地方），比如某个计算属性依赖其他计算属性的场景。</br>
  
  后续因为计算属性内响应式变量的变量伴生的dep实例内收集当前计算属性的Watcher，所以当这个变量触发set时，会通知计算属性的Watcher。</br>
  ```javascript
  /* core/observer/watcher.js */
  update () {
    if (this.lazy) {
      this.dirty = true
    }
  }
  ```
  因为计算属性的Watcher实例传入的lazy是true，所以就会把dirty重新置为true，然后就没其他操作了。</br>

  上文提到了computedGetter被代理到了实例上，所以在后续render中，当要获取计算属性值时都会调用computedGetter方法。</br>
  如果计算属性内的响应式数据值没变，那么计算属性的dirty就是false，那么就不会触发watcher.evaluate()重新求值，而是直接返回watcher.value，</br>
  反过来如果计算属性内的响应式数据值改变了，那么计算属性的dirty就置为了true，在下一次调用到计算属性触发computedGetter方法时候就会调用watcher.evaluate()重新求值。</br>

  以上就是计算属性的创建和执行过程，同时也可以注意到，Vue并不会为计算属性伴生一个dep实例。

- 自定义watch是怎么实现的：</br>
  在上面计算属性中提过Vue初始化数据都是在initState方法中，自定义watch也不列外。</br>
  ```javascript
  /* core/instance/state.js 有删减 */

  // Firefox has a "watch" function on Object.prototype...
  // export const nativeWatch = ({}).watch

  export function initState (vm: Component) {
    vm._watchers = []
    const opts = vm.$options
    if (opts.watch && opts.watch !== nativeWatch) {
      initWatch(vm, opts.watch)
    }
  }
  ```

  很明显的如果存在自定义watch，则会执行initWatch方法：

  ```javascript
  /* core/instance/state.js */
  function initWatch (vm: Component, watch: Object) {
    for (const key in watch) {
      const handler = watch[key]
      if (Array.isArray(handler)) {
        for (let i = 0; i < handler.length; i++) {
          createWatcher(vm, key, handler[i])
        }
      } else {
        createWatcher(vm, key, handler)
      }
    }
  }
  ```

  这里的handle就是watch中的key对应的值，这个值可以是方法名，或者包含选项的对象。</br>
  最终执行的是createWatcher方法：

  ```javascript
  /* core/instance/state.js */
  function createWatcher (
    vm: Component,
    expOrFn: string | Function,
    handler: any,
    options?: Object
  ) {
    if (isPlainObject(handler)) {
      options = handler
      handler = handler.handler
    }
    if (typeof handler === 'string') {
      handler = vm[handler]
    }
    return vm.$watch(expOrFn, handler, options)
  }
  ```

  createWatcher中做的事也很简单，先处理了handler是纯对象的场景（上文提到的包含选项的对象），再处理了handler是字符串的场景（上文提到的方法名）。</br>
  然后调用了vm.$watch方法，这个$watch是在Vue的初始化执行stateMixin方法中挂载在Vue的原型链的上（实际上$set，$delete也是在这个方法中挂载Vue的原型链上）。</br>

  ```javascript
  /* core/instance/state.js */
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
  ```

  $watch方法先处理了下直接用vm.$watch定义的自定义watch的cb（第二个参数）如果是纯对象的场景。</br>
  然后创建了一个Watcher实例，要注意的这个Watcher实例的options中传了user为true，同时还可能有个deep属性。</br>
  如果options中immediate属性为true，则直接执行一次。</br>
  最后返回了一个卸载方法。</br>
  如果注意到$watch中的第一个参数也就是expOrFn，它可能是个字符串也可能是个方法。</br>
  字符串的场景有：</br>
    在options中配置的watch，也就是initWatch中调用createWatcher传入的是key；</br>
    vm.$watch中可能传入的是字符串；</br>
  方法的场景只有vm.$watch才能这么定义。</br>
  
  最最重要的是字符串传入的可能是'a.b'这种键路径。</br>
  这种键路径在Watcher实例的构造方法中会调用parsePath来处理。</br>
  后续的收集依赖就和上文的响应式原理一模一样了。</br>

  到这里就可以带出另外一个问题，给定一个Vue实例，会创建多少个Watcher，</br>
  首先每个组件会对应一个Watcher，</br>
  接下来每个计算属性会对应一个，</br>
  最后就是每个自定义watch会对应一个。</br>
  实际上最快的统计方式是还是回到initState方法中，直接在vm上定义了`vm._watchers = []`，</br>
  后续每当创建Watcher实例，在构造方法中都会执行`vm._watchers.push(this)`，</br>
  所以打印一下这个`vm._watchers = []`的长度就可以知道这个Vue的实例内部有多少个Watcher实例了。

- Vue2 响应式设计的问题：</br>
  Vue的响应式过程是对options中的data中的数据在初始化过程中递归的全部做了响应式处理，开销较大。</br>
  纯对象新增或删除属性，数组通过下标改变元素无法监听。</br>
  数组响应式需要额外的修改原型链，变更数组操作的七种方法。</br>
  一些ES6新增的，例如Map、Set等无法实现响应式。</br>
  要实现响应式的增删需要学习额外的api（$set和$delete）。</br>

- Vue的响应式优化：<br />
  上文提到了Vue会对要做响应式处理的数据内每个对象，对象的key都分别做对应的响应式处理。</br>
  实际项目中很可能遇到纯展示的场景，而这些场景使用的数据是对象，那么这个对象实际上没必要做响应式处理。</br>
  那么可以对这个对象本身调用Object.freeze()方法冻结它，这样就可以Vue就只会为其创建一个Observer实例。</br>
  同时，Vue内部的空对象也是这么定义的：
  ```javascript
  export const emptyObject = Object.freeze({})
  ```




  