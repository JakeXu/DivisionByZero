# JSON

```go
package main

import (
   "encoding/json"
   "fmt"
)

func main() {
   type ColorGroup struct {
      ID     int
      Name   string
      Colors []string
   }
   group := ColorGroup{
      ID:     1,
      Name:   "Reds",
      Colors: []string{"Crimson", "Red", "Ruby", "Maroon"},
   }
   b, err := json.Marshal(group)
   if err != nil {
      fmt.Println("error:", err)
   }

   fmt.Println(string(b))

   var u ColorGroup

   err = json.Unmarshal(b, &u)

   fmt.Println("ID", u.ID, "Name", u.Name, "Color", u.Colors)
}
```

执行结果

```
{"ID":1,"Name":"Reds","Colors":["Crimson","Red","Ruby","Maroon"]}
ID 1 Name Reds Color [Crimson Red Ruby Maroon]
```

# XML

```go
package main

import (
   "encoding/xml"
   "fmt"
)

func main() {
   type Address struct {
      City, State string
   }
   type Person struct {
      XMLName   xml.Name `xml:"person"`
      Id        int      `xml:"id,attr"`
      FirstName string   `xml:"name>first"`
      LastName  string   `xml:"name>last"`
      Age       int      `xml:"age"`
      Height    float32  `xml:"height,omitempty"`
      Married   bool
      Address
      Comment   string `xml:",comment"`
   }

   v := &Person{Id: 13, FirstName: "John", LastName: "Doe", Age: 42}
   v.Comment = " Need more details. "
   v.Address = Address{"Hanga Roa", "Easter Island"}

   output, err := xml.MarshalIndent(v, "  ", "    ")
   if err != nil {
      fmt.Printf("error: %v\n", err)
   }

   fmt.Println(string(output))

   var u Person
   xml.Unmarshal(output, &u)
   fmt.Println(u.FirstName)
}
```

# PB

需要先给protoc安装go语言的扩展

```
go get -u github.com/golang/protobuf/protoc-gen-go
go install github.com/golang/protobuf/protoc-gen-go
```

```
// test.proto
package example;

enum FOO { X = 17; };

message Test {
  required string label = 1;
  optional int32 type = 2 [default=77];
  repeated int64 reps = 3;
  optional group OptionalGroup = 4 {
    required string RequiredField = 5;
  }
}
```

生成相关的go文件

```shell
protoc --go_out=. test.proto
```

```go
package main

import (
	"log"
	"github.com/golang/protobuf/proto"
	"./example"
	"fmt"
)

func main() {
	test := &example.Test{
		Label: proto.String("hello"),
		Type:  proto.Int32(17),
		Reps:  []int64{1, 2, 3},
		Optionalgroup: &example.Test_OptionalGroup{
			RequiredField: proto.String("good bye"),
		},
	}
	data, err := proto.Marshal(test)
	if err != nil {
		fmt.Println(err.Error())
	}

	fmt.Println(len(data))

	newTest := &example.Test{}
	err = proto.Unmarshal(data, newTest)
	if err != nil {
		log.Fatal("unmarshaling error: ", err)
	}
	// Now test and newTest contain the same data.
	if test.GetLabel() != newTest.GetLabel() {
		log.Fatalf("data mismatch %q != %q", test.GetLabel(), newTest.GetLabel())
	}
	// etc.
	fmt.Println(*newTest.Label)
}
```

# 参考文档

[json](https://golang.org/pkg/encoding/json/)

[xml](https://golang.org/pkg/encoding/xml/)

[pb](https://github.com/golang/protobuf)