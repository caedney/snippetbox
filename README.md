# Let's Go!

## Go Run

During development the `go run` command is a convenient way to try out your code. It’s essentially a shortcut that compiles your code, creates an executable binary in your `/tmp` directory, and then runs this binary in one step.

It accepts either a space-separated list of `.go` files, the path to a specific package (where the . character represents your current directory), or the full module path. For our application at the moment, the three following commands are all equivalent:

```sh
go run .
go run main.go
go run github.com/caedney/lets-go
```

## Route Patterns

### Trailing slashes

When a route pattern ends with a trailing slash — like `/` or `/static/` — it is known as a _subtree path pattern_. Subtree paths as acting a bit like they have a wildcard at the end, like `/**` or `/static/**`

### Restricting subtree paths

To prevent subtree path patterns from acting like they have a wildcard at the end, you can append the special character sequence `{$}` to the end of the pattern — like `/{$}` or `/static/{$}`. This effectively means match a single slash, followed by nothing else.

### Wildcards

Wildcard segments in a route pattern are denoted by an wildcard identifier inside `{}` brackets. Like this:

```go
mux.HandleFunc("/products/{category}/item/{itemID}", exampleHandler)
```

When defining a route pattern, each path segment (the bit between forward slash characters) can only contain one wildcard and the wildcard needs to fill the whole path segment. Patterns like `/products/c_{category}`, `/date/{y}-{m}-{d}` or `/{slug}.html` are not valid.

Inside your handler, you can retrieve the corresponding value for a wildcard segment using its identifier and the `r.PathValue()` method. For example:

```go
func exampleHandler(w http.ResponseWriter, r *http.Request) {
    category := r.PathValue("category")
    itemID := r.PathValue("itemID")
}
```

### Wildcard Conflicts

When defining route patterns with wildcard segments, it’s possible that some of your patterns will overlap. For example, if you define routes with the patterns `/post/edit` and `/post/{id}` they overlap because an incoming HTTP request with the path `/post/edit` is a valid match for _both_ patterns.

The rule for this is very neat and succinct: _the most specific route pattern wins_. Formally, Go defines a pattern as more specific than another if it matches only a subset of requests that the other pattern matches.

Continuing with the example above, the route pattern `/post/edit` only matches requests with the exact path `/post/edit`, whereas the pattern `/post/{id}` matches requests with the path `/post/edit`, `/post/123`, `/post/abc` and many more. Therefore `/post/edit` is the more specific route pattern and will take precedent.

#### Notes

-   You can register patterns in any order _and it won’t change how the servemux_ behaves.
-   The patterns `/post/new/{id}` and `/post/{author}/latest` overlap because they both match the request path `/post/new/latest`. In this scenario, Go’s servemux considers the patterns to conflict, and will panic at runtime when initializing the routes.

### Remainder Wildcards

If you declare a route pattern like `/post/{path...}` it will match requests like `/post/a`, `/post/a/b`, `/post/a/b/c` and so on — very much like a subtree path pattern does. But the difference is that you can access the entire wildcard part via the `r.PathValue()` method in your handlers. In this example, you could get the wildcard value for `{path...}` by calling `r.PathValue("path")`.

## Headers

```go
// Add to header map
w.Header().Add("Server", "Go")
// Set content type
w.Header().Set("Content-Type", "application/json")
// Set a new cache-control header. If an existing "Cache-Control" header
// exists it will be overwritten.
w.Header().Set("Cache-Control", "public, max-age=31536000")
// In contrast, the Add() method appends a new "Cache-Control" header and can
// be called multiple times.
w.Header().Add("Cache-Control", "public")
w.Header().Add("Cache-Control", "max-age=31536000")
// Delete all values for the "Cache-Control" header.
w.Header().Del("Cache-Control")
// Retrieve the first value for the "Cache-Control" header.
w.Header().Get("Cache-Control")
// Retrieve a slice of all values for the "Cache-Control" header.
w.Header().Values("Cache-Control")
```

### Header canonicalization

When you’re using the `Set()`, `Add()`, `Del()`, `Get()` and `Values()` methods on the header map, the header name will always be canonicalized using the `textproto.CanonicalMIMEHeaderKey()` function. This converts the first letter and any letter following a hyphen to upper case, and the rest of the letters to lowercase. This has the practical implication that when calling these methods the header name is case-insensitive.

f you need to avoid this canonicalization behavior, you can edit the underlying header map directly. It has the type `map[string][]string` behind the scenes. For example:

```go
w.Header()["X-XSS-Protection"] = []string{"1; mode=block"}
```

#### Note

If a HTTP/2 connection is being used, Go will always automatically convert the header names and values to lowercase for you when writing the response, as per [the HTTP/2 specifications](https://tools.ietf.org/html/rfc7540#section-8.1.2).
