# JSON and IO (WIP)

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/master/json-and-io)**

[In the previous chapter](http-server.md) we created a web server to store how many games players have won.

Our product-owner was mostly delighted but was somewhat perturbed by the software losing the scores when the server was restarted. This was because our implementation of our store was in-memory.

She also has a new requirement; to have a new endpoint called `/league` which returns a list of all players stored, ordered by wins. She would like this to be returned as JSON.

She says she doesn't have a preference for how we persist the scores, she trusts us - just make it work!

## Here is the code we have so far

```go
// server.go
package main

import (
	"fmt"
	"net/http"
)

// PlayerStore stores score information about players
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

// PlayerServer is a HTTP interface for player information
type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := r.URL.Path[len("/players/"):]

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

```go
// InMemoryPlayerStore.go
package main

// NewInMemoryPlayerStore initialises an empty player store
func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

// InMemoryPlayerStore collects data about players in memory
type InMemoryPlayerStore struct {
	store map[string]int
}

// RecordWin will record a player's win
func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

// GetPlayerScore retrieves scores for a given player
func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}

```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

You can find the corresponding tests in the link at the top of the chapter.

We'll start by making the league table endpoint.

## Write the test first

We'll extend the existing suite as we have some useful test functions and a fake `PlayerStore` to use.

```go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

Before worrying about actual scores and JSON we will try and keep the changes small with the plan to iterate toward our goal. The simplest start is to check we can hit `/league` and get an `OK` back.

## Try to run the test

```
=== RUN   TestLeague/it_returns_200_on_/league
panic: runtime error: slice bounds out of range [recovered]
	panic: runtime error: slice bounds out of range

goroutine 6 [running]:
testing.tRunner.func1(0xc42010c3c0)
	/usr/local/Cellar/go/1.10/libexec/src/testing/testing.go:742 +0x29d
panic(0x1274d60, 0x1438240)
	/usr/local/Cellar/go/1.10/libexec/src/runtime/panic.go:505 +0x229
github.com/quii/learn-go-with-tests/json-and-io/v2.(*PlayerServer).ServeHTTP(0xc420048d30, 0x12fc1c0, 0xc420010940, 0xc420116000)
	/Users/quii/go/src/github.com/quii/learn-go-with-tests/json-and-io/v2/server.go:20 +0xec
```

Your `PlayerServer` should be panicking like this. Go to the line of code in the stack trace which is pointing to `server.go`.

```go
player := r.URL.Path[len("/players/"):]
```

In the previous chapter we mentioned this was a fairly naive way of doing our routing. What is happening is it's trying to split the string of the path starting at an index beyond `/league` so it is `slice bounds out of range`.

## Write enough code to make it pass

Go does have a built in routing mechanism called [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux) (request multiplexer) which lets you attach `http.Handler`s to particular request paths.

Let's commit some sins and get the tests passing in the quickest way we can, knowing we can refactor it with safety once we know the tests are passing.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := r.URL.Path[len("/players/"):]

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- When the request starts we create a router and then we tell it for `x` path use `y` handler.
- So for our new endpoint, we use `http.HandlerFunc` and an _anonymous function_ to `w.WriteHeader(http.StatusOK)` when `/league` is requested to make our new test pass.
- For the `/players/` route we just cut and paste our code into another `http.HandlerFunc`.
- Finally we handle the request that came in by calling our new router's `ServeHTTP` (notice how `ServeMux` is _also_ a `http.Handler`?)

If you run all the tests, it should all be passing.

## Refactor

There's a few improvements we can make.

`ServeHTTP` is looking quite big, we can separate things out a bit by refactoring our handlers into separate methods.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := r.URL.Path[len("/players/"):]

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

Looking better!

Next, it's quite odd (and inefficient) to be setting up a router as a request comes in and then calling it. What we ideally want to do is have some kind of `NewPlayerServer` function which will take our dependencies and do the one time setup of creating the router. Each request can then just use that one instance of the router.

Here are the relevant changes.

```go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` now needs to store a router.
- We have moved the routing creation out of `ServeHTTP` and into our `NewPlayerServer` so this only has to be done once, not per request
- You will need to update all the test and production code where we used to do `PlayerServer{&store}` with `NewPlayerServer(&store)`.

