# GraphQL alternative
- seamless interesting with rest api framework (using middleware)
- สามารถเลือกได้เฉพาะ fields ที่ต้องการ
- ไอเดียคือ client ส่ง list ของ fields ที่อยากได้มา แล้วทำการ filter json ให้

## Possible design

- HTTP custom header
- Convert every request to POST method + fields list
- Should able to config Header vs Payload
- Performance: custom JSON marshal to github/pkg/json Marshal + filter

## MVP:
- [ ]  Json Marshal filter fields
- [ ]  สร้างการเชื่อมต่อที่ไม่มีรอยต่อกับ REST API framework โดยใช้ middleware
- [ ]  สร้างระบบที่สามารถเลือก fields ที่ต้องการได้
- [ ]  ออกแบบระบบที่สามารถใช้ HTTP custom header หรือแปลงทุกคำขอเป็น POST method พร้อมกับรายการ fields
- [ ]  สามารถกำหนดค่า Header และ Payload ได้
- [ ]  ปรับปรุงประสิทธิภาพด้วยการใช้ custom JSON marshal ไปยัง github/pkg/json Marshal และ filter

## First Design
skill struct in Go
```go
type Skill struct {
	ID          string `json:"id" bson:"_id,omitempty"`
	Name        string             `json:"name" bson:"name"`
	Kind        Kind               `json:"kind" bson:"kind"`
	Description string             `json:"description" bson:"description"`
	Logo        string             `json:"logo" bson:"logo"`
}
```

skill json response from REST API
```json
{
	"id": "1",
	"name": "Go",
	"kind": "Programming",
	"description": "Go programming language",
	"logo": "https://golang.org/doc/gopher/frontpage.png"
}
```

post-middleware will filter feilds base on "fields" key in request body by using custom JSON marshal filter on top of https://github.com/pkg/json Marshal
to make sure performance is not affected
```json
{
	"fields": ["id", "name"]
}
```

actual response to client
```json
{
	"id": "1",
	"name": "Go"
}
```

## Implementation First Design
1. [ ]  Implement custom JSON marshal filter
2. [ ]  Implement middleware to filter fields

### 1. custom JSON marshal filter
input: schema: `[ "id", "name" ]`, json: `{"id": "1", "name": "Go", "kind": "Programming", "description": "Go programming language", "logo": "https://golang.org/doc/gopher/frontpage.png"}`
output: `{"id": "1", "name": "Go"}`


v1
```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"strings"
)

type Skill struct {
	ID          string `json:"id" bson:"_id,omitempty"`
	Name        string             `json:"name" bson:"name"`
	Kind        Kind               `json:"kind" bson:"kind"`
	Description string             `json:"description" bson:"description"`
	Logo        string             `json:"logo" bson:"logo"`
}

type Kind string

const (
	Programming Kind = "Programming"
	Soft        Kind = "Soft"
	Hard        Kind = "Hard"
)

func filterJSONMarshal(schema []string, jsonData []byte) ([]byte, error) {
	// Create a map to store the allowed fields
	allowedFields := make(map[string]bool)
	for _, field := range schema {
		allowedFields[field] = true
	}

	// Decode the JSON data into a temporary struct
	var tempSkill Skill
	if err := json.Unmarshal(jsonData, &tempSkill); err != nil {
		return nil, err
	}

	// Create a new map to store the filtered data
	filteredData := make(map[string]interface{})
	for key, value := range tempSkill {
		if allowedFields[key] {
			filteredData[key] = value
		}
	}

	// Marshal the filtered data back into JSON
	filteredJSON, err := json.MarshalIndent(filteredData, "", "  ")
	if err != nil {
		return nil, err
	}

	return filteredJSON, nil
}

func main() {
	// Sample schema and JSON data
	schema := []string{"id", "name"}
	jsonData := []byte(`{"id": "1", "name": "Go", "kind": "Programming", "description": "Go programming language", "logo": "https://golang.org/doc/gopher/frontpage.png"}`)

	// Filter the JSON data using the custom marshal filter
	filteredJSON, err := filterJSONMarshal(schema, jsonData)
	if err != nil {
		panic(err)
	}

	// Print the filtered JSON data
	fmt.Println(string(filteredJSON))
}
```

v2
```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"reflect"
	"strings"
)

// CustomJSONMarshaler interface for custom marshaling
type CustomJSONMarshaler interface {
	MarshalJSONWithFields(fields []string) ([]byte, error)
}

// Skill struct representing the data model
type Skill struct {
	ID          string `json:"id"`
	Name        string `json:"name"`
	Kind        string `json:"kind"`
	Description string `json:"description"`
	Logo        string `json:"logo"`
}

// MarshalJSONWithFields method for custom marshaling
func (s Skill) MarshalJSONWithFields(fields []string) ([]byte, error) {
	filteredFields := make(map[string]interface{}, len(fields))
	v := reflect.ValueOf(s)
	for _, field := range v.NumField() {
		fieldName := v.Type().Field(field).Name
		if contains(fields, fieldName) {
			filteredFields[fieldName] = v.Field(field).Interface()
		}
	}

	return json.MarshalIndent(filteredFields, "", "  ")
}

// contains helper function to check if a string is in a slice
func contains(slice []string, item string) bool {
	for _, s := range slice {
		if s == item {
			return true
		}
	}
	return false
}

// Example usage
func main() {
	// Example data
	schema := []string{"id", "name"}
	jsonData := []byte(`{"id": "1", "name": "Go", "kind": "Programming", "description": "Go programming language", "logo": "https://golang.org/doc/gopher/frontpage.png"}`)

	// Decode JSON into a struct
	var skill Skill
	if err := json.Unmarshal(jsonData, &skill); err != nil {
		panic(err)
	}

	// Marshal JSON with filtered fields
	filteredJSON, err := skill.MarshalJSONWithFields(schema)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(filteredJSON)) // Output: {"id": "1", "name": "Go"}
}
```
