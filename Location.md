# Location Objects

A location object is an exotic object. It has an internal slot [[crossOriginProperties]\] which is a List consisting of { [[property]\]: "href", [[get]\]: false, [[set]\]: true } and { [[property]\]: "replace" }.

A location object has an internal slot [[crossOriginPropertyDescriptorWeakMap]\] which is a WeakMap object.

## Location[DONE\] (_DONE_...)

See HTML.

## get Location[DONE]

See HTML.

## set Location[DONE]

See HTML.

## [[GetPrototypeOf]\] ( )

1. If "same-origin", then return DefaultInternalMethod([[GetPrototypeOf]\], this).

1. Return null.

### DefaultInternalMethod(_internalMethod_, _O_, _arguments_...)

1. Return the result of calling the default ordinary object _internalMethod_ internal method on _O_ passing _arguments_ as the arguments.

## [[SetPrototypeOf]\] (_V_)

1. Throw a TypeError exception.

## [[IsExtensible]\] ( )

1. Return true.

## [[PreventExtensions]\] ( )

1. Return false.

## [[GetOwnProperty]\] (_P_)

1. If "same-origin", then return DefaultInternalMethod([[GetOwnProperty]\], this, _P_).

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. If SameValue(_e_.[[property]\], _P_) is true, then:

    1. Let _crossOriginKey_ be TODO.

    1. If this@[[crossOriginPropertyDescriptorWeakMap]\].has(_crossOriginKey_) is true, return this@[[crossOriginPropertyDescriptorWeakMap]\].get(_crossOriginKey_).

    1. Let _originalDesc_ be DefaultInternalMethod([[GetOwnProperty]\], this, _P_).

    1. Let _crossOriginDesc_ be CrossOriginPropertyDescriptor(_e_, _originalDesc_).

    1. this@[[crossOriginPropertyDescriptorWeakMap]\].set(_crossOriginKey_, _crossOriginDesc_).

    1. Return _crossOriginDesc_.

1. Throw a TypeError exception.

### CrossOriginPropertyDescriptor (_crossOriginProperty_, _originalDesc_)

1. If _crossOriginProperty_.[[get]\] and _crossOriginProperty_.[[set]\] are absent, then:

  1. Return PropertyDescriptor{ [[Value]]: CrossOriginFunctionWrapper(true, _crossOriginFunction_), [[Enumerable]]: true, [[Writable]]: false, [[Configurable]]: false }.

1. Otherwise, then:

  1. Let _crossOriginGet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[get]\], _originalDesc_.[[Get]\]).

  1. Let _crossOriginSet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[set]\], _originalDesc_.[[Set]\]).

  1. Return PropertyDescriptor{ [[Get]]: _crossOriginGet_, [[Set]]: _crossOriginSet_, [[Enumerable]]: true, [[Configurable]]: false }.

### CrossOriginFunctionWrapper (_needsWrapping_, _functionToWrap_)

1. If _needsWrapping_ is false, return undefined.

1. Return a new cross-origin wrapper function whose [[Wrapped]\] internal slot is _functionToWrap_.

### Cross-origin Wrapper Functions

A cross-origin wrapper function is an anonymous built-in function that has a [[Wrapped]\] internal slot.

When a cross-origin wrapper function _F_ is called with a list of arguments _argumentsList_, the following steps are taken:

1. Assert: _F_ has a [[Wrapped]\] internal slot whose value is a function.

1. Let _wrappedFunction_ be the value of _F_'s [[Wrapped]\] internal slot.

1. Return Call(_wrappedFunction_, this, _argumentsList_)

Note: due to this being invoked from a cross-origin context, a cross-origin wrapper function will have a different value for `Function.prototype` from the function being wrapped. This follows from how ECMAScript creates anonymous built-in functions.

## [[DefineOwnProperty]\] (_P_, _Desc_)

1. If "same-origin", then return DefaultInternalMethod([[DefineOwnProperty]\], this, _P_, _Desc_).

1. Throw a TypeError exception.

## [[HasProperty]\] (_P_)

1. If "same-origin", then return DefaultInternalMethod([[HasProperty]\], this, _P_).

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. If SameValue(_e_.[[property]\], _P_) is true, return true.

1. Throw a TypeError exception.

## [[Get]\] (_P_, _Receiver_)

1. If "same-origin", then return DefaultInternalMethod([[Get]\], this, _P_, _Receiver_).

1. Let _desc_ be this.[[GetOwnProperty]\](_P_).

1. If IsDataDescriptor(_desc_) is true, return _desc_.[[Value]\].

1. Throw a TypeError exception.

Note: "href" can only be set when not "same-origin".

## [[Set]\] (_P_, _V_, _Receiver_)

1. If "same-origin", then return DefaultInternalMethod([[Set]\], this, _P_, _Receiver_).

1. Let _desc_ be this.[[GetOwnProperty]\](_P_).

1. If IsAccessorDescriptor(_desc_) is true, then:

  1. Call(_desc_.[[Set]\], _Receiver_, «_V_»).

  1. Return true.

1. Throw a TypeError exception.

## [[Delete]\] (_P_)

1. If "same-origin", then return DefaultInternalMethod([[Delete]\], this, _P_).

1. Throw a TypeError exception.

## [[Enumerate]\] ( )

1. If "same-origin", then return DefaultInternalMethod([[Enumerate]\], this).

1. Return an empty iterator.

## [[OwnPropertyKeys]\] ( )

1. If "same-origin", then return DefaultInternalMethod([[OwnPropertyKeys]\], this).

1. Let _list_ be the empty List.

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. Append _e_.[[property]\] as the last element of _list_.

1. Return _list_.
