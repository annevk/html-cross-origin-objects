# Implement IDL's "perform a security check"

When perform a security check is invoked, with a _platformObject_, _realm_, _identifier_, and _type_, run these steps:

1. If _platformObject_ is a `Location` or `Window` object, then:

  1. Repeat for each _e_ that is an element of CrossOriginProperties(_platformObject_):

    1. If SameValue(_e_.[[property]\], _identifier_) is **true**, then:

      1. If _type_ is "method" and _e_ has neither [[needsGet]\] nor [[needsGet]\], then return.

      2. Otherwise, if _type_ is "getter" and _e_.[[needsGet]\] is **true**, then return.

      3. Otherwise, if _type_ is "setter" and _e_.[[needsSet]\] is **true**, then return.

1. If _platformObject_'s global object's effective script origin is not same origin with _realm_'s global object's effective script origin, throw a **TypeError**.

Note: The _realm_ passed in is equal to "the current Realm" concept defined by ECMAScript. We should probably refactor this to use the latter and simplify IDL and this algorithm in the process.

## CrossOriginProperties( _O_ )

1. Assert: _O_ is a `Location` or `Window` object.

1. If _O_ is a `Location` object, then return « { [[property]\]: "href", [[needsGet]\]: **false**, [[needsSet]\]: **true** }, { [[property]\]: "replace" } ».

1. Let _crossOriginWindowProperties_ be « { [[property]\]: "location", [[needsGet]\]: **true**, [[needsSet]\]: **true** }, { [[property]\]: "postMessage" }, { [[property]\]: "window", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "frames", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "self", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "top", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "parent", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "opener", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "closed", [[needsGet]\]: **true**, [[needsSet]\]: **false** }, { [[property]\]: "close" }, { [[property]\]: "blur" }, { [[property]\]: "focus" } ».

1. Repeat for each _e_ that is an element of the **dynamic nested browsing context properties**:

  1. Add { [[property]\]: _e_ } as the last element of _crossOriginWindowProperties_.

1. Repeat for each _e_ that is an element of the **child browsing context name properties** that are **supported property names**:

  1. Add { [[property]\]: _e_ } as the last element of _crossOriginWindowProperties_.

1. Return _crossOriginWindowProperties_.

Note: this last line depends on https://github.com/whatwg/html/pull/544 being correct.

# `WindowProxy`

## Extensions to `Window` objects

Every `Window` object has an [[crossOriginPropertyDescriptorMap]\] internal slot which is a map.

## `WindowProxy` Exotic Objects

A _window proxy_ is an exotic object that wraps a `Window` object, indirecting most operations through to the wrapped object. Each browsing context has a window proxy. When the browsing context is navigated, the `Window` object wrapped by the browsing context's window proxy is changed.

Every window proxy object has a [[Window]] internal slot representing the wrapped window.

### [[GetPrototypeOf]\] ( )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginGetPrototypeOf(_W_).

#### CrossOriginGetPrototypeOf ( _O_ )

1. If IsWindowOrLocationSameOrigin(_O_), then return DefaultInternalMethod([[GetPrototypeOf]\], _O_).

1. Return null.

#### IsWindowOrLocationSameOrigin ( _O_ )

1. Return **true** if the effective script origin of the current Realm's global object is same origin with the effective script origin of _O_'s global object, and **false** otherwise.

#### DefaultInternalMethod ( _internalMethod_, _O_, _arguments_... )

1. Return the result of calling the default ordinary object _internalMethod_ internal method on _O_ passing _arguments_ as the arguments.

### [[SetPrototypeOf]\] ( _V_ )

1. Return **false**.

### [[IsExtensible]\] ( )

1. Return **true**.

### [[PreventExtensions]\] ( )

1. Return **false**.

### [[GetOwnProperty]\] ( _P_ )

1. Let _W_ be this.[[Window]].

1. If _P_ is an array index property name, then:

   XXX: reference IDL for "array index property name".

  1. Let _index_ be ToUint32(_P_).

  1. If _index_ is a supported property index, then return PlatformObjectGetOwnProperty(_W_, _P_, **true**).

     XXX: flatten this in a PR against HTML. See https://html.spec.whatwg.org/#dom-window-item and the paragraph before in particular.

  1. Return PropertyDescriptor{ [[Value]]: **undefined**, [[Writable]]: **false** [[Enumerable]]: **false**, [[Configurable]]: **true** }.

