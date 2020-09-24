# purescript-protobuf

[![Test](https://github.com/xc-jp/purescript-protobuf/workflows/Test/badge.svg?branch=master)](https://github.com/xc-jp/purescript-protobuf/actions)
[![Pursuit](http://pursuit.purescript.org/packages/purescript-protobuf/badge)](http://pursuit.purescript.org/packages/purescript-protobuf/)

Purescript library and code generator for
[Google Protocol Buffers version 3](https://developers.google.com/protocol-buffers/docs/proto3).

This library operates on
[`ArrayBuffer`](https://pursuit.purescript.org/packages/purescript-arraybuffer-types/docs/Data.ArrayBuffer.Types#t:ArrayBuffer), so it will run both
[in *Node.js*](https://pursuit.purescript.org/packages/purescript-node-buffer/docs/Node.Buffer.Class)
and in browser environments.

## Code Generation

The `shell.nix` environment provides

* The Purescript toolchain
* The [`protoc`](https://github.com/protocolbuffers/protobuf/blob/master/src/README.md) compiler
* The `protoc-gen-purescript` executable plugin for `protoc` on the `PATH` so that
  [`protoc` can find it](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.compiler.plugin).

```
$ nix-shell

Purescript Protobuf development environment.
To build purescript-protobuf, run:

    npm install
    spago build

To test purescript-protobuf, run:

    protoc --purescript_out=./test/generated test/*.proto
    spago -x test.dhall build
    spago -x test.dhall test

To generate Purescript .purs files from .proto files, run:

    protoc --purescript_out=path_to_output file.proto

[nix-shell]$
```

## Writing programs with the generated code

*None of the modules in this package should be imported directly in our program.*

Rather, we'll import the message modules from the generated `.purs` files,
as well as modules for reading and writing `ArrayBuffer`s.

For example, a message in a `.proto` file declared as

```
message MyMessage {
  sint32 my_field = 1;
}
```

will export these four names in the generated `.purs` modules.

1. A message record type
   * ```purescript
     type MyMessageR = { my_field :: Maybe Int }
     ```
2. A message data type
   * ```purescript
     newtype MyMessage = MyMessage MyMessageR
     ```
3. A message encoder which works with
   [__purescript-arraybuffer-builder__](http://pursuit.purescript.org/packages/purescript-arraybuffer-builder/)
   * ```purescript
     putMyMessage :: forall m. MonadEffect m => MyMessage -> PutM m Unit
     ```
4. A message decoder which works with
   [__purescript-parsing-dataview__](http://pursuit.purescript.org/packages/purescript-parsing-dataview/)
   * ```purescript
     parseMyMessage :: forall m. MonadEffect m => Int -> ParserT DataView m MyMessage
     ```
   The message decoder needs an argument which tells it the
   length of the message which it’s about to decode, because
   [“the Protocol Buffer wire format is not self-delimiting.”](https://developers.google.com/protocol-buffers/docs/techniques#streaming)

Then, in our program, our imports will look something like this.


```purescript
import Generated.Module (MyMessage(..), putMyMessage, parseMyMessage)
import Text.Parsing.Parser (runParserT)
import Data.ArrayBuffer.Builder (execPutM)
```

The generated code modules will import modules from this
package.

The generated code depends on packages

```
  , "protobuf"
  , "arraybuffer"
  , "arraybuffer-types"
  , "arraybuffer-builder"
  , "parsing"
  , "parsing-dataview"
  , "uint"
  , "long"
  , "text-encoding"
```

which are in
[__package-sets__](https://github.com/purescript/package-sets),
except for
[__purescript-longs__](https://pursuit.purescript.org/packages/purescript-longs)
(see `spago.dhall` in this package for the particulars).

It also depends on the Javascript package
[__long__](https://www.npmjs.com/package/long).

### Generated message instances

We cannot easily derive common instances like `Eq` for the
generated message types because
1. The types [might be recursive](https://github.com/purescript/documentation/blob/master/errors/CycleInDeclaration.md).
2. The types might contain fields of type
   [`ArrayBuffer`](https://pursuit.purescript.org/packages/purescript-arraybuffer-types/docs/Data.ArrayBuffer.Types#t:ArrayBuffer),
   which doesn't have those instances.

All of the generated message types have an instance of
[`Generic`](https://pursuit.purescript.org/packages/purescript-generics-rep/docs/Data.Generic.Rep#t:Generic).
This allows us to sometimes use
[`genericEq`](https://pursuit.purescript.org/packages/purescript-generics-rep/docs/Data.Generic.Rep.Eq#v:genericEq)
and
[`genericShow`](https://pursuit.purescript.org/packages/purescript-generics-rep/docs/Data.Generic.Rep.Show#v:genericShow)
on a generated message, if the generated message has those instances for
all of its fields.

All of the generated message types have an instance of
[`NewType`](https://pursuit.purescript.org/packages/purescript-newtype/docs/Data.Newtype#t:Newtype).

### Examples

The __purescript-protobuf__ repository contains three executable *Node.js*
programs which use code generated by __purescript-protobuf__. Refer to these
for further examples of how to use the generated code.

1. The `protoc`
   [compiler plugin](https://github.com/xc-jp/purescript-protobuf/blob/master/src/ProtocPlugin/Main.purs).
   The code generator imports generated code. Trippy, right? This program
   literally writes itself.
2. The
   [unit test suite](https://github.com/xc-jp/purescript-protobuf/blob/master/test/Main.purs)
3. The Google
   [conformance test program](https://github.com/xc-jp/purescript-protobuf/blob/master/conformance/Main.purs)

### Interpreting invalid encoding parse failures

When the decode parser encounters an invalid encoding in the protobuf input
stream then it will fail to parse.

When
[`Text.Parsing.Parser.ParserT`](https://pursuit.purescript.org/packages/purescript-parsing/docs/Text.Parsing.Parser#t:ParserT)
fails it will return a `ParseError String (Position {line::Int,column::Int})`.

The byte offset at which the parse failure occured is given by the
formula `column - 1`.

The path to the protobuf definition which failed to parse will be included
in the `ParseError String` and delimited by `'/'`, something
like `"Message1 / string_field_1 / Invalid UTF8 encoding."`.

## Features

We aim to support binary-encoded (not JSON-encoded)
[__proto3__](https://developers.google.com/protocol-buffers/docs/proto3).
Many __proto2__-syntax descriptor files will
also work, as long as they don't use __proto2__ features.

We don't support
[extensions](https://developers.google.com/protocol-buffers/docs/proto?hl=en#extensions).

The generated optional record fields will use `Nothing` instead of the
[default values](https://developers.google.com/protocol-buffers/docs/proto3?hl=en#default).

We do not preserve
[unknown fields](https://developers.google.com/protocol-buffers/docs/proto3?hl=en#unknowns).

We do not support
[services](https://developers.google.com/protocol-buffers/docs/proto3?hl=en#services).

### Conformance

At the time of this writing, we pass 193 out of 194 of the
[Google conformance tests](https://github.com/protocolbuffers/protobuf/tree/master/conformance)
for binary-encoded proto3.
The one test we fail is
the *Required.Proto3.ProtobufInput.UnknownVarint.ProtobufOutput* test, which
is the test for preserving unknown fields, which we do not support, see above.

See the `conformance/README.md` in this repository for details.

### Imports

The code generator will use the `package` statement in the `.proto` file
and the base file name as the Purescript module name for that file.

The Protobuf
[`import`](https://developers.google.com/protocol-buffers/docs/proto3#importing_definitions)
statement allows Protobuf messages to have fields
consisting of Protobuf messages imported from another file, and qualified
by the package name in that file. In order to generate
the correct Purescript module name qualifier on the types of imported message
fields, the code generator must be able to lookup the package name
statement in the imported file.

For that reason, we can only use top-level
(not [nested](https://developers.google.com/protocol-buffers/docs/proto3#nested))
`message` and `enum` types from an `import`.

The generated Purescript code will usually have module imports which cause
the `purs` compiler to emit warnings. Sorry.

## Performance

The implementation is simple and straightforward. We haven't done
any special optimizations. For example, when encoding a protobuf varint, we
allocate a list of new one-byte `ArrayBuffers`s and then copy them all into
position in the final `ArrayBuffer`. For another example, when decoding a
packed field of numbers, we build a list of the numbers, and then copy them
all into the final `Array`. Also, this whole library is very stack-unsafe.
This may all be improved in later versions.

## Contributing

Pull requests welcome.
