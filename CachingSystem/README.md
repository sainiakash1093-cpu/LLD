# Caching System Design

## Overview

This project implements a pluggable, in-memory caching system designed for **low latency**, **high concurrency**, and **extensibility**. It supports multiple eviction strategies and TTL-based expiration.

---

## Requirements

### Functional
- GET / SET operations using key-value pairs
- Multiple eviction strategies (LRU implemented, LFU/FIFO extensible)
- TTL support

### Non-Functional
- Low latency (O(1) operations)
- Thread-safe
- Extensible design

---

## High-Level Architecture

```text
Client -> CacheManager -> CacheInstance -> Strategy -> Storage
                         |
                 Cleanup / Snapshot
```

---

## Sequence Diagrams

### GET Flow

```text
Client
  |
  |---- Get(key) ---->
  |                  CacheManager
  |                        |
  |                        |---- get cache ---->
  |                        |
  |                  CacheInstance (LRU)
  |                        |
  |              [Check TTL / Lazy Delete]
  |                        |
  |              [Move node to front]
  |                        |
  |<------ Value -----------
```

### SET Flow

```text
Client
  |
  |---- Set(key,value) --->
  |                        CacheManager
  |                              |
  |                              |---- get cache ---->
  |                              |
  |                        CacheInstance (LRU)
  |                              |
  |                 [Check existing?]
  |                     | Yes -> Update + Move to front
  |                     | No  -> Insert
  |                              |
  |                 [If capacity full -> Evict LRU]
  |                              |
  |<----------- ACK -------------
```

---

## UML Class Diagram

```text
+----------------------+
|    CacheManager      |
+----------------------+
| - keySpaceMap        |
| - strategies         |
+----------------------+
| + RegisterKeySpace() |
| + GetCache()         |
+----------+-----------+
           |
           v
+----------------------+
|      ICache          |
+----------------------+
| + Get()              |
| + Set()              |
| + Delete()           |
+----------+-----------+
           |
   ---------------------
   |         |         |
   v         v         v
 LRU       LFU       FIFO

+----------------------+
|    CacheEntry        |
+----------------------+
| Key                  |
| Value                |
| Expiry               |
+----------------------+

+----------------------+
|   CleanupService     |
+----------------------+
| + Start()            |
+----------------------+

+----------------------+
|   SnapshotService    |
+----------------------+
| + TakeSnapshot()     |
+----------------------+
```

---

## Implementation

### CacheEntry

```csharp
public class CacheEntry<K,V>
{
    public K Key;
    public V Value;
    public DateTime? Expiry;

    public bool IsExpired()
    {
        return Expiry.HasValue && Expiry.Value < DateTime.UtcNow;
    }
}
```

---

### ICache

```csharp
public interface ICache<K, V>
{
    V Get(K k);
    void Set(K k, V v, TimeSpan? ttl = null);
    void Delete(K k);
}
```

---

### LRU Cache

```csharp
public class LRUCache<K, V> : ICache<K, V>
{
    private readonly int _capacity;
    private readonly Dictionary<K, LinkedListNode<CacheEntry<K,V>>> _map;
    private readonly LinkedList<CacheEntry<K,V>> _list;
    private readonly object _lock = new object();

    public LRUCache(int capacity = 100)
    {
        _capacity = capacity;
        _map = new Dictionary<K, LinkedListNode<CacheEntry<K,V>>>();
        _list = new LinkedList<CacheEntry<K,V>>();
    }

    private void RemoveNode(LinkedListNode<CacheEntry<K,V>> node)
    {
        _list.Remove(node);
        _map.Remove(node.Value.Key);
    }

    public V Get(K k)
    {
        lock (_lock)
        {
            if (!_map.TryGetValue(k, out var node))
                return default(V);

            if (node.Value.IsExpired())
            {
                RemoveNode(node);
                return default(V);
            }

            _list.Remove(node);
            _list.AddFirst(node);
            return node.Value.Value;
        }
    }

    public void Set(K k, V v, TimeSpan? ttl = null)
    {
        lock (_lock)
        {
            if (_map.TryGetValue(k, out var node))
            {
                node.Value.Value = v;
                node.Value.Expiry = ttl.HasValue ? DateTime.UtcNow.Add(ttl.Value) : null;
                _list.Remove(node);
                _list.AddFirst(node);
                return;
            }

            if (_map.Count >= _capacity)
            {
                var lru = _list.Last;
                if (lru != null)
                    RemoveNode(lru);
            }

            var entry = new CacheEntry<K,V>
            {
                Key = k,
                Value = v,
                Expiry = ttl.HasValue ? DateTime.UtcNow.Add(ttl.Value) : null
            };

            var newNode = new LinkedListNode<CacheEntry<K,V>>(entry);
            _list.AddFirst(newNode);
            _map[k] = newNode;
        }
    }

    public void Delete(K k)
    {
        lock (_lock)
        {
            if (_map.TryGetValue(k, out var node))
                RemoveNode(node);
        }
    }
}
```

---

### CachingStrategy

```csharp
public enum CachingStrategy
{
    LRU,
    LFU,
    FIFO
}
```

---

### CacheManager

```csharp
public class CacheManager<K, V>
{
    private readonly Dictionary<string, ICache<K,V>> keySpaceMap;
    private readonly Dictionary<CachingStrategy, Func<ICache<K, V>>> strategies;

    public CacheManager()
    {
        keySpaceMap = new Dictionary<string, ICache<K, V>>();

        strategies = new Dictionary<CachingStrategy, Func<ICache<K, V>>>
        {
            { CachingStrategy.LRU, () => new LRUCache<K, V>() }
        };
    }

    public ICache<K, V> RegisterKeySpace(string keyspace, CachingStrategy strategy)
    {
        if (keySpaceMap.ContainsKey(keyspace))
            return keySpaceMap[keyspace];

        if (!strategies.ContainsKey(strategy))
            throw new Exception("Strategy not supported");

        var cache = strategies[strategy]();
        keySpaceMap[keyspace] = cache;
        return cache;
    }

    public ICache<K, V> GetCache(string keyspace)
    {
        if (!keySpaceMap.TryGetValue(keyspace, out var cache))
            throw new Exception("Keyspace not found");

        return cache;
    }
}
```

---

### Cleanup Service

```csharp
public class CleanupService<K,V>
{
    private readonly ICache<K,V> _cache;

    public CleanupService(ICache<K,V> cache)
    {
        _cache = cache;
    }

    public void Start()
    {
        // Use Timer / BackgroundService in production
    }
}
```

---

### Snapshot Service

```csharp
public class SnapshotService<K,V>
{
    private readonly Dictionary<K,V> _snapshot = new Dictionary<K,V>();

    public void TakeSnapshot(Dictionary<K, CacheEntry<K,V>> store)
    {
        _snapshot.Clear();

        foreach (var kv in store)
        {
            if (!kv.Value.IsExpired())
                _snapshot[kv.Key] = kv.Value.Value;
        }
    }
}
```

---

## Key Design Highlights

- Strategy Pattern for eviction
- O(1) LRU using HashMap + Doubly Linked List
- TTL with lazy + background cleanup
- Thread safety via locking

---

## Future Improvements

- LFU / FIFO implementations
- Distributed cache (sharding)
- Persistence (disk / WAL)
- Metrics & observability
- Lock striping for better concurrency

---

## Summary

This system provides a strong foundation for a scalable caching layer with clean abstractions, extensibility, and production-relevant trade-offs.

