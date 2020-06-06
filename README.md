# Protox

[![Build Status](https://travis-ci.org/ahamez/protox.svg?branch=master)](https://travis-ci.org/ahamez/protox) [![Coverage Status](https://coveralls.io/repos/github/ahamez/protox/badge.svg?branch=master)](https://coveralls.io/github/ahamez/protox?branch=master) [![Hex.pm Version](http://img.shields.io/hexpm/v/protox.svg)](https://hex.pm/packages/protox) [![Inline docs](https://inch-ci.org/github/ahamez/protox.svg)](https://inch-ci.org/github/ahamez/protox)


Protox is a native Elixir library to work with [Google's Protocol Buffers](https://developers.google.com/protocol-buffers) (aka protobuf), versions 2 and 3.

Generally speaking, a lof of effort has been put into making sure that the library is reliable (for intance using [property based testing](https://github.com/alfert/propcheck) and by having a [100% code coverage](https://coveralls.io/github/ahamez/protox?branch=master)). As such, this library passes all the tests of the conformance checker provided by Google. See [Conformance](https://github.com/ahamez/protox#conformance) section for more information.

## Prerequisites

Protox uses Google's `protoc` (>= 3.0) to parse `.proto` files. It must be available in `$PATH`. This dependency is only required at compile-time.
You can get it [here](https://github.com/google/protobuf).


## Installation

Add `:protox` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [{:protox, "~> 0.22.0"}]
end
```

## Usage with a textual description

Here's how to generate the modules from a textual description:

```elixir
defmodule Bar do
  use Protox, schema: """
  syntax = "proto3";

  package fiz;

  message Baz {
  }

  message Foo {
    int32 a = 1;
    map<int32, Baz> b = 2;
  }
  """
end
```

This example will generate two modules: `Fiz.Baz` and `Fiz.Foo`.
Note that the module in which the `Protox` macro is called is completely ignored and as such does not appear in the names of the generated modules.

## Usage with files

Here's how to generate the modules from a set of files:

```elixir
defmodule Foo do
  @external_resource "./defs/foo.proto"
  @external_resource "./defs/bar.proto"
  @external_resource "./defs/baz/fiz.proto"

  use Protox, files: [
    "./defs/foo.proto",
    "./defs/bar.proto",
    "./defs/baz/fiz.proto",
  ]
end
```

Again, the module in which the `Protox` macro is called is completely ignored.

## Encode

Here's how to create and encode a new message:

```elixir
iex> msg = %Fiz.Foo{a: 3, b: %{1 => %Fiz.Baz{}}}
iex> Protox.Encode.encode(msg)
```

As you can see, you can interact with protobuf messages as if they were native Elixir structures!

Note that `Protox.Encode.encode/1` returns an [IO data](https://hexdocs.pm/elixir/IO.html#module-use-cases-for-io-data), not a binary, for efficiency reasons. Such  IO data can be used
directly with [files](https://hexdocs.pm/elixir/IO.html#binwrite/2) or sockets write operations, and as such you don't need to transform them:
```elixir
iex> {:ok, file} = File.open("msg.bin", [:write])
{:ok, #PID<0.1023.0>}

iex> iodata = Protox.Encode.encode(%Fiz.Foo{a: 3, b: %{1 => %Fiz.Baz{}}})
[[[], <<18>>, <<4>>, "\b", <<1>>, <<18>>, <<0>>], "\b", <<3>>]

iex> IO.binwrite(file, iodata)
:ok
```

However, you can use [`:binary.list_to_bin/1`](https://erlang.org/doc/man/binary.html#list_to_bin-1) or [`IO.iodata_to_binary`](https://hexdocs.pm/elixir/IO.html#iodata_to_binary/1) to get a binary should the need arises:

```elixir
iex> %Fiz.Foo{a: 3, b: %{1 => %Fiz.Baz{}}} |> Protox.Encode.encode() |> :binary.list_to_bin()
<<8, 3, 18, 4, 8, 1, 18, 0>>
```

## Decode

Here's how to decode a message from a binary:

```elixir
iex> Fiz.Foo.decode(<<8, 3, 18, 4, 8, 1, 18, 0>>)
{:ok,
 %Fiz.Foo{__uf__: [], a: 3,
  b: %{1 => %Fiz.Baz{__uf__: []}}}}
```

The `__uf__` field is explained in the section [Unknown fields](https://github.com/ahamez/protox#unknown-fields).


## Working with namespaces

It's possible to prepend a namespace to all generated modules:

```elixir
defmodule Bar do
  use Protox, schema: """
    syntax = "proto3";

    message Msg {
        int32 a = 1;
      }
    """,
    namespace: Namespace
end
```

In this example, the module `Namespace.Msg` is generated.

```elixir
iex> msg = %Namespace.Msg{a: 42}
```

## Specify import path

An import path can be specified using the `path:` option:

```elixir
defmodule Baz do
  @external_resource "./defs/prefix/foo.proto"
  @external_resource "./defs/prefix/bar/bar.proto"

  use Protox,
    files: [
      "./defs/prefix/foo.proto",
      "./defs/prefix/bar/bar.proto",
    ],
    path: "./defs"
end
```

It corresponds to the `-I` option of `protoc`.

## Unknown fields

If any unknown fields are encountered when decoding, they are kept in the decoded message.
It's possible to access them with the function `unknown_fields/1` defined with the message.

```elixir
iex> msg = Msg.decode!(<<8, 42, 42, 4, 121, 97, 121, 101, 136, 241, 4, 83>>)
%Msg{a: 42, b: "", z: -42, __uf__: [{5, 2, <<121, 97, 121, 101>>}]}

iex> Msg.unknown_fields(msg)
[{5, 2, <<121, 97, 121, 101>>}]
```

You must always use `unknown_fields/1` as the name of the field
(e.g. `__uf__`) is generated at compile-time to avoid collision with the actual
fields of the Protobuf message.

This function returns a list of tuples `{tag, wire_type, bytes}`.


## Unsupported features

* Protobuf 3 JSON mapping
* Groups ([deprecated in protobuf](https://developers.google.com/protocol-buffers/docs/proto#groups))
* All [options](https://developers.google.com/protocol-buffers/docs/proto3#options) other than `packed` and `default` are ignored as they concern other languages implementation details.


## Implementation choices

* Required fields (Protobuf 2): an error is raised when encoding or decoding a message with a missing required
  field.

* When decoding enum aliases, the last encountered constant is used. For instance, in the following example, `:BAR` is always used if the value `1` is read on the wire:
    ```protobuf
    enum E {
      option allow_alias = true;
      FOO = 0;
      BAZ = 1;
      BAR = 1;
    }
    ```

* Unset optionals
    * For Protobuf 2, unset optional fields are mapped to `nil`. You can use the generated `default/1` function to get the default value of a field:
        ```elixir
        defmodule Bar do
          use Protox,
          schema: """
            syntax = "proto2";

            message Foo {
              optional int32 a = 1 [default = 42];
            }
          """
        end

        iex> Foo.default(:a)
        {:ok, 42}

        iex> %Foo{}.a
        nil

        ```

    * For Protobuf 3, unset optional fields are mapped to their default values, as mandated by the [Protobuf spec](https://developers.google.com/protocol-buffers/docs/proto3#default):
        ```elixir
        defmodule Bar do
          use Protox,
          schema: """
            syntax = "proto3";

            message Foo {
              int32 a = 1;
            }
          """
        end

        iex> Foo.default(:a)
        {:ok, 0}

        iex> %Foo{}.a
        0

        ```

* Messages and enums names: names are converted using the [`Macro.camelize/1`](https://hexdocs.pm/elixir/Macro.html#camelize/1) function.
  Thus, in the following example, `non_camel_message` becomes `NonCamelMessage`, but the field `non_camel_field` is left unchanged:
    ```elixir
    defmodule Bar do
      use Protox,
      schema: """
        syntax = "proto3";

        message non_camel_message {
        }

        message CamelMessage {
          int32 non_camel_field = 1;
        }
      """
    end


    iex> msg = %NonCamelMessage{}
    %NonCamelMessage{__uf__: []}

    iex> msg = %CamelMessage{}
    %CamelMessage{__uf__: [], non_camel_field: 0}
    ```

## Types mapping

The following table shows how Protobuf types are mapped to Elixir's ones.

Protobuf   | Elixir
-----------|--------------
int32      | integer()
int64      | integer()
uint32     | integer()
uint64     | integer()
sint32     | integer()
sint64     | integer()
fixed32    | integer()
fixed64    | integer()
sfixed32   | integer()
sfixed64   | integer()
float      | float() \| :infinity \| :'-infinity' \| :nan
double     | float() \| :infinity \| :'-infinity' \| :nan
bool       | boolean()
string     | String.t()
bytes      | binary()
map        | %{}
oneof      | {:field, value}
enum       | atom() \| integer()
message    | struct()

## Conformance

The protox library has been thoroughly tested using the [conformance checker provided by Google](https://github.com/protocolbuffers/protobuf/tree/master/conformance). Note that only the binary part is tested as protox supports only this format. For instance, JSON tests are skipped.

Here's how to launch the conformance test:

* Get conformance-test-runner [sources](https://github.com/google/protobuf/archive/v3.12.1.tar.gz).
* Compile conformance-test-runner:
  `tar xf protobuf-3.12.1.tar.gz && cd protobuf-3.12.1 && ./autogen.sh && ./configure && make -j && cd conformance && make -j`
* `mix protox.conformance --runner=/path/to/protobuf-3.12.1/conformance/conformance-test-runner`.
  A report will be generated in a directory `conformance_report`.
  If everything's fine, the following text should be displayed:

  ```
  CONFORMANCE TEST BEGIN ====================================

  CONFORMANCE SUITE PASSED: 1302 successes, 705 skipped, 0 expected failures, 0 unexpected failures.


  CONFORMANCE TEST BEGIN ====================================

  CONFORMANCE SUITE PASSED: 0 successes, 69 skipped, 0 expected failures, 0 unexpected failures.
  ```

You can alternatively launch these conformance tests with `mix test` by setting the `PROTOBUF_CONFORMANCE_RUNNER` environment variable and including the `conformance` tag:
   ```
   PROTOBUF_CONFORMANCE_RUNNER=./protobuf-3.12.1/conformance/conformance-test-runner MIX_ENV=test mix test --include conformance
   ```

# Benchmarks

You can launch benchmarks to see how Protox perform:
```
MIX_ENV=benchmarks mix run benchmarks/run.exs
```

# Credits

Both [gpb](https://github.com/tomas-abrahamsson/gpb) and [exprotobuf](https://github.com/bitwalker/exprotobuf) were very useful in understanding how to implement Protocol Buffers.
