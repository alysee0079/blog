实现 LRU 算法（keep-alive 的缓存淘汰算法）
缓存淘汰算法：内存容量是有限的，当你要缓存的数据超出容量，就得有部分数据删除，这时候哪些数据删除，哪些数据保留，就是 LRU 算法和 LFU 算法，FU 强调的是访问次数，而 LRU 强调的是访问时间。

选择内存中最近最久未使用的页面予以淘汰，如果我们想要实现缓存机制 – 满足最近最少使用淘汰原则，我们就可以使用 LRU 算法缓存机制。如：vue 中 keep-alive 中就用到了此算法。

⭐LRU：即 Least Recently Used（最近最少使用算法）。把长期不使用的数据被认定为无用数据，在缓存容量满了后，会优先删除这些被认定的数据。

```javascript
var LRUCache = function (capacity) {
  this.cache = new Map()
  this.capacity = capacity
}

LRUCache.prototype.get = function (key) {
  if (this.cache.has(key)) {
    let temp = this.cache.get(key)
    this.cache.delete(key)
    this.cache.set(key, temp)
    return temp
  }
  return -1
}

LRUCache.prototype.put = function (key, value) {
  if (this.cache.has(key)) {
    this.cache.delete(key)
  } else if (this.cache.size >= this.capacity) {
    this.cache.delete(this.cache.keys().next().value)
  }
  this.cache.set(key, value)
}
```
