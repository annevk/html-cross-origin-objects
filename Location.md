# Modifications to the `Location` object

## Remove the existing security section for `Location`

As stated there, it is bogus.

## Implement IDL's "perform a security check"

When perform a security check is invoked, with a _platformObject_, _realm_, _identifier_, and _type_, run these steps:

1. If _platformObject_ has a [[crossOriginProperties]\] slot, then:

  1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

    1. If SameValue(_e_.[[property]\], _identifier_) is true, then:

      1. If _type_ is "method" and _e_ has neither [[get]\] nor [[get]\], return.

      2. Otherwise, if _type_ is "getter" and _e_.[[get]\] is true, return.

      3. Otherwise, if _type_ is "setter" and _e_.[[set]\] is true, return.

1. If _platformObject_'s global object's effective script origin is not same origin with _realm_'s global object's effective script origin, throw a TypeError.

Note: The _realm_ passed in is equal to "the current Realm" concept defined by ECMAScript. We should probably refactor this to use the latter and simplify IDL and this algorithm in the process.

## Return the correct `Location` object

Wherever `Location` objects are returned, we need to make sure to return the correct one. Basically, each `Document` has one or more `Location` objects associated with it, each created in a different Realm. Whenever a `Location` object for a `Document` is requested, if the `Document`'s effective script origin is not the same as entry settings object's effective script origin, the one returned needs to have been created in the entry settings object's Realm, otherwise it can be the default one for the `Document` object.

Note: this should automatically make the LocationIsSameOrigin abstract operation consider it cross-origin if it came from a `Document` object with another origin.

## Get rid of `[Unforgeable]`

Rather than being non-configurable, we want `Location` objects to appear configurable, but not actually be configurable. The rationale is that `Location` objects can go from being same-origin to cross-origin at which point a number of properties disappear and the identities of those that remain change. However, we do not actually want properties that appear in IDL to be configurable in either the same-origin or cross-origin case. (The intention is to preserve the internal method invariants.)

This is done by changing [[GetOwnProperty]\] and [[DefineOwnProperty]\].

## Add security check to existing member definitions

For every member other than `href`'s setter and `replace()`, add this step at the beginning:

1. If this `Location` object's relevant `Document`'s effective script origin is not the same as entry settings object's effective script origin, throw a `SecurityError` exception.

## New internal slots

A `Location` object has a [[crossOriginProperties]\] slot which is a List consisting of { [[property]\]: "href", [[get]\]: false, [[set]\]: true } and { [[property]\]: "replace" }.

A `Location` object has an [[crossOriginPropertyDescriptorMap]\] slot which is a map.

User agents should allow a value held in the map to be garbage collected along with its corresponding key when nothing holds a reference to any part of the value. I.e., as long as garbage collection is not observable.

E.g., with `const href = Object.getOwnPropertyDescriptor(crossOriginLocation, "href").set` the value and its corresponding key in the map cannot be garbage collected as that would be observable.

User agents may have an optimization whereby they remove key-value pairs from the map when `document.domain` is set. This is not observable as `document.domain` cannot revisit an earlier value.

## Internal method overrides

This might need a corresponding change to IDL that makes it okay for internal methods to be overridden (using ECMAScript prose).

### [[GetPrototypeOf]\] ( )

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[GetPrototypeOf]\], this).

1. Return null.

#### LocationIsSameOrigin(_O_)

1. Let _internalOrigin_ be the effective script origin of the _O_'s Realm's global object.

2. Let _externalOrigin_ be the effective script origin of _O_'s associated Document.

3. Return _internalOrigin_ same origin _externalOrigin_.

#### DefaultInternalMethod(_internalMethod_, _O_, _arguments_...)

1. Return the result of calling the default ordinary object _internalMethod_ internal method on _O_ passing _arguments_ as the arguments.

### [[SetPrototypeOf]\] (_V_)

1. Return false.

### [[IsExtensible]\] ( )

1. Return true.

### [[PreventExtensions]\] ( )

1. Return false.

### [[GetOwnProperty]\] (_P_)

