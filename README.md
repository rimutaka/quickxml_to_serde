# quickxml_to_serde

Convert XML to JSON using [quick-xml](https://github.com/tafia/quick-xml) and [serde](https://github.com/serde-rs/json). Inspired by [node2object](https://github.com/vorot93/node2object).

## Usage examples

#### Basic
Dependencies:

```rust
use std::fs::File;
use std::io::prelude::*;
use quickxml_to_serde::xml_string_to_json;
```
Rust code to perform a conversion:
```rust
// read an XML file into a string
let mut xml_file = File::open("test.xml")?;
let mut xml_contents = String::new();
xml_file.read_to_string(&mut xml_contents)?;

// convert the XML string into JSON with default config params
let json = xml_string_to_json(xml_contents, &Config::new_with_defaults());

println!("{}", json);
```

#### Custom config

The following config example changes the default behavior to:

1. Treat numbers starting with `0` as strings. E.g. `0001` will be `"0001"`
2. Do not prefix JSON properties created from attributes
3. Use `text` as the JSON property name for values of XML text nodes where the text is mixed with other nodes
4. Exclude empty elements from the output

```rust
let conf = Config::new_with_custom_values(true, "", "text", NullValue::Ignore);
```

#### Enforcing JSON types

The default for this library is to attempt to infer scalar data types, which can be `int`, `float`, `bool` or `string` in JSON. Sometimes it is not desirable like in the example below. Let's assume that attribute `id` is always numeric and can be safely converted to JSON integer.
The `card_number` element looks like a number for the first two users and is a string for the third one. This inconsistency in JSON typing makes it
difficult to deserialize the structure, so we may be better off telling the converter to use a particular JSON data type for some XML nodes.
```xml
<users>
	<user id="1">
		<name>Andrew</name>
		<card_number>000156</card_number>
	</user>
	<user id="2">
		<name>John</name>
		<card_number>100263</card_number>
	</user>
	<user id="3">
		<name>Mary</name>
		<card_number>100263a</card_number>
	</user>
</users>
```

Use `quickxml_to_serde = { version = "0.4", features = ["json_types"] }` feature in your *Cargo.toml* file to enable support for enforcing JSON types for some XML nodes using xPath-like notations.

Sample XML document:
```xml
<a attr1="007"><b attr1="7">true</b></a>
```
Configuration to make attribute `attr1="007"` always come out as a JSON string:
```rust
let conf = Config::new_with_defaults().add_json_type_override("/a/@attr1", JsonType::AlwaysString);
```
Configuration to make both attributes and the text node of `<b />` always come out as a JSON string:
```rust
let conf = Config::new_with_defaults()
          .add_json_type_override("/a/@attr1", JsonType::AlwaysString)
          .add_json_type_override("/a/b/@attr1", JsonType::AlwaysString)
          .add_json_type_override("/a/b", JsonType::AlwaysString);
```

See embedded docs for `Config` struct and its members for more details. 

## Conversion specifics

- The order of XML elements is not preserved
- Namespace identifiers are dropped. E.g. `<xs:a>123</xs:a>` becomes `{ "a":123 }`
- Integers and floats are converted into JSON integers and floats, unless the JSON type is specified in `Config`.
- XML attributes become JSON properties at the same level as child elements. E.g.
```xml
<Test TestId="0001">
  <Input>1</Input>
</Test>
```
is converted into
```json
"Test":
  {
    "Input": 1,
    "TestId": "0001"
  }
```
- XML prolog is dropped. E.g. `<?xml version="1.0"?>`.
- XML namespace definitions are dropped. E.g. `<Tests xmlns="http://www.adatum.com" />` becomes `"Tests":{}`
- Processing instructions, comments and DTD are ignored
- **Presence of CDATA in the XML results in malformed JSON**
- XML attributes can be prefixed via `Config::xml_attr_prefix`. E.g. using the default prefix `@` converts `<a b="y" />` into `{ "a": {"@b":"y"} }`. You can use no prefix or set your own value. 
- Complex XML elements with text nodes put the XML text node value into a JSON property named in `Config::xml_text_node_prop_name`. E.g. setting `xml_text_node_prop_name` to `text` will convert
```xml
<CardNumber Month="3" Year="19">1234567</CardNumber>
```
into
```json
{
  "CardNumber": {
          "Month": 3,
          "Year": 19,
          "text": 1234567
        }
}
```
- Elements with identical names are collected into arrays. E.g.
```xml
<Root>
  <TaxRate>7.25</TaxRate>
  <Data>
    <Category>A</Category>
    <Quantity>3</Quantity>
    <Price>24.50</Price>
  </Data>
  <Data>
    <Category>B</Category>
    <Quantity>1</Quantity>
    <Price>89.99</Price>
  </Data>
</Root>
```
is converted into
```json
{
  "Root": {
    "Data": [
      {
        "Category": "A",
        "Price": 24.5,
        "Quantity": 3
      },
      {
        "Category": "B",
        "Price": 89.99,
        "Quantity": 1
      }
    ],
    "TaxRate": 7.25
  }
}
```
-  If `TaxRate` element from the above example was inserted between `Data` elements it would still produce the same JSON with all `Data` properties grouped into a single array.

#### Additional info and examples

See `mod tests` inside [lib.rs](src/lib.rs) for more usage examples.

## Edge cases

XML and JSON are not directly compatible for 1:1 conversion without additional hints to the converter. Feel free to post an issue if you come across any incorrect conversion.
