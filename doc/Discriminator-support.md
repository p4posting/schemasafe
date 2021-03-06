# Discriminator support

`@exodus/schemasafe` supports a strict subset of the
[`discriminator` API](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.1.0.md#discriminatorObject).

It has the following additional requirements to be safe and compatible with upstream JSON Schema
specification:

  1. At least one of `oneOf` or `anyOf` is required. Using both at the same time is not allowed.

  2. Type of the object using `discriminator` must be provable to be `object` — can be specified
     either as `type` on the same level as `discriminator`, or in each branch separately.

  3. Each `oneOf`/`anyOf` branch **must** have a `const` _or_ `enum` rule on the property targeted
     by `propertyName` of the `discriminator`, either directly or inside a `$ref`.

     _While seeming a bit excessive, this is the rule that makes `discriminator` both well-defined
     and compatible with JSON Schema spec, while being a subset of OAPI `discriminator`._

     Currently, those `const` or `enum` values must be unique strings, which additionally makes
     `oneOf` and `anyOf` identical there in relation to the validation result. The uniqueness and
     being a string requirement might be lifted in the future, if it would be deemed useful.

  4. Property targeted by `propertyName` of the `discriminator` **must** be placed in `required`,
     either on the same level as `discriminator`, or in each branch separately. Failing to do so
     would make _all_ `const` checks pass on that property, per the JSON Schema specification.

  5. `mapping` is supported but only when it has the exact same set of branches as `oneOf`/`anyOf`,
     values of the `mapping` correspond to used `$ref` values of the branches and the keys of
     the mapping match `const`/`enum` values on the `propertyName` of the `discriminator` in
     corresponding branches.

     That way, `mapping` doesn't really do anything at all, and brings in no new information to the
     validator. It only serves as a coherence check (`@exodus/schemasafe` will refuse to compile a
     schema which has a `mapping` that mismatches the above constraints) and as an additional
     commentary to whoever is reading the schema.

## Action

Given these constraints, `discriminator` is a provable noop in relation to the validation result.

The resulting constructions are 1:1 compatible with JSON Schema Spec — i.e. for each input, the
validation result stays the same with or without the `discriminator` rule.

It affects three things though:

  1. Error reporting — only errors related to the target `oneOf`/`anyOf` branch would be reported
     (or a single error if doesn't match any), instead of errors from each branch merged together.

     Without `discriminator` in the same `oneOf` + `const`/`enum` combination, it might be very hard
     to understand from a large set of errors (of a first unrelated error) which exactly mismatched,
     if the validation failed.

  2. Optimization — the validator can optimize this efficiently, i.e. first just check the `const`
     or `enum` value, then only check the corresponding branch of rules.

     This is also possible in `allErrors` mode due to filtering out errors from other branches.

     While this would have been doable without a `discriminator` rule, that affects the list of
     errors that are reported, and affecting that list without an explicit opt-in does not seem
     to be a good idea.

  3. Schema readability — this makes the schema more clear that just a `oneOf`/`anyOf` over
     a set of branches with `const`/`enum` rules.

## Examples

All of the examples below are equivalent, i.e. pass and fail on the same input and produce a similar
list of errors, differing in just the `keywordLocation` pointer because of refs.

```json
{
  "type": "object",
  "required": ["objectType"],
  "discriminator": { "propertyName": "objectType" },
  "oneOf": [{
    "properties": { "objectType": { "const": "obj1" } },
    "required": ["a"]
  }, {
    "properties": { "objectType": { "const": "obj2" } },
    "required": ["b"]
  }]
}
```

```json
{
  "$defs": {
    "obj1": {
      "type": "object",
      "properties": { "objectType": { "const": "obj1" } },
      "required": ["objectType", "a"]
    },
    "obj2": {
      "type": "object",
      "properties": { "objectType": { "const": "obj2" } },
      "required": ["objectType", "b"]
    }
  },
  "discriminator": { "propertyName": "objectType" },
  "oneOf": [
    { "$ref": "#/$defs/obj1" },
    { "$ref": "#/$defs/obj2" }
  ]
}
```

```json
{
  "$defs": {
    "obj1": {
      "type": "object",
      "properties": { "objectType": { "const": "obj1" } },
      "required": ["objectType", "a"]
    },
    "obj2": {
      "type": "object",
      "properties": { "objectType": { "const": "obj2" } },
      "required": ["objectType", "b"]
    }
  },
  "discriminator": {
    "propertyName": "objectType",
    "mapping": {
      "obj1": "#/$defs/obj1",
      "obj2": "#/$defs/obj2"
    }
  },
  "oneOf": [
    { "$ref": "#/$defs/obj1" },
    { "$ref": "#/$defs/obj2" }
  ]
}
```

```json
{
  "$defs": {
    "obj1": { "required": ["a"] },
    "obj2": { "required": ["b"] }
  },
  "type": "object",
  "required": ["objectType"],
  "discriminator": { "propertyName": "objectType" },
  "oneOf": [{
    "properties": { "objectType": { "const": "obj1" } },
    "$ref": "#/$defs/obj1"
  }, {
    "properties": { "objectType": { "const": "obj2" } },
    "$ref": "#/$defs/obj2"
  }]
}
```