1. If LocationIsSameOrigin(this), then:

  1. Let _desc_ be DefaultInternalMethod([[GetOwnProperty]\], this, _P_).

  1. If IDLDefined(this, _P_), then set _desc_.[[Configurable]\] to true.

  1. Return _desc_.

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. If SameValue(_e_.[[property]\], _P_) is true, then:

    1. Let _crossOriginKey_ be a tuple consisting of the current Realm's global object's effective script origin, this' associated `Document`'s effective script origin, and _P_.

    1. If this@[[crossOriginPropertyDescriptorMap]\] has _crossOriginKey_, return the value corresponding to _crossOriginKey_ in this@[[crossOriginPropertyDescriptorMap]\].

    1. Let _originalDesc_ be DefaultInternalMethod([[GetOwnProperty]\], this, _P_).

    1. Let _crossOriginDesc_ be CrossOriginPropertyDescriptor(_e_, _originalDesc_).

    1. Append key _crossOriginKey_ with its corresponding value _crossOriginDesc_ to this@[[crossOriginPropertyDescriptorMap]\].

    1. Return _crossOriginDesc_.

1. Throw a TypeError exception.

#### IDLDefined(_idlObject_, _property_)

This operation needs to be defined by IDL, see [IDL bug 29376](https://www.w3.org/Bugs/Public/show_bug.cgi?id=29376).

#### CrossOriginPropertyDescriptor (_crossOriginProperty_, _originalDesc_)

1. If _crossOriginProperty_.[[get]\] and _crossOriginProperty_.[[set]\] are absent, then:

  1. Return PropertyDescriptor{ [[Value]]: CrossOriginFunctionWrapper(true, _crossOriginFunction_), [[Enumerable]]: true, [[Writable]]: false, [[Configurable]]: true }.

1. Otherwise, then:

  1. Let _crossOriginGet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[get]\], _originalDesc_.[[Get]\]).

  1. Let _crossOriginSet_ be CrossOriginFunctionWrapper(_crossOriginProperty_.[[set]\], _originalDesc_.[[Set]\]).

  1. Return PropertyDescriptor{ [[Get]]: _crossOriginGet_, [[Set]]: _crossOriginSet_, [[Enumerable]]: true, [[Configurable]]: true }.

#### CrossOriginFunctionWrapper (_needsWrapping_, _functionToWrap_)

1. If _needsWrapping_ is false, return undefined.

1. Return a new cross-origin wrapper function whose [[Wrapped]\] internal slot is _functionToWrap_.

#### Cross-origin Wrapper Functions

A cross-origin wrapper function is an anonymous built-in function that has a [[Wrapped]\] internal slot.

When a cross-origin wrapper function _F_ is called with a list of arguments _argumentsList_, the following steps are taken:

1. Assert: _F_ has a [[Wrapped]\] internal slot whose value is a function.

1. Let _wrappedFunction_ be the value of _F_'s [[Wrapped]\] internal slot.

1. Return Call(_wrappedFunction_, this, _argumentsList_)

Note: due to this being invoked from a cross-origin context, a cross-origin wrapper function will have a different value for `Function.prototype` from the function being wrapped. This follows from how ECMAScript creates anonymous built-in functions.

### [[DefineOwnProperty]\] (_P_, _Desc_)

1. If LocationIsSameOrigin(this), then:

  1. If IDLDefined(this, _P_), return false.

  1. Return DefaultInternalMethod([[DefineOwnProperty]\], this, _P_, _Desc_).

1. Return false.

### [[HasProperty]\] (_P_)

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[HasProperty]\], this, _P_).

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. If SameValue(_e_.[[property]\], _P_) is true, return true.

1. Throw a TypeError exception.

### [[Get]\] (_P_, _Receiver_)

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[Get]\], this, _P_, _Receiver_).

1. Let _desc_ be this.[[GetOwnProperty]\](_P_).

1. If IsDataDescriptor(_desc_) is true, return _desc_.[[Value]\].

1. Throw a TypeError exception.

Note: "href" can only be set when not LocationIsSameOrigin(this).

### [[Set]\] (_P_, _V_, _Receiver_)

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[Set]\], this, _P_, _Receiver_).

1. Let _desc_ be this.[[GetOwnProperty]\](_P_).

1. If IsAccessorDescriptor(_desc_) is true, then:

  1. Call(_desc_.[[Set]\], _Receiver_, «_V_»).

  1. Return true.

1. Return false.

### [[Delete]\] (_P_)

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[Delete]\], this, _P_).

1. Return false.

### [[Enumerate]\] ( )

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[Enumerate]\], this).

1. Return CreateListIterator(« »).

### [[OwnPropertyKeys]\] ( )

1. If LocationIsSameOrigin(this), then return DefaultInternalMethod([[OwnPropertyKeys]\], this).

1. Let _keys_ be a new empty List.

1. Repeat for each _e_ that is an element of this@[[crossOriginProperties]\]:

  1. Add _e_.[[property]\] as the last element of _keys_.

1. Return _keys_.