1. If IsWindowOrLocationSameOrigin(_W_), then return OrdinaryGetOwnProperty(_W_, _P_).

   Note: This violates ECMAScript's internal method invariants. https://bugzilla.mozilla.org/show_bug.cgi?id=1197958#c4 has further discussion on the manner. For now we document what is implemented.

1. Let _property_ be CrossOriginGetOwnPropertyHelper(_W_, _P_).

1. If _property_ is not **undefined**, return _property__.

1. If _property_ is undefined and _P_ is a child browsing context name, then ...

   XXX: define this when integrating with HTML. See https://github.com/whatwg/html/pull/544 for details.

1. Throw a **TypeError** exception.

#### CrossOriginGetOwnPropertyHelper ( _O_, _P_ )

Note: If this abstract operation returns undefined and there is no custom behavior, the caller needs to throw a **TypeError* exception.

1. If _P_ is @@toStringTag, @@hasInstance, or @@isConcatSpreadable, then return PropertyDescriptor{ [[Value]]: **undefined**, [[Writable]]: **false** [[Enumerable]]: **false**, [[Configurable]]: **true** }.

1. Let _crossOriginKey_ be a tuple consisting of the current Realm's global object's effective script origin, _O_'s global object's effective script origin, and _P_.

1. Repeat for each _e_ that is an element of CrossOriginProperties(_O_):

  1. If SameValue(_e_.[[property]\], _P_) is **true**, then:

    1. If _O_.[[crossOriginPropertyDescriptorMap]\] has _crossOriginKey_, then return the value corresponding to _crossOriginKey_ in _O_.[[crossOriginPropertyDescriptorMap]\].

    1. Let _originalDesc_ be OrdinaryGetOwnProperty(_O_, _P_).

    1. Let _crossOriginDesc_ be CrossOriginPropertyDescriptor(_e_, _originalDesc_).

    1. Append key _crossOriginKey_ with its corresponding value _crossOriginDesc_ to _O_.[[crossOriginPropertyDescriptorMap]\].

    1. Return _crossOriginDesc_.

1. Return **undefined**.

#### CrossOriginPropertyDescriptor ( _crossOriginProperty_, _originalDesc_ )

1. If _crossOriginProperty_.[[needsGet]\] and _crossOriginProperty_.[[needsSet]\] are absent, then:

  1. Let _value_ be _originalDesc_.[[Value]\].

  1. If IsCallable(_value_) is **true**, set _value_ to CrossOriginFunctionWrapper(**true**, _value_).

  1. Return PropertyDescriptor{ [[Value]]: _value_, [[Enumerable]]: **true**, [[Writable]]: **false**, [[Configurable]]: **true** }.

1. Otherwise, then:

  1. Let _crossOriginGet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[needsGet]\], _originalDesc_.[[Get]\]).

  1. Let _crossOriginSet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[needsSet]\], _originalDesc_.[[Set]\]).

  1. Return PropertyDescriptor{ [[Get]]: _crossOriginGet_, [[Set]]: _crossOriginSet_, [[Enumerable]]: **true**, [[Configurable]]: **true** }.

#### CrossOriginFunctionWrapper ( _needsWrapping_, _functionToWrap_ )

1. If _needsWrapping_ is **false**, then return undefined.

1. Return a new cross-origin wrapper function whose [[Wrapped]\] internal slot is _functionToWrap_.

#### Cross-origin Wrapper Functions

A cross-origin wrapper function is an anonymous built-in function that has a [[Wrapped]\] internal slot.

When a cross-origin wrapper function _F_ is called with a list of arguments _argumentsList_, the following steps are taken:

1. Assert: _F_ has a [[Wrapped]\] internal slot whose value is a function.

1. Let _wrappedFunction_ be the value of _F_'s [[Wrapped]\] internal slot.

1. Return ? Call(_wrappedFunction_, this, _argumentsList_).

Note: due to this being invoked from a cross-origin context, a cross-origin wrapper function will have a different value for `Function.prototype` from the function being wrapped. This follows from how ECMAScript creates anonymous built-in functions.

