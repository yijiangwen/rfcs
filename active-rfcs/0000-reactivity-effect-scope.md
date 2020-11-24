- Start Date: 2020-08-20
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: [#2195](https://github.com/vuejs/vue-next/pull/2195)

# Summary

A new `effectScope` API for `@vue/reactivity` that automatically collects the effects in a function call, and returns a `scope` instance that can be cleaned up by passing to the `stop()` API.

# Basic example

```ts
// effect, computed, watch, watchEffect created inside the scope will be collected

const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects in the scope
stop(scope)
```

# Motivation

In Vue's component `setup()`, effects will be collected and bound to the current instance. When the instance get unmounted, effects will be disposed automatically. This is a convenient and intuitive feature.

However, when we are using them out side of components or as a standalone package, it's not that simple. For example, this might be what we need to do for disposing the effects of `computed` & `watch`

```ts
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)

disposables.push(() => stop(doubled.effect))

const stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`)
})

disposables.push(stopWatch1)

const stopWatch2 = watch(doubled, () => {
  console.log(double.value)
})

disposables.push(stopWatch2)
```

And to stop the effects:

```ts
disposables.forEach(f => f())
disposables = []
```

Especially when we have some long and complex composable code, it's laborious to manually collect all the effects . It's also easy to forget collecting them (or you don't have access to effects created in the composable functions) which might result memory leakage and unexpected behavior.

This RFC is trying to abstract the component's `setup()` effect collecting and disposing feature into a more general API that can be reused out side of the component model.

It also provides the functionality to `escape` some effects from the component's `setup()` scope or user-defined scope. Resolving https://github.com/vuejs/vue-next/issues/1532.

# Detailed design

### Scope

In `effectScope`, `effect` created inside the scope will be collected into the `effects` array. A scope instance will be returned by `effectScope`.

```ts
const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})
```

When passing the scope instance into `stop()` API, it will stop all the effects and nested scopes recursively.

```ts
import { stop } from 'vue'

stop(scope)
```

### Nested Scopes

Nested scopes should also be collected by its parent scope. And when the parent scope get disposed, all its descendant scopes will be also stopped.

```ts
import { stop } from 'vue'

const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)
  
  // not need to get the stop handler, it will be collected by the outer scope
  effectScope(() => {
    watch(doubled, () => console.log(double.value))
  })

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects, as well as the nested scopes
stop(scope)
```

### Escaping Nested Scopes

An option can be passed as the second parameter of `effectScope`. Specifying `detached: true` allows nested scopes not to be collected by its parent scopes. The same behavior should apply to component's `setup()` scope.

This also makes usages like ["lazy initialization"](https://github.com/vuejs/vue-next/issues/1532) possible.

```ts
import { stop } from 'vue'

let nestedScope

const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)
  
  // with "escaped: true" set, 
  // the scope will not be collected and disposed by the outer scope
  nestedScope = effectScope(
    () => {
      watch(doubled, () => console.log(double.value))
    }, 
    { detached: true } // <--
  )

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects, except from "nestedScope"
stop(scope)

// stop the nested scope
stop(nestedScope)
```

### Forward returning values

Values returned in the scope function will be forward as the return value of `effectScope` and able to be destructured.

```ts
const scope = effectScope(() => {
  const { x } = useMouse()
  const doubled = computed(() => x.value * 2)

  return { x, doubled }
})

const { x, doubled } = scope

console.log(doubled.value)
```

The return value should be an object.

### onCleanup

A `onCleanup` hook is passed to the scope function

```ts
const scope = effectScope((onCleanup) => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))

  onCleanup(()=> {
    console.log('cleaned!')
  })
})

stop(scope) // logs 'cleaned!'
```

### Extend the Scope

A scope instance is also a callable function, which allows users to extend the scope and combine their effects.

```ts
const myScope = effectScope()

myScope(() => {
  watchEffect(/* ... */)
})

myScope(() => {
  watch(/* ... */)
})

// both watchEffect and watch will be disposed
stop(myScope)
```

The extending function will also forward its return values

```ts
const { foo } = myScope(() => {
  return {
    foo: computed(/* ... */)
  }
})
```

## Implementation

A scope instance could be defined as

```ts
interface EffectScope {
  _isEffectScope: true
  id: number
  active: boolean
  effects: (ReactiveEffect | EffectScope)[]
}
```

Similar to what we did for `currentInstance`, on effect creation, if the `activeEffectScope` is presented, push itself into the scope.

```ts
// simplified version 

function createReactiveEffect( /* ... */ ) {
  const effect = /* ... */
  
  if (activeEffectScope)
    activeEffectScope.effects.push(effect) // <-
  
  return effect
}

export function effectScope(
  fn?: () => void
) {
  /* ... */
  try {
    effectScopeStack.push(scope)
    activeEffectScope = scope
    fn()
  } finally {
    effectScopeStack.pop()
    activeEffectScope = effectScopeStack[effectScopeStack.length - 1]
  }
  return scope
}
```

### Existing Library

I have implemented a third-party library [@vue-reactivity/scope](https://github.com/vue-reactivity/scope) which aims to solve the exact same problem. However, since we have no way to get access the internal states of effects, [it has to wrap & re-export](https://github.com/vue-reactivity/scope/blob/master/src/hijack.ts#L11) the APIs (`effect` and `computed` etc.) to make the auto collecting work. This deviated from the Vue's design and can not be directly reused for Vue applications. As this is a quite low-level feature, I think it's better to be done in `@vue/reactivity` itself. And could benefit Vue itself and the ecosystem built on top of `@vue/reactivity`.

### Pull Request

A draft PR: [#2195](https://github.com/vuejs/vue-next/pull/2195).

## Impacting Vue

Currently in `@vue/runtime-dom`, [it wraps the `computed` to add the instance binding](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/apiComputed.ts). Which makes the following statements NOT equivalent

```ts
import { computed } from '@vue/reactivity'
import { computed } from 'vue'
```

This should not be an issue for most of the users, but for some libraries that would like to only rely on `@vue/reactivity` (for more flexible usages) this might be a pitfall and cause some unwanted side-effects.

With this RFC, `@vue/runtime-dom` might also be able to do some refactoring by using the `effectScope` to collect those effects transparently. Effect collecting can be more general from the lower level and the computed rewrapping will not be necessary anymore.

# Drawbacks

- It doesn't work well with async functions
- ~~Variables created in the closure will be harder to access from the outer closure (or should we accept the return value and pass it out?)~~

# Alternatives

Maybe an `onEffectCreation` hook instead, which will be called on every effect creation. This can provide more flexibility and even let users create their own scope utils or custom frameworks' lifecycles.

```ts
onEffectCreation((effect) => {
  // custom function
  
  // for example, in runtime-core, we can use this to collect the effect
  recordInstanceBoundEffect(effect)
})
```

```ts
function createReactiveEffect( /* ... */ ) {
  /* ... */
  
  invokeHook('effectCreation', effect)
  
  return effect
}
```

# Adoption strategy

This is a new API and should not affect existing code.

# Unresolved questions

None