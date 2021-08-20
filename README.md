# thrifter

decode/encode thrift message without IDL

Why?

* because IDL generated model is ugly and inflexible, it is seldom used in application directly. instead we define another model, which leads to bad performance.
  * bytes need to be copied twice 
  * more objects to gc
* thrift proxy can not know all possible IDL in advance, in scenarios like api gateway, we need to decode/encode in a generic way to modify embedded header.
* official thrift library for go is slow, verified in several benchmarks. It is even slower than [json-iterator](https://github.com/json-iterator/go)

# works like encoding/json

`encoding/json` has a super simple api to encode/decode json.
thrifter mimic the same api.

```go
import "github.com/Numenorean/thrift-iterator"
// marshal to thrift
thriftEncodedBytes, err := thrifter.Marshal([]int{1, 2, 3})
// unmarshal back
var val []int
err = thrifter.Unmarshal(thriftEncodedBytes, &val)
```

even struct data binding is supported

```go
import "github.com/Numenorean/thrift-iterator"

type NewOrderRequest struct {
    Lines []NewOrderLine `thrift:",1"`
}

type NewOrderLine struct {
    ProductId string `thrift:",1"`
    Quantity int `thrift:",2"`
}

// marshal to thrift
thriftEncodedBytes, err := thrifter.Marshal(NewOrderRequest{
	Lines: []NewOrderLine{
		{"apple", 1},
		{"orange", 2},
	}
})
// unmarshal back
var val NewOrderRequest
err = thrifter.Unmarshal(thriftEncodedBytes, &val)
```

# without IDL

you do not need to define IDL. you do not need to use static code generation.
you do not event need to define struct.

```go
import "github.com/Numenorean/thrift-iterator"
import "github.com/Numenorean/thrift-iterator/general"

var msg general.Message
err := thrifter.Unmarshal(thriftEncodedBytes, &msg)
// the RPC call method name, type is string
fmt.Println(msg.MessageName)
// the RPC call arguments, type is general.Struct
fmt.Println(msg.Arguments)
```

what is `general.Struct`, it is defined as a map

```go
type FieldId int16
type Struct map[FieldId]interface{}
```

we can extract out specific argument from deeply nested arguments using one line

```go
productId := msg.MessageArgs.Get(
	protocol.FieldId(1), // lines of request
	0, // the first line
	protocol.FieldId(1), // product id
).(string)
```

You can unmarshal any thrift bytes into general objects. And you can marshal them back.

# Partial decoding

fully decoding into a go struct consumes substantial resources. 
thrifter provide option to do partial decoding. You can modify part of the 
message, with untouched parts in `[]byte` form.

```go
import "github.com/Numenorean/thrift-iterator"
import "github.com/Numenorean/thrift-iterator/protocol"
import "github.com/Numenorean/thrift-iterator/raw"

// partial decoding
decoder := thrifter.NewDecoder(reader)
var msgHeader protocol.MessageHeader
decoder.Decode(&msgHeader)
var msgArgs raw.Struct
decoder.Decode(&msgArgs)

// modify...

// encode back
encoder := thrifter.NewEncoder(writer)
encoder.Encode(msgHeader)
encoder.Encode(msgArgs)
```

the definition of `raw.Struct` is 

```go
type StructField struct {
	Buffer []byte
	Type protocol.TType
}

type Struct map[protocol.FieldId]StructField
```

# Performance

thrifter does not compromise performance. 

gogoprotobuf

```
5000000	       366 ns/op	     144 B/op	      12 allocs/op
```

thrift 

```
1000000	      1549 ns/op	     528 B/op	       9 allocs/op
```

thrifter by static codegen

```
5000000	       389 ns/op	     192 B/op	       6 allocs/op
```

thrifter by reflection

```
2000000	       585 ns/op	     192 B/op	       6 allocs/op
```

You can see the reflection implementation is not bad, much faster than the 
static code generated by thrift original implementation.

To have best performance, you can choose to use static code generation. The api
is unchanged, just need to add extra static codegen in your build steps, and include
the generated code in your package. The runtime will automatically use the 
generated encoder/decoder instead of reflection.

For example of static codegen, checkout [https://github.com/Numenorean/thrift-iterator/blob/master/test/api/init.go](https://github.com/Numenorean/thrift-iterator/blob/master/test/api/init.go)

# Sync IDL and Go Struct

Keep IDL and your object model is challenging. We do not always like the code 
generated from thrift IDL. But manually keeping the IDL and model in sync is
tedious and error prone. 

A separate toolchain to manipulate thrift IDL file, and keeping them bidirectionally in sync
will be provided in another project.

