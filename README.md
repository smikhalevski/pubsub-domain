## Contents

1. [API](#api)
  1. <a href="#event-dispatcher"><code>class <b>EventDispatcher</b></code></a>
    1. [`addEventListener`](#add-event-listener)
    2. [`removeEventListener`](#remove-event-listener)
    3. [`dispatchEvent`](#dispatch-event)
    4. [`transaction`](#transaction)
  2. <a href="#model"><code>class <b>Model</b> extends EventDispatcher</code></a>
    1. [`Model.attributesKey`](#model.attributes-key)
    2. [`attributes`](#attributes)
    3. [`new Model`](#model.constructor)
    4. [`getUniqueId`](#model_get-unique-id)
    4. [`getId`](#get-id)
    5. [`update`](#model_update)
  3. <a href="#descriptor"><code>interface <b>Descriptor</b></code></a>
    1. [`get`](#descriptor_get)
    2. [`set`](#descriptor_set)
    3. [`default`](#default)
    4. [`serializable`](#serializable)
    5. [`constant`](#constant)
    6. [`required`](#required)
  4. <a href="#list"><code>class <b>List</b></code></a>
    1. [`List.of`](#list.of)
    2. [`new List`](#list.constructor)
    3. [`[]`](#list_brackets)
    4. [`length`](#length)
    5. [`valueOf`](#list_value-of)

## API

### <a name="event-dispatcher"></a>`class EventDispatcher`

#### <a name="add-event-listener"></a><code><i>void</i> addEventListener (<i>class</i> eventClass ..., <i>function</i> listener)</code>

Registers an event listener of a specific event class. Listener receives a notification when an instance of event of the specified class is being dispatched. `instanceof` is used to detect which events should be called.

At invocation time `this` in `listener` is set to dispatching `EventDispatcher` instance.

```javascript
class FooEvent {
  construtor(message) {
    this.message = message;
  }
}

class BazEvent extends FooEvent {}

let dispatcher = new EventDispatcher;

dispatcher.addEventListener(FooEvent, event => console.log('Foo said ' + event.message));
dispatcher.addEventListener(BazEvent, event => console.log('Baz said ' + event.message));

dispatcher.dispatchEvent(new FooEvent('hello!')); // → Foo said hello!

dispatcher.dispatchEvent(new BazEvent('woops!'));
// → Foo said woops!
// → Baz said woops!
```

#### <a name="remove-event-listener"></a><code><i>void</i> removeEventListener ([<i>class</i> eventClass ...], <i>function</i> listener)</code>

Removes an event listener from dispatcher. If no event classes are provided, listener is removed for all event classes registered for object at runtime.

#### <a name="dispatch-event"></a><code><i>void</i> dispatchEvent (<i>object</i> event)</code>

Dispatches an event notifying listeners appropriate for provided event. If event has no property `target` defined then dispather assigns itself to `target`.

```javascript
let dispatcher = new EventDispatcher;
dispather.phrase = 'Le bro dispatcher!';

dispatcher.addEventListener(Object, event => console.log(event.target.phrase));

dispatcher.dispatchEvent({}); // → Le bro dispatcher!
```

#### <a name="transaction"></a><code><i>void</i> transaction (<i>function</i> callback)</code>

Invokes `callback` in transaction causing events dispatched by this dispatcher to wait until callback _successfully_ finishes.

```javascript
let dispatcher = new EventDispatcher;

dispatcher.addEventListener(Object, event => console.log(event.message));

dispatcher.transaction(() => {
  dispatcher.dispatchEvent({message: 'Transactions'});
  dispatcher.dispatchEvent({message: 'are cool!'});
});
// → Transactions
// → are cool!
```

### <a name="model"></a><code>class Model extends <a href="#event-dispatcher">EventDispatcher</a></code>

#### <a name="model.attributes-key"></a><code><i>string</i> Model.attributesKey</code>

Symbol or string representing key of static field where `Model` constructor should search for [attribute descriptors](#descriptor). Be default is set to `"attributes"`. If you want to change this key, do this before any model is instantiated.

```javascript
class SandwichModel extends Model {
  static blahblah = {
    needsBread: {default: true}
  };
}

Model.attributesKey = 'blahblah'; // Change before instantiation

let sandwich = new SandwichModel;
console.log(sandwich.needsBread) // → true
```

#### <a name="attributes"></a><code><i>object.&lt;string, <a href="#descriptor">Descriptor</a>&gt;</i> [<a href="model.attributeskey">@@attributesKey</a>]</code>

Optional definition of descriptors for a particular model. Read more [about attribute descriptors below](#descriptor).

```javascript
class CarModel extends Model {
  static attributes = {
    brand: {}
  };
}
```

This snippet describes `CarModel` class that has single attribute `brand`. When value of that attribute is changed by assignment operator an instance of `ChangeEvent` is being dispatched by `CarModel`.

```javascript
let car = new CarModel;

function changeListener(event) {
  console.log(`Changed ${event.key} to ${this[event.key]}`);
}
car.addEventListener(ChangeEvent, changeListener);

car.brand = 'Porshe'; // → Changed brand to Porshe
```

Attributes are inherited and can be overridden:

```javascript
class SportsCarModel extends CarModel {
  static attributes = {
    topSpeed: {default: 200}
  };
}

let sportsCar = new SportsCarModel;
sportsCar.addEventListener(ChangeEvent, changeListener);

console.log(sportsCar.topSpeed); // → 200

sportsCar.brand = 'Porshe'; // → Changed brand to Porshe
sportsCar.topSpeed = 320; // → Changed topSpeed to 320
```

#### <a name="model.constructor"></a><code>new Model ([<i>object</i> initilas])</code>

Creates new `Model` instance updating it with values provided by optional `initials` object.

```javascript
let model = new Model({foo: 123});
console.log(model.foo) // → 123
```

#### <a name="model_get-unique-id"></a><code><i>String</i> getUniqueId ()</code>

Returns unique model identifier. Serves same purpose as [Backbone.Model.cid](http://backbonejs.org/#Model-cid).

#### <a name="get-id"></a><code><i>\*</i> getId ()</code>

Returns model identifier used to distinguish models in [`List`](#list). Serves same purpose as [Backbone.Model.cid](http://backbonejs.org/#Model-id). By default, returns `Model#id`, so if one is not defined as attribute or as a property, `undefined` is returned.

```javascript
let model = new Model();
model.id = 'abc';

console.log(model.getId()) // → abc
```

#### <a name="model_update"></a><code><i>void</i> update (<i>object</i> source)</code>

Performs deep transactional update of this model, recursively calling `update` method on stored objects if available. Transactional means that change events are dispatched when all fields from `source` are assigned to model, so listeners don't see partially updated model.

Triggers change events for regular model properties as well as for model attributes if their values change basing on `Object.is` comparison.

Non-enumerable properties of `source` are ignored during update.

```javascript
let model = new Model();

function changeListener(event) {
  console.log(`Changed ${event.key} to ${this[event.key]}`);
}
model.addEventListener(ChangeEvent, changeListener);

model.update({foo: 'bar'}); // → Changed foo to bar
```

### <a name="descriptor"></a>`interface Descriptor`

#### <a name="descriptor_get"></a><code><i>\*</i> get (<i>\*</i> storedValue)</code>

Optional attribute getter receives value that is currently being stored in model. Getter returns value that should be served to requester. By default, `get` returns `storedValue` as is.

```javascript
class UserModel extends Model {
  static attributes = {
    greeting: {
      get (storedValue) {
        return `Hello ${storedValue}!`;
      }
    }
  };
}

let user = new UserModel({greeting: 'Peter'});
console.log(user.greeting); // → Hello Peter!
```

#### <a name="descriptor_set"></a><code><i>\*</i> set (<i>\*</i> value, <i>\*</i> storedValue)</code>

Optional attribute setter receives value user inteneded to assign and value that is currently being stored in model. If `set` returns value that is not equal to `storedValue` then returned value is first stored in model and then instance of `ChangeEvent` is dispatched by model. Values are compared using `Object.is`. By default, `set` returns `value` as is.

```javascript
class FooModel extends Model {
  static attributes = {
    even: {
      set (value, storedValue) {
        return value + value % 2;
      }
    }
  };
}

let foo = new FooModel;

function changeListener(event) {
  console.log(`Even is set to ${foo.even}`);
}
foo.addEventListener(ChangeEvent, changeListener);
foo.even = 1; // → Even is set to 2

foo.even = 2; // Attribute did not change its value so no changes are dispatched
```

#### <a name="default"></a><code><i>\*</i> default = undefined</code>

Attribute default value stored in model during instantiation. [Setter](#model.set) is used to assign value. This value can be overridden by initials provided to [model constructor](#model.constructor).

You can reset attribute values back to defaults assigning `undefined`. In this case corresponding `set` would receive default value as first argument instead of `undefined`.

If attribute descriptor has only `default` and default value is `undefined`, `null`, primitive or its wrapper, function or an array you can use shorthand sintax:

```javascript
class CarModel extends Model {
  static attributes = {
    brand: 'Porshe', // Shorthand sintax, same as {default: 'Porshe'}
    speed: {default: 250}
  };
}

let car = new CarModel({speed: 300});
console.log(`${car.brand} can drive ${car.speed} km/h`); // → Porshe can drive 300 km/h

car.speed = undefined;
console.log(car.speed); // → 250
```

#### <a name="serializable"></a><code><i>boolean</i> serializable = true</code>

Boolean flag that toggles attribute enumerability.

```javascript
class UserModel extends Model {
  static attributes = {
    isActive: {
      default: false,
      serializable: false
    }
  };
}

let user = new UserModel({name: 'Johnny'});

console.log(JSON.stringify(user)); // → {"name":"Johnny"}
console.log(user.isActive); // → false
```

#### <a name="constant"></a><code><i>boolean</i> constant = false</code>

If set to `true` prevents attribute from being changed after intantiation. If attribute value was not provided as `default` or among initials then `Error` is thrown.

```javascript
class UserModel extends Model {
  static attributes = {
    userId: {constant: true}
  };
}

let user = new UserModel({userId: 512});
user.userId = 128; // → TypeError: Cannot set property which has only a getter

new UserModel; // → Error: Uninitialized constant attribute UserModel[userId]
```

#### <a name="required"></a><code><i>boolean</i> required = false</code>

Boolean flag that toggles weather attribute accepts `null` and `undefined` values or not. By default is set to `false`.

```javascript
class FooModel extends Model {
  static attributes = {
    baz: {required: true}
  };
}

let foo = new FooModel({baz: 'Okaay'});
try {
  foo.baz = undefined; // → Error: Required attribute FooModel[baz] cannot be undefined
} catch(e) {
  console.log(foo.baz); // → Okaay
}
```

### <a name="list"></a><code>class List extends <a href="#event-dispatcher">EventDispatcher</a></code>

`List` is array-like sparse ordered collection of _non unique_ models of particular type. Sparse means that list can have `undefined` values in it.

`List` mixes all methods from native `Array` including `@@iterator` so both `for of` and `for in` are available for `List` instances as well.

If any model stored in list dispatches an event, this event is dispathed by list too. Same model can be contained by multiple lists and all those lists would be notified on model events.

#### <a name="list.of"></a><code><i>List</i> of (<i>class extends <a href="model">Model</a></i> modelClass)</code>

Creates new typed list class.

For multiple invocations with the same `modelClass` the same instance of typed list constructor is returned.

```javascript
class FooModel extends Model {
  sayHo () {
    return this.ho;
  }
}

let FooList = List.of(FooModel);

let foos = new FooList([{ho: 'Yay!'}]);
foos[0].sayHo(); // → Yay!
```

You can create custom list classes with ease:

```javascript
class BetterFooList extends List.of(FooModel) {
  areYouReadyKids () {
    return this[0].sayHo();
  }
}

let betterFoos = new BetterFooList([{ho: 'Aye aye captain!'}]);
console.log(betterFoos.areYouReadyKids()); // → Aye aye captain!
```

#### <a name="list.constructor"></a><code>new List ([<i>array.&lt;object|<a href="model">Model</a>&gt;</i> models])</code>

Creates new list initiating it with provided models. All non-`undefined` items from `models` are converted to `Model` instances.

```javascript
let list = new List([{id: 1}, {id: 2}]);

console.log(list[1]); // → Model {id: 2}
```

#### <a name="list_brackets"></a><code><i>undefined|<a href="model">Model</a></i> [<i>integer</i> index]</code>

Sets of retrieves list element by index.

If value provided for assignment is not strictly equal to `undefined` then it is converted to `Model`. If provided value is already a `Model` instance then it is stored as is. In case incompatible model is provided, then new model instance of compatible type is created and populated with attributes and properties of provided model.

If assigned model is not equal to model that is currently stored by index then stored model is detached: it stops propagation of its events to list.

```javascript
let list = new List;
list[1] = {foo: 'bar'};

console.log(list[0]); // → undefined
console.log(list[1]); // → Model {foo: "bar"}

list.addEventListener(Object, event => console.log('List is notified by',${event.target}));

let model = list[1];
model.dispatchEvent({}); // → List is notified by Model {foo: "bar"}

list[1] = undefined;
model.dispatchEvent({}); // No event propagation occurs.
```

#### <a name="length"></a><code><i>integer</i> length</code>

Length of list that behaves the same way as native `Array#length`. 

```javascript
let list = new List;
list[9] = {};

console.log(list.length); // → 10

list.length = 2;
console.log(list.length); // → 2
console.log(list[9]); // → undefined
// Model that was stored at index 9 is now detached.
```

#### <a name="list_value-of"></a><code><i>array</i> valueOf</code>

Returns this list as sparse array of models. Modifications of this array do not affect list.

