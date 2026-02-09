---
title: 'Sharing a Cache Safely Across HTTP Handlers in Go'
date: '2025-12-17'
hero: ''
---
![cache function image](https://github.com/The-Lennzer/images/blob/main/tech%20images/cache-go.png?raw=true)
I've been working on a URL Shortener in Go and I know it isn't much but I believe it's a solid project to understand the fundaments when working with a new language.

During my initial implementation of a cache I did not think much about it and simply created a map that stored the latest key-value pairs and thought that was enough.

```go
var cache map[string]string = make(map[string]string)
```

Declared globally, I thought this would be more than enough for my simple use case.

But alas, how wrong I was!

One important detail in Go is that **HTTP handlers run concurrently**, and maps are **not safe for concurrent access**. So I had to go about implementing concurrency control in the form of mutexes.

```go
package handlers

import (
	"net/http"
	"sync"
)

func handler(
	w http.ResponseWriter, 
	r *http.Request,
){
	var mutex sync.RWMutex
	
	mutex.Lock()
	//modify map
	mutex.Unlock()
}
```

The above strategy wouldn't work cause each request gets one handler and therefore every request that hits this endpoint gets a new mutex and in result there is no concurrency control - they have to share the same mutex for this to work.

What I did to fix this is create a cache structure which held a map and a mutex inside it.

```go
type Cache struct {
	cacheMap map[string]string
	mu       sync.RWMutex
}
// The type is exported so it can be used from other packages, while the fields remain private.
```

 Then we can add a constructor for easy initialization(you see, in go a map has to be initialized by using the make() function) along with a setter and getter. Perfect!

```go
// constructor
func New() *Cache {
    return &Cache{
        cacheMap: make(map[string]string),
    }
}
// setter
func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.cacheMap[key] = value
}
// getter
func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
  
    value, ok := c.cacheMap[key]
    return value, ok
}

// Pointer receivers are important here to avoid copying the mutex and to ensure all handlers share the same state.
```

Now we need to replace the old map with this structure. In order to do that we go to the function where our server is created and initialize this cache. In my case that would be the *main* function.

```go
import (
	"github.com/segfaultscribe/hpurls/cache" //my cache package
)

func main(){
	c := cache.New() // initialize the cache	
}
```

Now how do we connect this to all the endpoints that need the cache?

A clean way to wire this into the HTTP server is to introduce a `server` struct.

```go
// in the same main package; but outside func main(){}
type server struct {
	cache *cache.Cache
}
```

I'm creating a server structure that'll help us wrap the handler functions that require the cache

```go
import (
	"github.com/segfaultscribe/hpurls/cache" //my cache package
)

type server struct {
	cache *cache.Cache
}

func main(){
	c := cache.New() // initialize the cache	
	
	s := &server{
		cache: c,
	}
	
	mux := http.NewServeMux()
	
	mux.Handle("POST /shorten", s.shortenHandler)  | now the handlers are
    mux.Handle("GET /{short}", s.redirectHandler)  | methods of 's'
}
```

and so we make the handlers methods of server struct

```go
func (s *server) shortenHandler(
	w http.ResponseWriter, 
	r *http.Request,
) { ... }

func (s *server) redirectHandler(
	w http.ResponseWriter, 
	r *http.Request,
) { ... }
```

Now u can safely use `s.cache.Set` to set a key-value pair in the cache and `s.cache.Get` to retrieve a value from the cache without worrying about concurrency issues because those methods handle it inside them.

That's it! 

> ❕ Know that this is not a cache implementation per se, but on how you can use the same cache with multiple handlers without breaking concurrency

*(With tongue firmly in cheek)*<br/>
*I shall assume the esteemed reader of this mini-blog is quite well versed in writing splendid Go code and is a master of the great arts of data structures and can create his/her own implementation of a rather well regarded cache unlike the simple one my poor self has contrived.*



