# Proposal: Sensu Custom Attributes

Author(s): Eric Chlebek

Last updated: 2017-11-21

Discussion at https://github.com/sensu/sensu-go/issues/586

## Abstract

Allow users to set custom attributes on selected entities in Sensu, and
reference these entities in filters and mutators.

## Background

Custom attributes will necessarily involve dynamic behaviour and reflection,
by their nature. Luckily, we can leverage some existing libraries to take the
burden of maintaining piles of reflection away from us.

In Sensu 1.x, existing custom attributes look like the following:
```
{
  "checks": {
    "check_mysql_replication": {
      "command": "check-mysql-replication-status.rb --user sensu --password secret",
      "subscribers": [
        "mysql"
      ],
      "interval": 30,
      "playbook": "http://docs.example.com/wiki/mysql-replication-playbook"
    }
  }
}
```

Notice the 'playbook' field, which is not normally described by the check data
structure. Sensu allows users to define custom logic that access these custom
attributes.

## Proposal

Having arbitrary fields is something of an anti-pattern in Go, but there are
some things we can do to support them. One, already implemented, is using
`govaluate`, which sets up a number of approaches for doing just this.

Another library we can utilize is `jsoniterator`, which lets users arbitrarily
iterate through JSON data structures instead of unmarshalling in a single shot.

By combining these two libraries, we can implement a custom unmarshaler and
marshaler for any data types that want to use custom attributes.

## Reflection-based encoding and decoding helpers

By writing a few helpers, we can take the brunt of the task away from type
implementers. I've included a small, largely untested proof-of-concept here.

