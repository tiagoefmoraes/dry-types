# v0.13 to be released

## Changed

* [BREAKING] Renamed `Types::Form` to `Types::Params`. You can opt-in the former name with `require 'dry/types/compat/form_types'`. It will be dropped in the next release (ndrluis)
* [BREAKING] The `Int` types was renamed to `Integer`, this was the only type named differently from the standard Ruby classes so it has been made consistent. The former name is available with `require 'dry/types/compat/int'` (GustavoCaso + flash-gordon)
* [BREAKING] Default types are not evaluated on `nil`. Default values are evaluated _only_ if no value were given.
  ```ruby
    type = Types::Strict::String.default("hello")
    type[nil] # => constraint error
    type[] # => "hello"
  ```
  This change allowed to greatly simplify hash schemas, make them a lot more flexible yet predictable (see below).
* [BREAKING] `Dry::Types.register_class` was removed, `Dry::Types.register` was made private API, do not register your types in the global `dry-types` container, use a module instead, e.g. `Types` (flash-gordon)


## Added

* Hash schemas were rewritten. The old API is still around but is going to be deprecated and removed before 1.0. The new API is simpler and more flexible. Instead of having a bunch of predefined schemas you can build your own by combining the following methods:

  1. `Schema#with_key_transform`—transforms keys of input hashes, for things like symbolizing etc.
  2. `Schema#strict`—makes a schema intolerant to unknown keys.
  3. `Hash#with_type_transform`—transforms member types with an arbitrary block. For instance,

    ```ruby
    optional_keys = Types::Hash.with_type_transform { |t, _key| t.optional }
    schema = optional_keys.schema(name: 'strict.string', age: 'strict.int')
    schema.(name: "Jane", age: nil) # => {name: "Jane", age: nil}
    ```

  Note that by default all keys are required, if a key is expected to absent, add to the corresponding type's meta `omittable: true`:

  ```ruby
  intolerant = Types::Hash.schema(name: Types::Strict::String)
  intolerant[{}] # => Dry::Types::MissingKeyError
  tolerant = Types::Hash.schema(name: Types::Strict::String.meta(omittable: true))
  tolerant[{}] # => {}
  tolerant_with_default = Types::Hash.schema(name: Types::Strict::String.meta(omittable: true).default("John"))
  tolerant[{}] # => {name: "John"}
  ```

  The new API is composable in a natural way:

  ```ruby
  TOLERANT = Types::Hash.with_type_transform { |t| t.meta(omittable: true) }.freeze
  user = TOLERANT.schema(name: 'strict.string', age: 'strict.int')
  user.(name: "Jane") # => {name: "Jane"}

  TOLERANT_SYMBOLIZED = TOLERANT.with_key_transform(&:to_sym)
  user_sym = TOLERANT_SYMBOLIZED.schema(name: 'strict.string', age: 'strict.int')
  user_sym.("name" => "Jane") # => {name: "Jane"}
  ```

  (flash-gordon)

* `Types.Strict` is an alias for `Types.Instance` (flash-gordon)
  ```ruby
  strict_range = Types.Strict(Range)
  strict_range == Types.Instance(Range) # => true
  ```
* `Enum#include?` is an alias to `Enum#valud?` (d-Pixie + flash-gordon)
* `Range` was added (GustavoCaso)
* `Array` types filter out `Undefined` values, if you have an array type with a constructor type as its member, the constructor now can return `Dry::Types::Undefined` to indicate empty value:
  ```ruby
  filter_empty_strings = Types::Strict::Array.of(
    Types::Strict::String.constructor { |input|
      input.to_s.yield_self { |s| s.empty? ? Dry::Types::Undefined : s }
    }
  )
  filter_empty_strings.(["John", nil, "", "Jane"]) # => ["John", "Jane"]
  ```
* `Types::Map` was added for homogeneous hashes, when only types of keys and values are known in advance, not specific key names (fledman + flash-gordon)
  ```ruby
    int_to_string = Types::Hash.map('strict.integer', 'strict.string')
    int_to_string[0 => 'foo'] # => { 0 => "foo" }
    int_to_string[0 => 1] # Dry::Types::MapError: input value 1 for key 0 is invalid: type?(String, 1)
  ```
