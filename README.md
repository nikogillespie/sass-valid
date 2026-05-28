# sass-valid

Sass schema-driven validation — per the [Error Message Specification](https://github.com/nikogillespie/sass-error-spec).

---

## Installation

```sh
npm install sass-valid sass-funcs sass-error
```

> Requires [`sass`](https://www.npmjs.com/package/sass) or [`sass-embedded`](https://www.npmjs.com/package/sass-embedded) `>= 1.33.0` — install one, not both (`sass-embedded` recommended). Also requires [`sass-funcs`](https://www.npmjs.com/package/sass-funcs) and [`sass-error`](https://www.npmjs.com/package/sass-error).

---

## Configuration

All constants are configurable via `@use ... with (...)`:

```scss
@use 'sass-valid' as v with (
  $VALIDATION_MODE_SELECTED: 'strict',
  $ERROR_HINT_TEXT: 'Refer to the schema'
);
```

| Variable                    | Default                           | Description                                                        |
| --------------------------- | --------------------------------- | ------------------------------------------------------------------ |
| `$VALUE_ID_ALLOWED_TYPES`   | `('string', 'number', 'bool')`    | Allowed Sass types for map keys (entity attributes, schema fields) |
| `$FIELD_REQUIRED`           | `'is-required'`                   | Schema field name for the required flag                            |
| `$FIELD_ALLOWED_TYPES`      | `'allowed-types'`                 | Schema field name for allowed types                                |
| `$FIELD_ALLOWED_VALUES`     | `'allowed-values'`                | Schema field name for allowed values                               |
| `$FIELD_VALIDATOR`          | `'validator'`                     | Schema field name for the custom validator name                    |
| `$ERROR_HINT_TEXT`          | `'See schema for allowed values'` | Hint appended when the options list is truncated                   |
| `$ERROR_LABEL_ENTRY`        | `'ENTRY'`                         | Default label for collection member keys in error codes            |
| `$ERROR_LABEL_FIELD`        | `'FIELD'`                         | Default label for schema field keys in error codes                 |
| `$ERROR_ENTRY_LABELS`       | `('ENTRY', 'FIELD')`              | Allowed values for `$custom-entry-label`                           |
| `$VALIDATION_MODE_SELECTED` | `'rigid'`                         | Default validation mode                                            |

> **Note:** See the **[Error Message Specification](https://github.com/nikogillespie/sass-error-spec)** for details on how error codes and labels are structured.

---

## [Module: Valid](index.scss)

```scss
@use 'sass-valid' as v;
```

| Function                               | Description                                         |
| -------------------------------------- | --------------------------------------------------- |
| [`validate`](#vvalidateschema-entity-) | Validates an entity map against a schema definition |

---

### `v.validate($schema, $entity, ...)`

Validates an entity map against a schema. The schema is checked against a built-in metaschema on first use and the result is cached — repeated calls with the same schema incur no extra cost.

| Parameter             | Type             | Default                     | Description                                                                    |
| --------------------- | ---------------- | --------------------------- | ------------------------------------------------------------------------------ |
| `$schema`             | `Map`            |                             | Schema definition map                                                          |
| `$entity`             | `Map`            |                             | Entity map to validate                                                         |
| `$entity-type`        | `String`         | `'entity'`                  | Label for the entity type in error messages                                    |
| `$entity-name`        | `String`         | `'unnamed'`                 | Label for the entity name in error messages                                    |
| `$mode`               | `String`         | `$VALIDATION_MODE_SELECTED` | Validation mode: `'permissive'`, `'rigid'`, or `'strict'`                      |
| `$custom-entry-label` | `String`         | `$ERROR_LABEL_ENTRY`        | Label for entry-level error codes: `'ENTRY'` or `'FIELD'`                      |
| `$schema-id`          | `String \| Null` | `null`                      | Cache key override; use when the same `$entity-type` maps to different schemas |
| `$validators`         | `Map`            | `()`                        | Map of validator name strings to Sass function references                      |

**Returns** `Null` on success, `String` error message on failure (see [Error Message Specification](https://github.com/nikogillespie/sass-error-spec)) — pass to `@error` to raise.

```scss
$schema: (
  'name': ('is-required': true,  'allowed-types': 'string'),
  'role': ('is-required': true,  'allowed-types': 'string', 'allowed-values': ('admin', 'user', 'guest')),
  'age':  ('is-required': false, 'allowed-types': 'number'),
);

v.validate($schema, ('name': 'Alice', 'role': 'admin'), 'person', 'Alice')
// → null

v.validate($schema, ('name': 'Alice', 'role': 42), 'person', 'Alice')
// → '[PERSON_ROLE_TYPE] Person "Alice" @ role: Incorrect type "number" → Expected: string'

v.validate($schema, ('name': 'Alice', 'role': 'owner'), 'person', 'Alice')
// → '[PERSON_ROLE_VALUE] Person "Alice" @ role: Invalid value "owner" → Allowed: admin | user | guest'

v.validate($schema, ('name': 'Alice', 'role': 'admin', 'rank': 1), 'person', 'Alice')
// → '[PERSON_RANK_UNKNOWN] Person "Alice": Unknown attribute "rank" → Remove "rank" [context: mode "rigid"]'
```

---

## Schema Definition

A schema is a Sass map where each key is an attribute name and each value is a rules map:

```scss
$schema: (
  '<attribute>': (
    'is-required': <bool>,
    // optional — defaults to false
    'allowed-types': '<type>',
    // required — Sass type name or list of names
    'allowed-values': (
        <val>,
        ...,
      ),
    // optional — allowed values, or a nested schema map
    'validator': '<name>',
    // optional — custom validator key
  ),
  ...,
);
```

> The field names `'is-required'`, `'allowed-types'`, `'allowed-values'`, and `'validator'` are configurable via `$FIELD_REQUIRED`, `$FIELD_ALLOWED_TYPES`, `$FIELD_ALLOWED_VALUES`, and `$FIELD_VALIDATOR`.

| Field              | Type             | Required | Description                                                                                                                                                         |
| ------------------ | ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `'is-required'`    | `Bool`           | No       | Whether the attribute must be present. Always `true` in `'strict'` mode.                                                                                            |
| `'allowed-types'`  | `String \| List` | Yes      | One or more Sass type names: `'number'`, `'string'`, `'list'`, `'bool'`, `'map'`                                                                                    |
| `'allowed-values'` | `* \| Map`       | No       | Allowed values as a flat list, or a nested schema map when the attribute type is `'map'`. When a list, each element of the attribute value is checked individually. |
| `'validator'`      | `String`         | No       | Name key matching an entry in `$validators`                                                                                                                         |

### Nested Schemas

When `'allowed-values'` is a map and the attribute value is also a map, `validate` recurses into the nested schema using the same mode and custom validators:

```scss
$schema: (
  'config': (
    'is-required': true,
    'allowed-types': 'map',
    'allowed-values': (
      'theme': ('is-required': true,  'allowed-types': 'string', 'allowed-values': ('light', 'dark')),
      'scale': ('is-required': false, 'allowed-types': 'number'),
    ),
  ),
);

v.validate($schema, ('config': ('theme': 'dark', 'scale': 1.5)), 'settings', 'app')
// → null

v.validate($schema, ('config': ('theme': 'blue')), 'settings', 'app')
// → '[SETTINGS_THEME_VALUE] Settings "app" @ config > theme: Invalid value "blue" → Allowed: light | dark'
```

---

## Validation Modes

| Mode           | Unknown attributes | Missing optional attributes |
| -------------- | ------------------ | --------------------------- |
| `'permissive'` | Allowed            | Skipped                     |
| `'rigid'`      | Error              | Skipped                     |
| `'strict'`     | Error              | Error                       |

The default mode is `'rigid'`. Override per-call via `$mode`, or globally via `$VALIDATION_MODE_SELECTED`:

```scss
v.validate($schema, $entity, 'person', 'Alice', $mode: 'permissive')  // extra fields allowed
v.validate($schema, $entity, 'person', 'Alice', $mode: 'rigid')       // default
v.validate($schema, $entity, 'person', 'Alice', $mode: 'strict')      // all fields required
```

---

## Custom Validators

A custom validator is a Sass function that performs checks on an attribute value beyond type and `'allowed-values'` validation. It is called after both pass and must return `null` on success or an error string on failure.

Register validators by passing a map of name strings to Sass function references via `$validators`. A validator name present in a schema but absent from this map triggers an "unresolved function" error at runtime.

Custom validator functions receive these positional arguments:

| Position | Name                  | Type     | Description                               |
| -------- | --------------------- | -------- | ----------------------------------------- |
| 1        | `$entity`             | `Map`    | The full entity map                       |
| 2        | `$attribute`          | `String` | The attribute name being validated        |
| 3        | `$attribute-rules`    | `Map`    | The schema rules map for this attribute   |
| 4        | `$code-base`          | `String` | Error code prefix (e.g. `'PERSON_AGE'`)   |
| 5        | `$entity-type-cap`    | `String` | Capitalized entity type (e.g. `'Person'`) |
| 6        | `$entity-name`        | `String` | The entity name (e.g. `'Alice'`)          |
| 7        | `$current-path`       | `List`   | Current path list (e.g. `('age',)`)       |
| 8        | `$custom-entry-label` | `String` | Entry label for error codes               |

**Returns** `Null` on success, `String` error message on failure.

```scss
@use 'sass-valid' as v;
@use 'sass:meta';
@use 'sass:map';

@function -positive-number($entity, $attribute, $rules, $code, $type, $name, $path, $label) {
  $value: map.get($entity, $attribute);
  @if $value <= 0 {
    @return '[#{$code}_VALUE] #{$type} "#{$name}" @ #{$attribute}: Invalid value "#{$value}" → Expected: positive number';
  }
  @return null;
}

$schema: (
  'count': ('is-required': true, 'allowed-types': 'number', 'validator': 'positive-number'),
);

$validators: ('positive-number': meta.get-function('-positive-number'));

v.validate($schema, ('count': 5),  'item', 'box', $validators: $validators)
// → null

v.validate($schema, ('count': -1), 'item', 'box', $validators: $validators)
// → '[ITEM_COUNT_VALUE] Item "box" @ count: Invalid value "-1" → Expected: positive number'
```

---

## Migration

### v1 → v2

- `$custom-validators` renamed to `$validators`
- `ARGUMENT_CUSTOM_VALIDATORS_TYPE` renamed to `ARGUMENT_VALIDATORS_TYPE`

---

[Back to top](#sass-valid)
