What is a Redis String?
```
Redis Key
    │
    ▼
String Value
```
Example
```
user:123:name → "Alice"
```
But String does not mean only text

A Redis String can represent:
```
"Hello"
"12345"
JSON
tokens
serialized objects
binary data
```

>Redis String is a general-purpose byte/value container associated with one Redis key.

Redis uses **one logical String data type for arbitrary byte sequences**, while the internal metadata/encoding tells Redis how that value is represented and operated on.