[<img src="http://mholt.github.io/binding/resources/images/binding-sm.png" height="250" alt="binding is reflectionless data binding for Go"></a>](http://mholt.github.io/binding)


binding
=======

Reflectionless data binding for Go's net/http



Features
---------

- HTTP request data binding
- Data validation (custom and built-in)
- Error handling



Benefits
---------

- Moves data binding, validation, and error handling out of your application's handler
- Reads Content-Type to deserialize form, multipart form, and JSON data from requests
- No middleware: just a function call
- Usable in any setting where `net/http` is present (Negroni, gocraft/web, std lib, etc.)
- No reflection



Usage example
--------------

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/mholt/binding"
)

// First define a type to hold the data
// (If the data comes from JSON, see: http://mholt.github.io/json-to-go)
type ContactForm struct {
	User struct {
		ID int
	}
	Email   string
	Message string
}

// Then provide a field mapping (pointer receiver is vital)
func (cf *ContactForm) FieldMap() binding.FieldMap {
	return binding.FieldMap{
		&cf.User.ID: "user_id",
		&cf.Email:   "email",
		&cf.Message: binding.Field{
			Form:     "message",
			Required: true,
		},
	}
}

// Now your handlers can stay clean and simple
func handler(resp http.ResponseWriter, req *http.Request) {
	contactForm := new(ContactForm)
	errs := binding.Bind(req, contactForm)
	if errs.Handle(resp) {
		return
	}
	fmt.Fprintf(resp, "From:    %d\n", contactForm.User.ID)
	fmt.Fprintf(resp, "Message: %s\n", contactForm.Message)
}

func main() {
	http.HandleFunc("/contact", handler)
	http.ListenAndServe(":3000", nil)
}
```


Custom data validation
-----------------------

You may optionally have your type implement the `binding.Validator` interface to perform your own data validation. The `.Validate()` method is called after the struct is populated.

```go
func (cf ContactForm) Validate(req *http.Request, errs binding.Errors) binding.Errors {
	if cf.Message == "Go needs generics" {
		errs = append(errs, binding.Error{
			FieldNames:     []string{"message"},
			Classification: "ComplaintError",
			Message:        "Go has generics. They're called interfaces.",
		})
	}
	return errs
}
```



Error Handling
---------------

`binding.Bind()` and each deserializer returns errors. You don't have to use them, but the `binding.Errors` type comes with a kind of built-in "handler" to write the errors to the response as JSON for you. For example, you might do this in your HTTP handler:

```go
if binding.Bind(req, contactForm).Handle(resp) {
	return
}
```

As you can see, if `.Handle(resp)` wrote errors to the response, your handler may gracefully exit.



Binding custom types
---------------------

For types you've defined, you can bind form data to it by implementing the `Binder` interface. Here's a contrived example:


```go
type MyType map[string]string

func (t *MyType) Bind(fieldName string, strVals []string, errs binding.Errors) binding.Errors {
	t["formData"] = strVals[0]
	return errs
}
```

If you can't add a method to the type, you can still specify a `Binder` func in the field spec. Here's a contrived example that binds an integer (not necessary, but you get the idea):

```go
func (t *MyType) FieldMap() binding.FieldMap {
	return binding.FieldMap{
		"number": binding.Field{
			Binder: func(fieldName string, formVals []string, errs binding.Errors) binding.Errors {
				val, err := strconv.Atoi(formVals[0])
				if err != nil {
					errs.Add([]string{fieldName}, binding.DeserializationError, err.Error())
				}
				t.SomeNumber = val
				return errs
			},
		},
	}
}
```

Notice that the `binding.Errors` type has a convenience method `.Add()` which you can use to append to the slice if you prefer.


Supported types (forms)
------------------------

The following types are supported in form deserialization by default. (JSON requests are delegated to `encoding/json`.)

- uint, []uint, uint8, uint16, uint32, uint64
- int, []int, int8, int16, int32, int64
- float32, []float32, float64, []float64
- bool, []bool
- string, []string
- time.Time, []time.Time
- *multipart.FileHeader, []*multipart.FileHeader