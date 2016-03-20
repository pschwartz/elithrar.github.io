---
layout: post
title: "Testing Your (HTTP) Handlers in Go"
categories: golang, web, testing, http
---

You're building a web (HTTP) service in Go, and you want to unit test your handler functions. You've
got a grip on Go's [`net/http`](https://golang.org/pkg/net/http/) package, but you're not sure where
to start with testing that your handlers return the correct HTTP status codes, HTTP headers or
response bodies.

Let's walk through how you go about this, injecting the necessary dependencies, and mocking the
rest.

## A Basic Handler

We'll start by testing a basic test: we want to make sure our handler returns a HTTP 200 (OK) status code. This is our handler:

```go
// handlers.go
package handlers

// e.g. http.HandleFunc("/health-check", HealthCheckHandler)
func HealthCheckHandler(w http.ResponseWriter, r *http.Request) {
    // A very simple health check.
    w.WriteHeader(http.StatusOK)
    w.Header().Set("Content-Type", "application/json")

    // In the future we could report back on the status of our DB, or our cache 
    // (e.g. Redis) by performing a simple PING, and reporting back on each.
    w.Write(`{"alive": true}`)
}

```

And this is our test:

```go
// handlers_test.go
package handlers

import (
    "net/http"
    "testing"
)

func TestHealthCheckHandler(t *testing.T) {
    // Create a request to pass to our handler. We don't have any query parameters for now, so we'll
    // pass 'nil' as the third parameter.
    req, err := http.NewRequest("GET", "/health-check", nil)
    if err != nil {
        t.Fatal(err)
    }

    // We create a ResponseRecorder (which satisfies http.ResponseWriter) to record the response.
    rr := httptest.NewResponseRecorder()
    handler := HealthCheckHandler

    // Our handlers satisfy http.Handler, so we can call their ServeHTTP method 
    // directly and pass in our Request and ResponseRecorder.
    handler.ServeHTTP(rr, req)

    // Check the status code is what we expect.
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            rr.Code, http.StatusOK)
    }

    // Check the response body is what we expect.
    expected := `{"alive": true}`
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v",
            rr.Body.String(), expected)
    }
}
```

