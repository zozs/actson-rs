# Actson [![Actions Status](https://github.com/michel-kraemer/actson-rs/workflows/Rust/badge.svg)](https://github.com/michel-kraemer/actson-rs/actions) [![MIT license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE) [![Latest Version](https://img.shields.io/crates/v/actson.svg)](https://crates.io/crates/actson) [![Documentation](https://img.shields.io/docsrs/actson/latest)](https://docs.rs/actson/latest/actson/)

Actson is a reactive JSON parser (sometimes referred to as non-blocking or
asynchronous). It is event-based and can be used in asynchronous code (for example in combination with [Tokio](https://tokio.rs/)).

## Why another JSON parser?

* **Non-blocking.** Other JSON parsers use blocking I/O. If you want to develop a reactive application you should use
  non-blocking I/O (see the [Reactive Manifesto](http://www.reactivemanifesto.org/)).
* **Big Data.** Most parsers read the full JSON text into memory to map it to
  a struct, for example. Actson can handle arbitrarily large JSON text. It is
  event-based and can be used for streaming.
* **GeoRocket.** Actson was primarily developed for the [GeoJSON](http://geojson.org/) support support in [GeoRocket](http://georocket.io),
  a high-performance reactive data store for geospatial files.

## Usage

### Push-based parsing

Push-based parsing is the most flexible way of using Actson. Push new bytes
into a `PushJsonFeeder` and then let the parser consume them until it returns
`JsonEvent::NeedMoreInput`. Repeat this process until you receive
`JsonEvent::Eof` or `JsonEvent::Error`.

This approach is very low-level but gives you the freedom to provide new bytes
to the parser whenever they are available and to generate new JSON events
whenever you need them.

```rust
use actson::{JsonParser, JsonEvent};
use actson::feeder::{PushJsonFeeder, JsonFeeder};

let json = r#"{"name": "Elvis"}"#.as_bytes();

let feeder = PushJsonFeeder::new();
let mut parser = JsonParser::new(feeder);
let mut i: usize = 0;
loop {
    // feed as many bytes as possible to the parser
    let mut event = parser.next_event();
    while event == JsonEvent::NeedMoreInput {
        i += parser.feeder.push_bytes(&json[i..]);
        if i == json.len() {
            parser.feeder.done();
        }
        event = parser.next_event();
    }

    // do something useful with `event`
    // match event {
    //     ...
    // }

    assert_ne!(event, JsonEvent::Error);

    if event == JsonEvent::Eof {
        break;
    }
}
```

### Asynchronous parsing with Tokio

Actson can be used with Tokio to parse JSON asynchronously.

The main idea here is to call `JsonParser::next_event()` in a loop to
parse the JSON document and to produce events. Whenever you get
`JsonEvent::NeedMoreInput`, call `AsyncBufReaderJsonFeeder::fill_buf()`
to asynchronously read more bytes from the input and to provide them to
the parser.

*Heads up:* The `tokio` feature has to be enabled for this. It is disabled
by default.

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt, BufReader};

use actson::{JsonParser, JsonEvent};
use actson::tokio::AsyncBufReaderJsonFeeder;

#[tokio::main]
async fn main() {
    let file = File::open("tests/fixtures/pass1.txt").await.unwrap();
    let reader = BufReader::new(file);

    let feeder = AsyncBufReaderJsonFeeder::new(reader);
    let mut parser = JsonParser::new(feeder);
    loop {
        let mut event = parser.next_event();
        if event == JsonEvent::NeedMoreInput {
            parser.feeder.fill_buf().await.unwrap();
            event = parser.next_event();
        }

        // do something useful with `event`
        // match event {
        //     ...
        // }

        assert_ne!(event, JsonEvent::Error);

        if event == JsonEvent::Eof {
            break;
        }
    }
}
```

### Parsing from a `BufReader`

`BufReaderJsonFeeder` allows you to feed the parser from a `std::io::BufReader`.

*Note:* By following this synchronous and blocking approach, you are missing
out on Actson's reactive properties. We recommend using Actson together
with Tokio instead to parse JSON asynchronously (see above).

```rust
use actson::{JsonParser, JsonEvent};

use std::fs::File;
use std::io::BufReader;

let file = File::open("tests/fixtures/pass1.txt").unwrap();
let reader = BufReader::new(file);

let feeder = actson::feeder::BufReaderJsonFeeder::new(reader);
let mut parser = JsonParser::new(feeder);
loop {
    let mut event = parser.next_event();
    if event == JsonEvent::NeedMoreInput {
        parser.feeder.fill_buf().unwrap();
        event = parser.next_event();
    }

    // do something useful with `event`
    // match event {
    //     ...
    // }

    assert_ne!(event, JsonEvent::Error);

    if event == JsonEvent::Eof {
        break;
    }
}
```

### Parsing a slice of bytes

For convenience, `SliceJsonFeeder` allows you to feed the parser from a slice
of bytes.

```rust
use actson::{JsonParser, JsonEvent};
use actson::feeder::SliceJsonFeeder;

let json = r#"{"name": "Elvis"}"#.as_bytes();

let feeder = SliceJsonFeeder::new(json);
let mut parser = JsonParser::new(feeder);
loop {
    let event = parser.next_event();

    // do something useful with `event`
    // match event {
    //     ...
    // }

    assert_ne!(event, JsonEvent::Error);

    if event == JsonEvent::Eof {
        break;
    }
}
```

### Parsing into a Serde JSON Value

For testing and compatibility reasons, Actson is able to parse a byte slice
into a [Serde JSON](https://github.com/serde-rs/json) Value.

*Heads up:* You need to enable the `serde_json` feature for this.

```rust
use actson::serde_json::from_slice;

let json = r#"{"name": "Elvis"}"#.as_bytes();
let value = from_slice(json).unwrap();

assert!(value.is_object());
assert_eq!(value["name"], "Elvis");
```

However, if you find yourself doing this, you probably don't need the reactive
features of Actson and your data seems to completely fit into memory. In this
case, you're most likely better off using Serde JSON directly.

## Other languages

Besides this implementation in Rust here, there is a [Java implementation](https://github.com/michel-kraemer/actson).

## Acknowledgments

The event-based parser code and the JSON files used for testing are largely
based on the file [JSON_checker.c](http://www.json.org/JSON_checker/) and
the JSON test suite from [JSON.org](http://www.json.org/) originally released
under [this license](LICENSE_JSON_checker) (basically MIT license).

## License

Actson is released under the **MIT license**. See the
[LICENSE](LICENSE) file for more information.
