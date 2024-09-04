 # VUE-DEEP-DIVE

 - vue를 통해 프론트엔드 개발을 해왔으나 ref, reactive, computed, watch, provide 등 자주 사용하지만 어떻게 구현되었는지에 대해 깊게 생각해보지 않은 부분들에 대해 공부하기 위한 저장소입니다.
 - 모든 코드는 https://github.com/vuejs/core (v3.4.35) 기준으로 작성되었으며 큰 업데이트가 있을 경우 주기적으로 업데이트할 예정입니다.


# Reactivity API
### ref
```ts
export function ref(value?: unknown) {
  return createRef(value, false)
}
```

### createRef
```ts
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
} 
```
- `rawValue`가 이미 `Ref` 타입의 값일 경우에는 `rawValue`를 그대로 return 한다.  <br/> 그렇지 않다면 `RefImpl` class를 통해 초기화된 새로운 ref 객체를 return 한다.

### isRef
```ts
export function isRef(r: any): r is Ref {
  return !!(r && r.__v_isRef === true)
}
```
- Vue에서는 `ref`를 생성할 때 내부적으로 `__v_isRef`라는 키를 사용하여 값이 `ref`인지 여부를 관리한다. `isRef` 함수는 이 키에 접근하여, 인자로 받은 `r`이 `ref`인지 판단한다.

### RefImpl
```ts
class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  public readonly __v_isRef = true

  constructor(
    value: T,
    public readonly __v_isShallow: boolean,
  ) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value = __v_isShallow ? value : toReactive(value)
  }

  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {
    const useDirectValue =
      this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) {
      const oldVal = this._rawValue
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      triggerRefValue(this, DirtyLevels.Dirty, newVal, oldVal)
    }
  }
}
```
- ref값에 접근할 때 trackRefValue를 통해 의존성 등록이 필요한 경우 등록한다.
- set을 통해 해당 값이 변경되면 triggerRefValue를 통해 반응형 업데이트, watchEffect, watch 로직 실행 등의 동작을 한다.

### trackRefValue
```ts
export function trackRefValue(ref: RefBase<any>) {
  if (shouldTrack && activeEffect) {
    ref = toRaw(ref)
    trackEffect(
      activeEffect,
      (ref.dep ??= createDep(
        () => (ref.dep = undefined),
        ref instanceof ComputedRefImpl ? ref : undefined,
      )),
      __DEV__
        ? {
            target: ref,
            type: TrackOpTypes.GET,
            key: 'value',
          }
        : void 0,
    )
  }
}
```
- ref 값의 .value에 접근했을 때(get) `watch`, `watchEffect`, `computed` 등 `trigger`를 위한 의존성 등록이 필요한 경우 의존성 등록을 한다. console.log(.value) 등 값이 변경되었을 때 trigger가 필요하지 않은 경우에는 따로 의존성을 등록하지 않는다.

### trackEffect
```ts
export function trackEffect(
  effect: ReactiveEffect,
  dep: Dep,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo,
) {
  if (dep.get(effect) !== effect._trackId) {
    dep.set(effect, effect._trackId)
    const oldDep = effect.deps[effect._depsLength]
    if (oldDep !== dep) {
      if (oldDep) {
        cleanupDepEffect(oldDep, effect)
      }
      effect.deps[effect._depsLength++] = dep
    } else {
      effect._depsLength++
    }
    if (__DEV__) {
      // eslint-disable-next-line no-restricted-syntax
      effect.onTrack?.(extend({ effect }, debuggerEventExtraInfo!))
    }
  }
}
```
- 의존성 추가 및 해당 값이 필요없어질 경우 가비지 컬렉터에서 제거되기 위한 cleanupDepEffect 등록하는 과정

### triggerRefValue
```ts
export function triggerRefValue(
  ref: RefBase<any>,
  dirtyLevel: DirtyLevels = DirtyLevels.Dirty,
  newVal?: any,
  oldVal?: any,
) {
  ref = toRaw(ref)
  const dep = ref.dep
  if (dep) {
    triggerEffects(
      dep,
      dirtyLevel,
      __DEV__
        ? {
            target: ref,
            type: TriggerOpTypes.SET,
            key: 'value',
            newValue: newVal,
            oldValue: oldVal,
          }
        : void 0,
    )
  }
}
```

