 # VUE-DEEP-DIVE

 - vue를 통해 프론트엔드 개발을 해왔으나 ref, reactive, computed, watch, provide 등 자주 사용하지만 어떻게 구현되었는지에 대해 깊게 생각해보지 않은 부분들에 대해 공부하기 위한 저장소입니다.
 - 모든 코드는 https://github.com/vuejs/core (v3.4.35) 기준으로 작성되었으며 큰 업데이트가 있을 경우 주기적으로 업데이트할 예정입니다.


# Reactivity API
### ref
```
export function ref(value?: unknown) {
  return createRef(value, false)
}
```
### createRef
```
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}
```
### isRef
```
export function isRef(r: any): r is Ref {
  return !!(r && r.__v_isRef === true)
}
```
### RefImpl
```
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
### toRaw
```
export function toRaw<T>(observed: T): T {
  const raw = observed && (observed as Target)[ReactiveFlags.RAW]
  return raw ? toRaw(raw) : observed
}
```