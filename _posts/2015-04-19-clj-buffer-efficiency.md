---
layout: post
title: Implementing a persistent fixed-sized buffer
categories: [Programming, Clojure]
---

Recently, I had need of a fixed-size buffer with a first-in-first-out eviction policy for use in a context of single-reader, multi-writer concurrency.  Since I only really needed to cache a set of objects, I did not need and associative data structure.  The right abstraction seemed to be a persistent, fixed-length stack -- push new element on the tail, and automatically evict elements from the head when the structure reaches its maximum capacity.

Exploring different implementation options turned out to be a good way to learn more about the internals of persistent data structures, and the specific characteristics of those offered in the Clojure core language, particularly the persistent vector.

I considered a ring buffer design, a naive vector-based design, an implementation on top of clojure.lang.PerisistentQueue and an implementation on top of clojure.core.rrb-vector.  If you just want to know the answer, skip to the rrb-vector section.

### Ring buffer implementation

The persistent ring buffer is implemented like a traditional array-based ring buffer, allocating a fixed-length vector, incrementing a `start` index whenever a element is evicted and incrementing the `len` offset when an element is added.  Indexes are modded with the buffer size, so that they wrap around when they reach the end.

```clojure
(deftype RingBuffer [^long start ^long len buf]
```

The main problems with this design are

* The traditional array-based implementation of a ring buffer is efficient because there is never any allocation after the initial buffer is allocated.  The slots in the original buffer are always reused.  Since the data structure underlying the persistent ring buffer is a persistent vector, not an array, update operations result in path copying, and always result in some amount of memory allocation.

* Because the tail of the persistent ring buffer is a random offset in the underlying buffer, it cannot take advantage of [tail optimizations](http://hypirion.com/musings/understanding-persistent-vector-pt-3) for append (`cons`), tail lookup (`peek`) and tail removal (`pop`) that are built into the persistent vector implementation.

```clojure
  (cons [this x]
    (if (= len (count buf))
      (RingBuffer. (rem (inc start) len) len (assoc buf start x) meta)
      (RingBuffer. start (inc len) (assoc buf (rem (+ start len) (count buf)) x) meta)))
```

The `cons` operation results in a head copy and a path copy for the underlying persistent vector, as well as the copy of the `RingBuffer` object.  Cons is unable to take advantage of tail optimization (except when the tail of the ring buffer happens to be at the tail of the underlying vector).

The `peek` and `pop` operations require a lookup to a random offset in the vector (log n), and pop requires a path copy to update the tail value to nil.

```clojure
  (peek [this]
    (nth buf (rem start (count buf))))
  (pop [this]
    (if (zero? len)
      (throw (IllegalStateException. "Can't pop empty queue"))
      (RingBuffer. (rem (inc start) (count buf)) (dec len) (assoc buf start nil) meta)))
```

Of course, given the large branching factor (32) of Clojure persistent vectors, the path copies are not deep, but each results in a copy of at least one 32-slot node.

### Naive persistent vector implementation

An alternative implementation uses a persistent vector as the underlying store, but uses its native `cons`, `peek` and `pop` operations, which can take advantage of tail optimizations.  This version needs to keep track of its maximum length, but does not need to keep track of offsets.

Here we can simply use the standard `peek` and `pop` functions with a persistent vector, and to enforce the eviction policy, we use a special `conj-fixed` function.

The trick here is that vectors do not have a native "dequeue" operation to remove an element from the head.  So when the buffer is at capacity, we need to implement the removal of the head either by copying the tail of the vector into a new vector using `rest`, or we use `subvec` to give us a view onto the original vector without the head element.

The version using `rest` will be O(N log N), the cost of building a new persistent vector from a sequence of N elements.  The version using `subvec` will be O(1), since `subvec` is O(1) and `cons` is O(1).  But, as pointed out in *Joy of Clojure*, the vector returned by `subvec` retains a reference to the original persistent vector, meaning that the head can never be garbage collected as long as we hold a reference to the modified vector.  If we are continually appending to our stack, which would be a typical application, we will leak memory.


```clojure
(defn conj-fixed-pv
  [capacity buf x & xs]
  (if xs
    (recur capacity (conj-fixed-pv capacity buf x) (first xs) (next xs))
    (let [b (if (= capacity (count buf))
              ;; subvec is O(1) but "leaks" memory, because it retains a reference to the
              ;; original vector -- the head is never GCd
              ;;(subvec buf 1)
              ;; alternate -- inefficient because requires rebuilding a vector from a sequence
              (vec (rest buf))
              ;; else, there's still room in buf for more 
              buf
              )]
      (conj b x))))
```

### clojure.lang.PersistentQueue

It turns out that Clojure does contain a persistent queue implementation, although it is not that easy to find, since there is no special reader syntax or even a core function for creating one.  The persistent fixed-length buffer using a persistent queue implements an efficient tail `cons` and an efficient head `pop`.  This is a good choice if you only need to read the head element (queue semantics), or if, as in my application, you plan to iterate over the whole buffer.

Internally, `clojure.lang.PersistentQueue` is implemented as a front list and a rear vector.  If you needed to be able to efficiently read the tail element, you could do so by extending `PersistentQueue` to expose the rear vector's `pop` method.


### clojure.core.rrb-vector

Finally, the best option for combining stack semantics with efficient head removal and tail appending is to use [clojure.core.rbb-vector](https://github.com/clojure/core.rrb-vector).  As stated in the documentation, the rbb-vector adds an O(log N) slicing (subvec) operation to `PersistentVector`.


```clojure
(defn conj-fixed-rrb
  [capacity buf x & xs]
    (if xs
      (recur capacity (conj-fixed-rrb capacity buf x) (first xs) (next xs))
      (let [b (if (= capacity (count buf))
                (rrb/subvec buf 1)
                ;; else, there's still room in buf for more 
                buf
                )]
        (conj b x))))
```

## References

See this excellent series of blog posts by [@hypirion](https://twitter.com/hyPiRion) on the implementation of [persistent data structures](http://hypirion.com/musings/understanding-persistent-vector-pt-1).

[The Joy of Clojure](http://www.manning.com/fogus2) has good, concise review of persistent data structures, and covers the limitations of persistent vectors.