### triggerEffects
```ts
export function triggerEffects(
  dep: Dep,
  dirtyLevel: DirtyLevels,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo,
) {
  pauseScheduling() // pauseScheduling을 통해 동시에 다른 반응형 업데이트 작업이 불필요하게 여러번 업데이트 되지 않도록 하는 안전장치 (무한루프나 불필요한 반응형 업데이트 방지)
  for (const effect of dep.keys()) {
    // dep.get(effect) is very expensive, we need to calculate it lazily and reuse the result
    let tracking: boolean | undefined
    if (
      effect._dirtyLevel < dirtyLevel && // 이펙트가 실제 갱신이 필요한 이펙트인지 판단
      (tracking ??= dep.get(effect) === effect._trackId) // dep.get이 비용이 높은 연산이라 실제 갱신이 필요한 이펙트이면서 tracking이 undefined일 때만 연산이 수행되도록 하고, 해당 값이 effect._trackId와 일치하는지(올바른 종속성에 의해 트래킹되고 있는지) 판단하여 아래 작업 실행
    ) {
      effect._shouldSchedule ||= effect._dirtyLevel === DirtyLevels.NotDirty // 위 조건들로 인하여 스케줄링이 필요한(업데이트가 필요한) effect인데도 불구하고 shouldSchedule이 false면서 effect._dirtyLevel이 NotDirty인 경우 스케줄링을 설정
      effect._dirtyLevel = dirtyLevel
    }
    if (
      effect._shouldSchedule &&
      (tracking ??= dep.get(effect) === effect._trackId)
    ) {
      if (__DEV__) {
        // eslint-disable-next-line no-restricted-syntax
        effect.onTrigger?.(extend({ effect }, debuggerEventExtraInfo))
      }
      effect.trigger()
      if (
        (!effect._runnings || effect.allowRecurse) && // 이펙트가 실행중이지 않거나 재귀적으로 허용이 되는지 확인 -> 따로 체크 필요
        effect._dirtyLevel !== DirtyLevels.MaybeDirty_ComputedSideEffect
      ) {
        effect._shouldSchedule = false // 이펙트가 더 이상 스케줄링 되지 않도록 설정
        if (effect.scheduler) {
          // 이펙트가 특정 스케줄러를 가지고 있다면 queueEffectSchedulers 큐에 추가하여 별도로 관리
          queueEffectSchedulers.push(effect.scheduler)
        }
      }
    }
  }
  resetScheduling()
}
```

### toRaw
```ts
export function toRaw<T>(observed: T): T {
  const raw = observed && (observed as Target)[ReactiveFlags.RAW]
  return raw ? toRaw(raw) : observed
}
```

### toReactive
```ts
/**
 * Returns a reactive proxy of the given value (if possible).
 *
 * If the given value is not an object, the original value itself is returned.
 *
 * @param value - The value for which a reactive proxy shall be created.
 */
export const toReactive = <T extends unknown>(value: T): T =>
  isObject(value) ? reactive(value) : value
```
- 객체 형태의 값을 value로 받으면 반응성을 부여하여 반환하고 그렇지 않으면 원본 값을 반환
- vue 공식 사이트에서도 소개되지 않는데 어떠한 용도로 사용되는건지 확인 필요

### reactive
```ts
/**
 * Returns a reactive proxy of the object.
 *
 * The reactive conversion is "deep": it affects all nested properties. A
 * reactive object also deeply unwraps any properties that are refs while
 * maintaining reactivity.
 *
 * @example
 * ```js
 * const obj = reactive({ count: 0 })
 * ```
 *
 * @param target - The source object.
 * @see {@link https://vuejs.org/api/reactivity-core.html#reactive}
 */
export function reactive<T extends object>(target: T): Reactive<T>
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap,
  )
}
```

### isReadonly
```ts
export function isReadonly(value: unknown): boolean {
  return !!(value && (value as Target)[ReactiveFlags.IS_READONLY])
}
```

### createReactiveObject
```ts
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>,
) {
  if (!isObject(target)) {
    if (__DEV__) {
      warn(
        `value cannot be made ${isReadonly ? 'readonly' : 'reactive'}: ${String(
          target,
        )}`,
      )
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // target already has corresponding Proxy
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // only specific value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers,
  )
  proxyMap.set(target, proxy)
  return proxy
}
```