### One final refactor

Try changing the code to the following.

```go
type PlayerServer struct {
	store  PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

Finally make sure you **delete** `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)` as it is no longer needed!

## Embedding

We changed second property of `PlayerServer`, removing the named property `router *http.ServeMux*` and replaced it with `http.Handler`; this is called _embedding_.

> Go does not provide the typical, type-driven notion of subclassing, but it does have the ability to “borrow” pieces of an implementation by embedding types within a struct or interface.

[Effective Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

What this means is that our `PlayerServer` now has all the methods that `http.Handler` has, which is just `ServeHTTP`

To "fill in" the `http.Handler` we assign it to the `router` we create in `NewPlayerServer`. We can do this because `http.ServeMux` has the method `ServeHTTP`

This lets us remove our own `ServeHTTP` method, as we are already exposing one via the embedded type

Embedding is a very interesting language feature. You can use it with interfaces to compose new interfaces.

```go
type Animal interface{
	Eater()
	Sleeper()
}
```

### Any downsides?

You must be careful with embedding types because you will expose all public methods and properties of the type you embed. In our case it is ok because we embedded just the _interface_ that we wanted to expose (`http.Handler`).

If we had been lazy and embedded `http.ServeMux` instead (the concrete type) it would still work _but_ users of `PlayerServer` would be able to add new routes to our server because `Handle(path, handler)` would be public.

**When embedding types, really think about what impact that has on your public API**.

Now we've restructured our application so that we can easily add new routes and have the start of the `/league` endpoint. We now need to make it return some useful information.

We should return some JSON that looks something like this.

```json
[ {"Name": "Bill", "Wins": 10},
  {"Name": "Alice", "Wins": 15} ]
```

## Write the test first

We'll start by just trying to parse the response into something meaningful.

```go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf ("Unable to parse response from server '%s' into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### Why not test the JSON string?

You could argue a simpler initial step would be just to assert that the response body has a particular JSON string.

In my experience tests that assert against JSON strings have the following problems.

- *Brittleness*. If you change the data-model your tests will fail.
- *Poor output*. If your API wont return a prettified JSON string it can be very hard to read.
- *Poor intention*. Whilst the output should be JSON, what's really important is exactly what the data is, rather than how it's encoded.
- *Re-testing the standard library*. There is no need to test how the standard library outputs JSON, it is already tested. Don't test other people's code.

Instead we should look to parse the JSON into data structures that are relevant for us to test with.

### Data modelling

Given the JSON data model, it looks like we need an array of `Player` with some fields so we have created a new type to capture this.

```go
type Player struct {
	Name string
	Wins int
}
```

### JSON decoding

```go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

To parse JSON into our data model we create a `Decoder` from `encoding/json` package and then call its `Decode` method. To create a `Decoder` it needs an `io.Reader` to read from which in our case is our response spy's `Body`.

`Decode` takes the address of the thing we are trying to decode into which is why we declare an empty slice of `Player` the line before.

Parsing JSON can fail so `Decode` can return an `error`. There's no point continuing the test if that fails so we check for the error and stop the test with `t.Fatalf` if it happens. Notice that we print the response body along with the error as it's important for someone running the test to see what string cannot be parsed.

## Try to run the test

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
    	server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

Our endpoint currently does not return a body so it cannot be parsed into JSON.

## Write enough code to make it pass

```go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

The test now passes

### Encoding and Decoding
Notice the lovely symmetry in the standard library

- To create an `Encoder` you need an `io.Writer` which is what `http.ResponseWriter` implements.
- To create a `Decoder` you need an `io.Reader` which the `Body` field of our response spy implements.

Throughout this book we have used `io.Writer` and this is another demonstration of its prevalence in the standard library and how a lot of libraries easily work with it.

## Refactor

It would be nice to introduce a separation of concern between our handler and getting the `leagueTable` as we know we're going to not hard-code that very soon.

```go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player{
	return []Player{
		{"Chris", 20},
	}
}
```

Next we'll want to extend our test so that we can control exactly what data we want back.

## Wrapping up

What we've covered:

- `struct` embedding
- JSON deserializing and serializing
- Routing