### [[DefineOwnProperty]\] ( _P_, _Desc_ )

1. Let _W_ be this.[[Window]].

1. If IsWindowOrLocationSameOrigin(_W_), then return ? OrdinaryDefineOwnProperty(_W_, _P_, _Desc_).

   Note: See above about how this violates ECMAScript's internal method invariants.

1. Return **false**.

### [[HasProperty]\] ( _P_ )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginHasProperty(_W_, _P_).

#### CrossOriginHasProperty ( _O_, _P_ )

1. If IsWindowOrLocationSameOrigin(_O_), then return ? OrdinaryHasProperty(_O_, _P_).

1. Repeat for each _e_ that is an element of CrossOriginProperties(_O_):

  1. If SameValue(_e_.[[property]\], _P_) is **true**, then return **true**.

1. Throw a **TypeError** exception.

### [[Get]\] ( _P_, _Receiver_ )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginGet(this, _W_, _P_, _Receiver_).

#### CrossOriginGet ( _O_, _proxyO_, _P_, _Receiver_ )

1. If IsWindowOrLocationSameOrigin(_proxyO_), then return DefaultInternalMethod([[Get]\], _proxyO_, _P_, _Receiver_).

1. Let _desc_ be _O_.[[GetOwnProperty]\](_P_).

1. If IsDataDescriptor(_desc_) is **true**, then return _desc_.[[Value]\].

1. Otherwise, IsAccessorDescriptor(_desc_) must be **true** so, let _getter_ be _desc_.[[Get]\].

1. If _getter_ is not **undefined**, return ? Call(_getter_, _Receiver_).

1. Throw a **TypeError** exception.

### [[Set]\] ( _P_, _V_, _Receiver_ )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginSet(this, _W_, _P_, _V_, _Receiver_).

#### CrossOriginSet ( _O_, _proxyO_, _P_, _V_, _Receiver_ )

1. If IsWindowOrLocationSameOrigin( _proxyO_ ), then return DefaultInternalMethod([[Set]\], _proxyO_, _P_, _Receiver_).

1. Let _desc_ be _O_.[[GetOwnProperty]\](_P_).

1. If IsAccessorDescriptor(_desc_) is **true**, then:

  1. Let _setter_ be _desc_.[[Set]\].

  1. If _setter_ is **undefined**, return **false**.

  1. Perform ? Call(_setter_, _Receiver_, «_V_»).

  1. Return **true**.

1. Return **false**.

### [[Delete]\] ( _P_ )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginDelete(_W_, _P_).

#### CrossOriginDelete ( _O_, _P_ )

1. If IsWindowOrLocationSameOrigin(_O_), then return DefaultInternalMethod([[Delete]\], _O_, _P_).

1. Return **false**.

### [[Enumerate]\] ( )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginEnumerate(_W_).

#### CrossOriginEnumerate( _O_ )

1. If IsWindowOrLocationSameOrigin(_O_), then return DefaultInternalMethod([[Enumerate]\], _O_).

1. Return CreateListIterator(« »).

### [[OwnPropertyKeys]\] ( )

1. Let _W_ be this.[[Window]].

1. Return CrossOriginOwnPropertyKeys(_W_).

#### CrossOriginOwnPropertyKeys ( _O_ )

1. If IsWindowOrLocationSameOrigin(_O_), then return DefaultInternalMethod([[OwnPropertyKeys]\], _O_).

1. Let _keys_ be a new empty List.

1. Repeat for each _e_ that is an element of CrossOriginProperties(_O_):

  1. Add _e_.[[property]\] as the last element of _keys_.

1. Return _keys_.

# Modifications to the `Location` object

## Remove the existing security section for `Location`

As stated there, it is bogus.

## Move `[Unforgeable]` from the interface to each property

We'll inline the bits specific to the interface. We'll also lie about being non-configurable to not break internal method invariants.

## Add security check to existing member definitions

For every member other than `href`'s setter and `replace()`, add this step at the beginning:

1. If this `Location` object's relevant `Document`'s effective script origin is not the same as entry settings object's effective script origin, throw a `SecurityError` exception.

## New internal slots

A `Location` object has a [[defaultProperties]\] slot which is an empty List.