* Enum supports mappings (bolshakov + flash-gordon)
  ```ruby
  dict = Types::Strict::String.enum('draft' => 0, 'published' => 10, 'archived' => 20)
  dict['published'] # => 'published'
  dict[10] # => 'published'
  ```

## Fixed

* Fixed applying constraints to optional type, i.e. `.optional.constrained` works correctly (flash-gordon)
* Fixed enum working with optionals (flash-gordon)

## Internal

* Dropped the `dry-configurable` dependency (GustavoCaso)
* The gem now uses `dry-inflector` for inflections instead of `inflecto` (GustavoCaso)

[Compare v0.12.2...master](https://github.com/dry-rb/dry-types/compare/v0.12.2...master)

# v0.12.2 2017-11-04

## Fixed

* The type compiler was fixed for simple rules such as used for strict type checks (flash-gordon)
* Fixed an error on `Dry::Types['json.decimal'].try(nil)` (nesaulov)
* Fixed an error on calling `try` on an array type built of constrained types (flash-gordon)
* Implemented `===` for enum types (GustavoCaso)

[Compare v0.12.1...v0.12.2](https://github.com/dry-rb/dry-types/compare/v0.12.1...v0.12.2)

# v0.12.1 2017-10-11

## Fixed

* `Constructor#try` rescues `ArgumentError` (raised in cases like `Integer('foo')`) (flash-gordon)
* `#constructor` works correctly for default and enum types (solnic)
* Optional sum types work correctly in `safe` mode (GustavoCaso)
* The equalizer of constrained types respects meta (flash-gordon)

[Compare v0.12.0...v0.12.1](https://github.com/dry-rb/dry-types/compare/v0.12.0...v0.12.1)

# v0.12.0 2017-09-15

## Added

* A bunch of shortcut methods for constructing types to the autogenerated module, e.g. `Types.Constructor(String, &:to_s)` (flash-gordon)

## Deprecated

* `Types::Array#member` was deprecated in favor of `Types::Array#of` (flash-gordon)

[Compare v0.11.1...v0.12.0](https://github.com/dry-rb/dry-types/compare/v0.11.1...v0.12.0)

# v0.11.1 2017-08-14

## Changed

* Constructors are now equalized using `fn` and `meta` too (flash-gordon)

## Fixed

* Fixed `Constructor#name` with `Sum`-types (flash-gordon)

[Compare v0.11.0...v0.11.1](https://github.com/dry-rb/dry-types/compare/v0.11.0...v0.11.1)

# v0.11.0 2017-06-30

## Added

* `#to_ast` available for all type objects (GustavoCaso)
* `Types::Array#of` as an alias for `#member` (maliqq)
* Detailed failure objects are passed to results which improves constraint violation messages (GustavoCaso)

[Compare v0.10.3...v0.11.0](https://github.com/dry-rb/dry-types/compare/v0.10.3...v0.11.0)

# v0.10.3 2017-05-06

## Added

* Callable defaults accept the underlying type (v-kolesnikov)

[Compare v0.10.2...v0.10.3](https://github.com/dry-rb/dry-types/compare/v0.10.2...v0.10.3)

# v0.10.2 2017-04-28

## Fixed

* Fixed `Type#optional?` for sum types (flash-gordon)

[Compare v0.10.1...v0.10.2](https://github.com/dry-rb/dry-types/compare/v0.10.1...v0.10.2)

# v0.10.1 2017-04-28

## Added

* `Type#optional?` returns true if type is Sum and left is nil (GustavoCaso)
* `Type#pristine` returns a type without `meta` (flash-gordon)

## Fixed

* `meta` is used in type equality again (solnic)
* `Any` works correctly with meta again (flash-gordon)

[Compare v0.10.0...v0.10.1](https://github.com/dry-rb/dry-types/compare/v0.10.0...v0.10.1)

# v0.10.0 2017-04-26

## Added

* Types can be used in `case` statements now (GustavoCaso)

## Fixed

* Return original value when Date.parse raises a RangeError (jviney)

## Changed

* Meta data are now stored separately from options (flash-gordon)
* `Types::Object` was renamed to `Types::Any` (flash-gordon)

[Compare v0.9.4...v0.10.0](https://github.com/dry-rb/dry-types/compare/v0.9.4...v0.10.0)

# v0.9.4 2017-01-24

## Added

* Added `Types::Object` which passes an object of any type (flash-gordon)

[Compare v0.9.3...v0.9.4](https://github.com/dry-rb/dry-types/compare/v0.9.3...v0.9.4)

# v0.9.3 2016-12-03

## Fixed

* Updated to dry-core >= 0.2.1 (ruby warnings are gone) (flash-gordon)

[Compare v0.9.2...v0.9.3](https://github.com/dry-rb/dry-types/compare/v0.9.2...v0.9.3)

# v0.9.2 2016-11-13

## Added

* Support for `"Y"` and `"N"` as `true` and `false` values, respectively (scare21410)

## Changed

* Optimized object allocation in hash schemas, resulting in up to 25% speed boost (davydovanton)

[Compare v0.9.1...v0.9.2](https://github.com/dry-rb/dry-types/compare/v0.9.1...v0.9.2)

# v0.9.1 2016-11-04

## Fixed

* `Hash#strict_with_defaults` properly evaluates callable defaults (bolshakov)

## Changed

* `Hash#weak` accepts Hash-descendants again (solnic)

[Compare v0.9.0...v0.9.1](https://github.com/dry-rb/dry-types/compare/v0.9.0...v0.9.1)

# v0.9.0 2016-09-21

## Added

* `Hash#strict_with_defaults` which validates presence of all required keys and respects default types for missing *values* (backus)
* `Type#constrained?` method (flash-gordon)

## Fixed

* Summing two constrained types works correctly (flash-gordon)
* `Types::Array::Member#valid?` in cases where member type is a constraint (solnic)
* `Hash::Schema#try` handles exceptions properly and returns a failure object (solnic)

## Changed

* [BREAKING] Renamed `Hash##{schema=>permissive}` (backus)
* [BREAKING] `dry-monads` dependency was made optional, Maybe types are available after `Dry::Types.load_extensions(:maybe)` (flash-gordon)
* [BREAKING] `Dry::Types::Struct` and `Dry::Types::Value` have been extracted to [`dry-struct`](https://github.com/dry-rb/dry-struct) (backus)
* `Types::Form::Bool` supports upcased true/false values (kirs)
* `Types::Form::{Date,DateTime,Time}` fail gracefully for invalid input (padde)
* ice_nine dependency has been dropped as it was required by Struct only (flash-gordon)

[Compare v0.8.1...v0.9.0](https://github.com/dry-rb/dry-types/compare/v0.8.1...v0.9.0)

# v0.8.1 2016-07-13

## Fixed

* Compiler no longer chokes on type nodes without args (solnic)
* Removed `bin/console` from gem package (solnic)

[Compare v0.8.0...v0.8.1](https://github.com/dry-rb/dry-types/compare/v0.8.0...v0.8.1)

# v0.8.0 2016-07-01

## Added

* `Struct` now implements `Type` interface so ie `SomeStruct | String` works now (flash-gordon)
* `:weak` Hash constructor which can partially coerce a hash even when it includes invalid values (solnic)
* Types include `Dry::Equalizer` now (flash-gordon)

## Fixed

* `Struct#to_hash` descends into arrays too (nepalez)
* `Default#with` works now (flash-gordon)

## Changed

* `:symbolized` hash schema is now based on `:weak` schema (solnic)
* `Struct::Value` instances are now **deeply frozen** via ice_nine (backus)

[Compare v0.7.2...v0.8.0](https://github.com/dry-rb/dry-types/compare/v0.7.2...v0.8.0)

# v0.7.2 2016-05-11

## Fixed

- `Bool#default` gladly accepts `false` as its value (solnic)
- Creating an empty schema with input processor no longer fails (lasseebert)

## Changed

- Allow multiple calls to meta (solnic)
- Allow capitalised versions of true and false values for boolean coercions (nil0bject)
- Replace kleisli with dry-monads (flash-gordon)
- Use coercions from Kernel (flash-gordon)
- Decimal coercions now work with Float (flash-gordon)
- Coerce empty strings in form posts to blank arrays and hashes (timriley)
- update to use dry-logic v0.2.3 (fran-worley)

[Compare v0.7.1...v0.7.2](https://github.com/dry-rb/dry-types/compare/v0.7.1...v0.7.2)

# v0.7.1 2016-04-06

## Added

- `JSON::*` types with JSON-specific coercions (coop)

## Fixed

- Schema is properly inherited in Struct (backus)
- `constructor_type` is properly inherited in Struct (fbernier)

[Compare v0.7.0...v0.7.1](https://github.com/dry-rb/dry-types/compare/v0.7.0...v0.7.1)

# v0.7.0 2016-03-30

Major focus of this release is to make complex type composition possible and improving constraint errors to be more meaningful.

## Added

- `Type#try` interface that tries to process the input and return a result object which can be either a success or failure (solnic)
- `#meta` interface for setting arbitrary meta data on types (solnic)
- `ConstraintError` has a message which includes information about the predicate which failed ie `nil violates constraints (type?(String) failed)` (solnic)
- `Struct` uses `Dry::Equalizer` too, just like `Value` (AMHOL)
- `Sum::Constrained` which has a disjunction rule built from its types (solnic)
- Compiler supports `[:constructor, [primitive, fn_proc]]` nodes (solnic)
- Compiler supports building schema-less `form.hash` types (solnic)

## Fixed

- `Sum` now supports complex types like `Array` or `Hash` with member types and/or constraints (solnic)
- `Default#constrained` will properly wrap a new constrained type (solnic)

## Changed

- [BREAKING] Renamed `Type#{optional=>maybe}` (AMHOL)
- [BREAKING] `Type#optional(other)` builds a sum: `Strict::Nil | other` (AMHOL)
- [BREAKING] Type objects are now frozen (solnic)
- [BREAKING] `Value` instances are frozen (AMHOL)
- `Array` is no longer a constructor and has a `Array::Member` subclass (solnic)
- `Hash` is no longer a constructor and is split into `Hash::Safe`, `Hash::Strict` and `Hash::Symbolized` (solnic)
- `Constrained` has now a `Constrained::Coercible` subclass which will try to apply its type prior applying its rule (solnic)
- `#maybe` uses `Strict::Nil` now (solnic)
- `Type#default` will raise if `nil` was passed for `Maybe` type (solnic)
- `Hash` with a schema will set maybe values for missing keys or nils (flash-gordon)

[Compare v0.6.0...v0.7.0](https://github.com/dry-rb/dry-types/compare/v0.6.0...v0.7.0)

# v0.6.0 2016-03-16

Renamed from `dry-data` to `dry-types` and:

## Added

* `Dry::Types.module` which returns a namespace for inclusion which has all
  built-in types defined as constants (solnic)
* `Hash#schema` supports default values now (solnic)
* `Hash#symbolized` passes through keys that are already symbols (solnic)
* `Struct.new` uses an empty hash by default as input (solnic)
* `Struct.constructor_type` macro can be used to change attributes constructor (solnic)
* `default` accepts a block now for dynamic values (solnic)
* `Types.register_class` accepts a second arg which is the name of the class'
  constructor method, defaults to `:new` (solnic)

## Fixed

* `Struct` will simply pass-through the input if it is already a struct (solnic)
* `default` will raise if a value violates constraints (solnic)
* Evaluating a default value tries to use type's constructor which makes it work
  with types that may coerce an input into nil (solnic)
* `enum` works just fine with integer-values (solnic)
* `enum` + `default` works just fine (solnic)
* `Optional` no longer responds to `primitive` as it makes no sense since there's
  no single primitive for an optional value (solnic)
* `Optional` passes-through a value which is already a maybe (solnic)

## Changed

* `Dry::Types::Definition` is now the base type definition object (solnic)
* `Dry::Types::Constructor` is now a type definition with a constructor function (solnic)

[Compare v0.5.1...v0.6.0](https://github.com/dry-rb/dry-types/compare/v0.5.1...v0.6.0)

# v0.5.1 2016-01-11

## Added

* `Dry::Data::Type#safe` for types which can skip constructor when primitive does
  not match input's class (solnic)
* `form.array` and `form.hash` safe types (solnic)

[Compare v0.5.0...v0.5.1](https://github.com/dry-rb/dry-types/compare/v0.5.0...v0.5.1)

# v0.5.0 2016-01-11

## Added

* `Type#default` interface for defining a type with a default value (solnic)

## Changed

* [BREAKING] `Dry::Data::Type.new` accepts constructor and *options* now (solnic)
* Renamed `Dry::Data::Type::{Enum,Constrained}` => `Dry::Data::{Enum,Constrained}` (solnic)
* `dry-logic` is now a dependency for constrained types (solnic)
* Constrained types are now always available (solnic)
* `strict.*` category uses constrained types with `:type?` predicate (solnic)
* `SumType#call` no longer needs to rescue from `TypeError` (solnic)

## Fixed

* `attribute` raises proper error when type definition is missing (solnic)

[Compare v0.4.2...v0.5.0](https://github.com/dry-rb/dry-types/compare/v0.4.2...v0.5.0)

# v0.4.2 2015-12-27

## Added

* Support for arrays in type compiler (solnic)

## Changed

* Array member uses type objects now rather than just their constructors (solnic)

[Compare v0.4.1...v0.4.2](https://github.com/dry-rb/dry-types/compare/v0.4.1...v0.4.2)

# v0.4.0 2015-12-11

## Added

* Support for sum-types with constraint type (solnic)
* `Dry::Data::Type#optional` for defining optional types (solnic)

## Changed

* `Dry::Data['optional']` was **removed** in favor of `Dry::Data::Type#optional` (solnic)

[Compare v0.3.2...v0.4.0](https://github.com/dry-rb/dry-types/compare/v0.3.2...v0.4.0)

# v0.3.2 2015-12-10

## Added

* `Dry::Data::Value` which works like a struct but is a value object with equalizer (solnic)

## Fixed

* Added missing require for `dry-equalizer` (solnic)

[Compare v0.3.1...v0.3.2](https://github.com/dry-rb/dry-types/compare/v0.3.1...v0.3.2)

# v0.3.1 2015-12-09

## Changed

* Removed require of constrained type and make it optional (solnic)

[Compare v0.3.0...v0.3.1](https://github.com/dry-rb/dry-types/compare/v0.3.0...v0.3.1)

# v0.3.0 2015-12-09

## Added

* `Type#constrained` interface for defining constrained types (solnic)
* `Dry::Data` can be configured with a type namespace (solnic)
* `Dry::Data.finalize` can be used to define types as constants under configured namespace (solnic)
* `Dry::Data::Type#enum` for defining an enum from a specific type (solnic)
* New types: `symbol` and `class` along with their `strict` versions (solnic)

[Compare v0.2.1...v0.3.0](https://github.com/dry-rb/dry-types/compare/v0.2.1...v0.3.0)

# v0.2.1 2015-11-30

## Added

* Type compiler supports nested hashes now (solnic)

## Fixed

* `form.bool` sum is using correct right-side `form.false` type (solnic)

## Changed

* Improved structure of the ast (solnic)

[Compare v0.2.0...v0.2.1](https://github.com/dry-rb/dry-types/compare/v0.2.0...v0.2.1)

# v0.2.0 2015-11-29

## Added

* `form.nil` which coerces empty strings to `nil` (solnic)
* `bool` sum-type (true | false) (solnic)
* Type compiler supports sum-types now (solnic)

## Changed

* Constructing optional types uses the new `Dry::Data["optional"]` built-in type (solnic)

[Compare v0.1.0...v0.2.0](https://github.com/dry-rb/dry-types/compare/v0.1.0...v0.2.0)

# v0.1.0 2015-11-27

## Added

* `form.*` coercible types (solnic)
* `Type::Hash#strict` for defining hashes with a strict schema (solnic)
* `Type::Hash#symbolized` for defining hashes that will symbolize keys (solnic)
* `Dry::Data.register_class` short-cut interface for registering a class and
  setting its `.new` method as the constructor (solnic)
* `Dry::Data::Compiler` for building a type from a simple ast (solnic)

[Compare v0.0.1...HEAD](https://github.com/dry-rb/dry-types/compare/v0.0.1...HEAD)

# v0.0.1 2015-10-05

First public release