As you can see, Go's [testing](https://golang.org/pkg/testing/) and
[httptest](https://golang.org/pkg/net/http/httptest/) packages make testing our handlers extremely
simple. We construct a `*http.Request`, a `*httptest.ResponseRecorder`, and then check how our
handler has responded: status code, body, etc.

If our handler also expected specific query parameters or looked for certain 
headers, we could also test those:

```go
    // e.g. GET /api/projects?page=1&per_page=100
    req, err := http.NewRequest("GET", "/api/projects",
        // Note: url.Values is a map[string][]string
        url.Values{"page": {"1"}, "per_page": {"100"}})
    if err != nil {
        t.Fatal(err)
    }

    // Our handler might also expect an API key.
    r.Header.Set("Authorization", "Bearer abc123")

    // Then: call handler.ServeHTTP(rr, req) like in our first example.
```

Further, if you want to test that a handler (or middleware) is mutating the request in a particular
way, you can define an anonymous function inside your test and capture variables from within by
declaring them in the outer scope. 

```go
    // Declare it outside the anonymous function
    var token string
    testHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
        // Note: Use the assignment operator '=' and not the initialize-and-assign 
        // ':=' operator so we don't shadow our token variable above.
        token = GetToken(r)
        // We'll also set a header on the response as a trivial example of 
        // inspecting headers.
        w.Header().Set("Content-Type", "application/json")
    })

    // Check the status, body, etc.

    if token != expectedToken {
        t.Errorf("token does not match: got %v want %v", token, expectedToken)
    }

    if ctype := rr.Header().Get("Content-Type"); ctype != "application/json") {
        t.Errorf("content type header does not match: got %v want %v",
            ctype, "application/json")
    }
```

**Tip**: make strings like `application/json` or `Content-Type` package-level constants, so you don't
have to type (or typo) them over and over. A typo in your tests can cause unintended behaviour,
becasue you're not testing what you think you are.

You should also make sure to test not just for success, but for failure too: test that your handlers
return errors when they should (e.g. a HTTP 403, or a HTTP 500).

## Populating context.Context in Tests

What about when our handlers are expecting data to be passed to them in a
[`context.Context`](https://godoc.org/golang.org/x/net/context)? How we can create a context and
populate it with (e.g.) an auth token and/or our User type?

> Assumption: you have a custom handler type that provides a `ServeHTTPC(context.Context,
> http.ResponseWriter, *http.Request)`. Go 1.7 will [add context.Context to
> http.Request](https://github.com/golang/go/issues/14660), which will make this even easier.


I use the [Goji](https://goji.io/) mux/router for `context.Context` compatible handlers today, but
this approach will work for any router/mux/framework that works with `context.Context`.

```go
func TestGetProjectsHandler(t *testing.T) {
    req, err := http.NewRequest("GET", "/api/users", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewResponseRecorder()
    // e.g. func GetUsersHandler(ctx context.Context, w http.ResponseWriter, r *http.Request)
    handler := goji.HandlerFunc(GetUsersHandler)

    // Create a new context.Context and populate it with data.
    ctx = context.Background()
    ctx = context.WithValue(ctx, "app.auth.token", "abc123")
    ctx = context.WithValue(ctx, "app.user",
        &YourUser{ID: "qejqjq", Email: "user@example.com"})

    // Pass in our context, *http.Request and ResponseRecorder.
    handler.ServeHTTPC(ctx, rr, req)

    // Check the status code is what we expect.
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            rr.Code, http.StatusOK)
    }

    // We could also test that our handler correctly mutates our context.Context:
    // this is useful if our handler is a piece of middleware.
    if id , ok := ctx.Value("app.req.id"); !ok {
        t.Errorf("handler did not populate the request ID: got %v", id)
    }
}
```

## Mocking Database Calls

Our handlers expect that we pass them a `datastore.ProjectStore` (an interface type) with three
methods (Create, Get, Delete). We want to stub this for testing so that we can test that our
handlers (endpoints) return the correct status codes.

> You should read [this Thoughtbot
> article](https://robots.thoughtbot.com/interface-with-your-database-in-go) and [this article from
> Alex Edwards](http://www.alexedwards.net/blog/organising-database-access#using-an-interface) if
> you're looking to use interfaces to abstract access to your database.

```go
// handlers_test.go
package handlers

// Throws errors on all of its methods.
type badProjectStore struct {
    // This would be a concrete type that satisfies datastore.ProjectStore.
    // We embed it here so that our goodProjectStub type has all the methods
    // needed to satisfy datastore.ProjectStore, without having to stub out
    // every method (we might not want to test all of them, or some might be
    // not need to be stubbed.
    *datastore.Project
}

func (ps *projectStoreStub) CreateProject(project *datastore.Project) error {
    return datastore.NetworkError{errors.New("Bad connection"}
}

func (ps *projectStoreStub) GetProject(id string) (*datastore.Project, error) {
    return nil, datastore.NetworkError{errors.New("Bad connection"}
}

func TestGetProjectsHandlerError(t *testing.T) {
    var store datastore.ProjectStore = &badProjectStore{}

    // We inject our environment into our handlers.
    // Ref: http://elithrar.github.io/article/http-handler-error-handling-revisited/
    env := handlers.Env{Store: store, Key: "abc"}

    req, err := http.NewRequest("GET", "/api/projects", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewResponseRecorder()
    // Handler is a custom handler type that accepts an env and a http.Handler
    // GetProjectsHandler here calls GetProject, and should raise a HTTP 500 if
    // it fails.
    handler := Handler{env, GetProjectsHandler)
    handler.ServeHTTP(rr, req)

    // We're now checking that our handler throws an error (a HTTP 500) when it
    // should.
    if status := rr.Code; status != http.StatusInternalServeError {
        t.Errorf("handler returned wrong status code: got %v want %v"
            rr.Code, http.StatusOK)
    }

    // We'll also check that it returns a JSON body with the expected error.
    expected := []byte(`{"status": 500, "error": "Bad connection"}`)
    if !bytes.Equals(rr.Body.Bytes(), expected) {
        t.Errorf("handler returned unexpected body: got %v want %v",
        rr.Body.Bytes(), expected)
    }
```

This was a slightly more complex example—but highlights how we might:

* Stub out our database implementation: the unit tests in `package handlers` should not need a
  test database.
* Create a stub that intentionally throws errors, so we can test that our
  handlers throw the right status code (e.g. a HTTP 500) and/or write the
  expected response.
* Create a 'good' stub that returns a (static) `*datastore.Project` and test
  that (for example) we can marshal it as JSON. This would catch the case where
  changes to the upstream type might cause it to be incompatible with
  `encoding/json`.

## What Next?

This is by no means an exhaustive guide, but it should get you started. If you're stuck building a
more complex example, then ask over on the [Gophers Slack
community](https://gophersinvite.herokuapp.com/), or take a look at [the packages that import
httptest](https://godoc.org/net/http/httptest?importers) via GoDoc.