A `Location` object has an [[crossOriginPropertyDescriptorMap]\] slot which is an empty map.

User agents should allow a value held in the map to be garbage collected along with its corresponding key when nothing holds a reference to any part of the value. I.e., as long as garbage collection is not observable.

E.g., with `const href = Object.getOwnPropertyDescriptor(crossOriginLocation, "href").set` the value and its corresponding key in the map cannot be garbage collected as that would be observable.

User agents may have an optimization whereby they remove key-value pairs from the map when `document.domain` is set. This is not observable as `document.domain` cannot revisit an earlier value.

## Location creation

To create a Location object, run these steps:

1. Let _location_ be a new `Location` platform object.

   XXX: reference IDL for "platform object".

1. Perform ! _location_.[[DefineOwnProperty]\]("toString", { [[Value]\]: %ObjProto_toString%, [[Writable]\]: **false**, [[Enumerable]\]: **false**, [[Configurable]\]: **false** }).

1. Perform ! _location_.[[DefineOwnProperty]\]("toJSON", { [[Value]\]: **undefined**, [[Writable]\]: **false**, [[Enumerable]\]: **false**, [[Configurable]\]: **false** }).

1. Let _valueOfFunction_ be a Function object whose behavior is as follows:

   1. Return ? ToObject(**this** value).

   The value of the Function object's "length" property is the Number value 0.

   The value of the Function object’s "name" property is the String value "valueOf".

1. Perform ! _location_.[[DefineOwnProperty]\]("valueOf", { [[Value]\]: _valueOfFunction_, [[Writable]\]: **false**, [[Enumerable]\]: **false**, [[Configurable]\]: **false** }).

1. Perform ! _location_.[[DefineOwnProperty]\](@@toPrimitive, { [[Value]\]: **undefined**, [[Writable]\]: **false**, [[Enumerable]\]: **false**, [[Configurable]\]: **false** }).

1. Set _location_.[[defaultProperties]\] to _location_.[[OwnPropertyKeys]\]().

1. Return _location_.

## Internal method overrides

This might need a corresponding change to IDL that makes it okay for internal methods to be overridden (using ECMAScript prose).

### [[GetPrototypeOf]\] ( )

1. Return CrossOriginGetPrototypeOf(this).

### [[SetPrototypeOf]\] ( _V_ )

1. Return **false**.

### [[IsExtensible]\] ( )

1. Return **true**.

### [[PreventExtensions]\] ( )

1. Return **false**.

### [[GetOwnProperty]\] ( _P_ )

1. If IsWindowOrLocationSameOrigin(this), then:

  1. Let _desc_ be OrdinaryGetOwnProperty(this, _P_).

  1. If IsLocationDefaultProperty(this, _P_), then set _desc_.[[Configurable]\] to **true**.

  1. Return _desc_.

1. Let _property_ be CrossOriginGetOwnProperty(this, _P_).

1. If _property_ is not **undefined**, return _property_.

1. Throw a **TypeError** exception.

#### IsLocationDefaultProperty ( _O_, _P_ )

1. Repeat for each _e_ that is an element of _O_.[[defaultProperties]\]:

  1. If _e_ is _P_, then return **true**.

1. Return **false**.

### [[DefineOwnProperty]\] ( _P_, _Desc_ )

1. If IsWindowOrLocationSameOrigin(this), then:

  1. If IsLocationDefaultProperty(this, _P_), then return **false**.

  1. Return ? OrdinaryDefineOwnProperty(this, _P_, _Desc_).

1. Return **false**.

### [[HasProperty]\] ( _P_ )

1. Return CrossOriginHasProperty(this, _P_).

### [[Get]\] ( _P_, _Receiver_ )

1. Return CrossOriginGet(this, this, _P_, _Receiver_).

### [[Set]\] ( _P_, _V_, _Receiver_ )

1. Return CrossOriginSet(this, this, _P_, _V_, _Receiver_).

### [[Delete]\] ( _P_ )

1. Return CrossOriginDelete(this, _P_).

### [[Enumerate]\] ( )

1. Return CrossOriginEnumerate(this).

### [[OwnPropertyKeys]\] ( )

1. Return CrossOriginOwnPropertyKeys(this).
