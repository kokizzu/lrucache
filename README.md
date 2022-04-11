# cache

Cache is a thread safe and lockless in memory cache object. This is achieved by partitioning values across many
smaller LRU (least recently used) caches and interacting with those caches over channels. 
Each smaller cache maintains access to its own elements and communicates information back to the Cache object, 
which then responds back to the original caller.

# examples

### Single Cache Partition

```go

package main

import (
	"context"
	"fmt"
	
	"github.com/alistanis/cache"
)

// Example_singleCache is a simple example that is shown with a concurrency of 1 in order to 
// illustrate how the smaller LRU caches work.
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    c := cache.New[int, int](ctx, 5, 1)
    defer c.Wait()

    c.Put(42, 42)
    c.Put(1, 1)
    c.Get(42)
    c.Put(0, 0)
    c.Put(2, 2)
    c.Put(3, 3)
    c.Get(42)
    // evict 1
    c.Put(4, 4)

    c.Each(func(key int, val int) {
        fmt.Println(key)
    })

    cancel()
    // Output:
    // 4
    // 42
    // 3
    // 2
    // 0
}
```

### Many Cache Partitions

```go

package main

import (
	"context"
	"fmt"

	"github.com/alistanis/cache"
)

// Example serves as a general example for using the Cache object. Since elements are spread across many partitions,
// order can not be guaranteed, and items will not be evicted in pure LRU terms; it is possible that some partitions
// may see more traffic than others and may be more eviction heavy, but generally, access patterns amortize evenly.
func main() {

	ctx, cancel := context.WithCancel(context.Background())
	concurrency := 10 // runtime.NumCPU() instead of 10 for actual use
	c := cache.New[int, int](ctx, 6, concurrency)
	defer c.Wait()

	fmt.Println(c.Meta().Len)
	fmt.Println(c.Meta().Cap)
	finished := make(chan struct{})
	go func() {
		for i := 0; i < 4*concurrency; i++ {
			c.Put(i, i)
		}
		finished <- struct{}{}
	}()

	go func() {
		for i := 8 * concurrency; i > 3*concurrency; i-- {
			c.Put(i, i)
		}
		finished <- struct{}{}
	}()

	<-finished
	<-finished

	for i := 0; i < 8*concurrency; i++ {
		v, found := c.Get(i)
		if !found {
			// get value from backing store
			// res := db.Query(...)
			// v = getValFromRes(res)
			v = 0
		} else {
			if i != v {
				panic("uh oh")
			}
		}

	}

	// we've put enough values into the cache that 10 partitions are filled with 6 elements each
	fmt.Println(c.Meta().Len)
	fmt.Println(c.Meta().Cap)
	// Output:
	// 0
	// 60
	// 60
	// 60
	cancel()
}
```