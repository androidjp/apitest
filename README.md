**This library is on version 0 and we won't guarantee backwards compatible API changes until we go to version 1. The roadmap for version 1 includes mocking external http calls and sequence diagram generation.**

# api-test

[![GoDoc](https://godoc.org/github.com/steinfletcher/api-test?status.svg)](https://godoc.org/github.com/steinfletcher/api-test)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/023e062d720847e08c1065cbb65a4068)](https://app.codacy.com/app/steinfletcher/api-test?utm_source=github.com&utm_medium=referral&utm_content=steinfletcher/api-test&utm_campaign=Badge_Grade_Dashboard)
[![Build Status](https://travis-ci.org/steinfletcher/api-test.svg?branch=master)](https://travis-ci.org/steinfletcher/api-test) [![Coverage Status](https://coveralls.io/repos/github/steinfletcher/api-test/badge.svg?branch=master&service=github)](https://coveralls.io/github/steinfletcher/api-test?branch=master)

Simple behavioural (black box) api testing library. 

In black box tests the internal structure of the app is not know by the tests. Data is input to the system and the outputs are expected to meet certain conditions.

**This library is dependency free and we intend to keep it that way**

## Installation

```bash
go get -u github.com/steinfletcher/api-test
```

## Examples

### Framework and library integration examples

| Example                                                                           | Comment                                 |
| --------------------------------------------------------------------------------- | --------------------------------------- |
| [gin](https://github.com/steinfletcher/api-test/tree/master/examples/gin)         | popular martini-like web framework      |
| [gorilla](https://github.com/steinfletcher/api-test/tree/master/examples/gorilla) | the gorilla web toolkit                 |
| [iris](https://github.com/steinfletcher/api-test/tree/master/examples/iris)       | iris web framework                      |
| [mocks](https://github.com/steinfletcher/api-test/tree/master/examples/mocks)     | example mocking out external http calls |

### Companion libraries

| Library                                                        | Comment                   |
| -------------------------------------------------------------- | ------------------------- |
| [JSONPath](https://github.com/steinfletcher/api-test-jsonpath) | JSONPath assertion addons |

### Code snippets

#### JSON body matcher

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/user/1234").
		Expect(t).
		Body(`{"id": "1234", "name": "Tate"}`).
		Status(http.StatusCreated).
		End()
}
```

#### JSONPath

For asserting on parts of the response body JSONPath may be used. A separate module must be installed which provides these assertions - `go get -u github.com/steinfletcher/api-test-jsonpath`. This is packaged separately to keep this library dependency free.

Given the response is `{"a": 12345, "b": [{"key": "c", "value": "result"}]}`

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		Expect(t).
		Assert(jsonpath.Contains(`$.b[? @.key=="c"].value`, "result")).
		End()
}
```

and `jsonpath.Equals` checks for value equality

```go
func TestApi(t *testing.T) {
	apitest.New(handler).
		Get("/hello").
		Expect(t).
		Assert(jsonpath.Equal(`$.a`, float64(12345))).
		End()
}
```

#### Custom assert functions

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		Expect(t).
		Assert(func(res *http.Response, req *http.Request) {
			assert.Equal(t, http.StatusOK, res.StatusCode)
		}).
		End()
}
```

#### Assert cookies

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Patch("/hello").
		Expect(t).
		Status(http.StatusOK).
		Cookies(apitest.Cookie"ABC").Value("12345")).
		CookiePresent("Session-Token").
		CookieNotPresent("XXX").
        	Cookies(
			apitest.Cookie("ABC").Value("12345"),
			apitest.Cookie("DEF").Value("67890")).
		End()
}
```

#### Assert headers

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		Expect(t).
		Status(http.StatusOK).
		Headers(map[string]string{"ABC": "12345"}).
		End()
}
```

#### Mocking external http calls

```go
var getUser = apitest.NewMock().
	Get("/user/12345").
	RespondWith().
	Body(`{"name": "jon", "id": "1234"}`).
	Status(http.StatusOK).
	End()

var getPreferences = apitest.NewMock().
	Get("/preferences/12345").
	RespondWith().
	Body(`{"is_contactable": true}`).
	Status(http.StatusOK).
	End()

func TestApi(t *testing.T) {
	apitest.New().
		Mocks(getUser, getPreferences).
		Handler(handler).
		Get("/hello").
		Expect(t).
		Status(http.StatusOK).
		Body(`{"name": "jon", "id": "1234"}`).
		End()
}
```

#### Debugging http requests and responses generated by api test and any mocks

```go
func TestApi(t *testing.T) {
	apitest.New().
		Debug().
		Handler(handler).
		Get("/hello").
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Provide basic auth in the request

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		BasicAuth("username:password").
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Provide cookies in the request

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		Cookies(apitest.Cookie("ABC").Value("12345")).
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Provide headers in the request

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Delete("/hello").
		Headers(map[string]string{"My-Header": "12345"}).
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Provide query parameters in the request

`Query` can be used in combination with `QueryCollection`

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		Query(map[string]string{"a": "b"}).
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Provide query parameters in collection format in the request

Providing `{"a": {"b", "c", "d"}` results in parameters encoded as `a=b&a=c&a=d`.
`QueryCollection` can be used in combination with `Query`

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Get("/hello").
		QueryCollection(map[string][]string{"a": {"b", "c", "d"}}).
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Capture the request and response data

```go
func TestApi(t *testing.T) {
	apitest.New().
		Observe(func(res *http.Response, req *http.Request) {
			// do something with res and req
    	}).
		Handler(handler).
		Get("/hello").
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

#### Intercept the request

This is useful for mutating the request before it is sent to the system under test.

```go
func TestApi(t *testing.T) {
	apitest.New().
		Handler(handler).
		Intercept(func(req *http.Request) {
			req.URL.RawQuery = "a[]=xxx&a[]=yyy"
		}).
		Get("/hello").
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

## Contributing

View the [contributing guide](CONTRIBUTING.md).