```
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"reflect"
	"strings"

	"github.com/Knetic/govaluate"
	jsoniter "github.com/json-iterator/go"
)

// Check is a simplified Check that doesn't know about protobufs
type Check struct {
	Duration float64 `json:"duration"`
	Executed int64   `json:"executed"`
	Issued   int64   `json:"issued"`
	Output   string  `json:"output"`
	State    string  `json:"state"`
	Status   int32   `json:"status"`

	Custom []byte `json:"-"`
}

// implement govaluate.Parameters interface
func (c Check) Get(name string) (interface{}, error) {
	switch name {
	case "duration":
		return c.Duration, nil
	case "executed":
		return c.Executed, nil
	case "issued":
		return c.Issued, nil
	case "output":
		return c.Output, nil
	case "state":
		return c.State, nil
	case "status":
		return c.Status, nil
	}
	parts := strings.Split(name, ".")
	dynamicParts := make([]interface{}, len(parts))
	for i, p := range parts {
		dynamicParts[i] = p
	}
	any := jsoniter.Get(c.Custom, dynamicParts...)
	return any.GetInterface(), any.LastError()
}

func (c *Check) UnmarshalJSON(b []byte) error {
	// create a temporary type to use default unmarshal behaviour for the
	// standard fields.
	type __ Check
	var x __
	if err := jsoniter.Unmarshal(b, &x); err != nil {
		return err
	}
	*c = Check(x)
	var err error
	c.Custom, err = ExtractCustom(c, b)
	return err
}

type structField struct {
	Field reflect.StructField
	Value reflect.Value
}

func (s structField) JSONFieldName() (string, bool) {
	fieldName := s.Field.Name
	tag, ok := s.Field.Tag.Lookup("json")
	omitEmpty := false
	if ok {
		parts := strings.Split(tag, ",")
		fieldName = parts[0]
		if len(parts) > 1 && parts[1] == "omitempty" {
			omitEmpty = true
		}
	}
	return fieldName, omitEmpty
}

func getFields(v reflect.Value) map[string]structField {
	typ := v.Type()
	numField := v.NumField()
	result := make(map[string]structField, numField)
	for i := 0; i < numField; i++ {
		field := typ.Field(i)
		if len(field.PkgPath) != 0 {
			// unexported
			continue
		}
		value := v.Field(i)
		sf := structField{Field: field, Value: value}
		fieldName, omitEmpty := sf.JSONFieldName()
		if fieldName == "-" || reflect.DeepEqual(reflect.Zero(typ), v.Interface()) && omitEmpty {
			continue
		}
		result[fieldName] = sf
	}
	return result
}

func ExtractCustom(v interface{}, b []byte) ([]byte, error) {
	strukt := reflect.Indirect(reflect.ValueOf(v))
	if kind := strukt.Kind(); kind != reflect.Struct {
		return nil, fmt.Errorf("invalid type passed to EncodeFields: %v", kind)
	}
	fields := getFields(strukt)
	stream := jsoniter.NewStream(jsoniter.ConfigDefault, nil, 4096)
	var anys map[string]jsoniter.Any
	if err := jsoniter.Unmarshal(b, &anys); err != nil {
		return nil, err
	}
	objectStarted := false
	for field, value := range anys {
		if _, ok := fields[field]; ok {
			// Not a custom field
			continue
		}
		if !objectStarted {
			objectStarted = true
			stream.WriteObjectStart()
		} else {
			stream.WriteMore()
		}
		stream.WriteObjectField(field)
		value.WriteTo(stream)
	}
	if objectStarted {
		stream.WriteObjectEnd()
	}
	return stream.Buffer(), nil
}

func (c Check) MarshalJSON() ([]byte, error) {
	s := jsoniter.NewStream(jsoniter.ConfigDefault, nil, 4096)

	s.WriteObjectStart()

	if err := EncodeFields(c, s); err != nil {
		return nil, err
	}

	if len(c.Custom) > 0 {
		if err := EncodeCustomFields(c.Custom, s); err != nil {
			return nil, err
		}
	}

	s.WriteObjectEnd()

	return s.Buffer(), nil
}

func EncodeFields(v interface{}, s *jsoniter.Stream) error {
	strukt := reflect.Indirect(reflect.ValueOf(v))
	if kind := strukt.Kind(); kind != reflect.Struct {
		return fmt.Errorf("invalid type passed to EncodeFields: %v", kind)
	}
	fields := getFields(strukt)
	objectStarted := false
	for fieldName, field := range fields {
		if !objectStarted {
			objectStarted = true
		} else {
			s.WriteMore()
		}
		s.WriteObjectField(fieldName)
		s.WriteVal(field.Value.Interface())
	}
	return nil
}

func EncodeCustomFields(custom []byte, s *jsoniter.Stream) error {
	var anys map[string]jsoniter.Any
	if err := jsoniter.Unmarshal(custom, &anys); err != nil {
		return err
	}
	for name, value := range anys {
		s.WriteMore()
		s.WriteObjectField(name)
		value.WriteTo(s)
	}
	return nil
}

func main() {
	const myCheck = `
	{
		"duration": 1.5,
		"executed": 1,
		"issued": 1,
		"output": "Hello, world!",
		"state": "Happy",
		"status": 0,
		"custom_1": ["foo", 5],
		"custom_2": {"bar": "baz"},
		"custom_3": 10
	}
	`
	var c Check
	if err := json.Unmarshal([]byte(myCheck), &c); err != nil {
		log.Fatalf("can't unmarshal: %s", err)
	}
	fmt.Printf("check: %#v\n\n", c)
	fmt.Printf("custom: %q\n\n", string(c.Custom))
	b, err := json.Marshal(c)
	if err != nil {
		log.Fatalf("can't marshal: %s", err)
	}
	fmt.Printf("marshalled: %s\n\n", string(b))

	expr, err := govaluate.NewEvaluableExpression(`state == "Happy" && custom_3 > 5`)
	if err != nil {
		log.Fatal(err)
	}

	result, err := expr.Eval(c)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(result.(bool))
}
```

If we decide to move forward with the proposal, this code will need to be
cleaned up and tested. Additionally, we will need to work around some issues
with govaluate.

TODO:
1. Support querying nested custom attribute structures
1. Optimize the code by reducing reflection as much as possible
1. Better naming
1. Unit tests

## Govaluate

The problem with govaluate is that it does not allow accessing nested maps or
`Parameters` types. There are two potential ways we can solve this issue:

1. Fork and fix govaluate.
1. Dynamically generate structs from JSON using reflect.

Both solutions will necessitate reflection, and forking govaluate will incur
additional maintenance overhead, especially if we don't manage to get it merged
into mainline.

## Rationale

There are essentially three ways to accomplish this type of dynamic behaviour
at runtime in Go.

1. Reflection
1. Code generation and execution out-of-process
1. Embedded VM, ala lua or v8

I believe that in our case, reflection is probably the design decision that
fits best with the libraries we've already chosen, and would allow us to
implement a solution without relying on embedding a VM or running things
outside the main executable's process.

## Compatibility

This feature is necessary to facilitate 1.x parity.

## Implementation

1. Implement a library for dealing with marshaling and unmarshaling data types
with custom attributes. Make sure the library handles govaluate integration.
1. Make use of the library in the Check data type.
1. Make use of the library in the Environment data type.
