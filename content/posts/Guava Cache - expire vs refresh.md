---
title: "Guava Cache 之 Expire vs Refresh"
date: 2019-05-22T16:08:51+08:00
draft: false
---


# Cache

LocalCache是Guava Cache本地缓存的实现类，可以看到LocalCache跟ConcurentHashMap有同样的继承关系。其实，在底层实现逻辑上，Cache也是借鉴了ConcurentHashMap的实现，也是将table划分为多个的Segment，提高读写的并法度，每个Segment是一个支持并发读的哈希表实现。跟ConcurentHashMap类似，不同的Segment可以支持并发的写操作。

![](http://ww2.sinaimg.cn/large/006tNc79ly1g3a7dq0nphj30ja0c474f.jpg)

Cache跟ConcurentHashMap主要的不同是，ConcurentHashMap当某个key不在用时，需要手动进行删除。而Cache最大的一个特点是根据配置的参数大小，可以自动过期掉其中的entry。主要的过期策略有：基于最大数量的过期，基于时间的过期，以及基于引用的过期。本文主要讨论基于时间的几个过期参数。

# refresh和expire的不同点

Guava Cache提供了下面三种基于时间的过期和刷新机制：

- expireAfterAccess：当缓存项在指定的时间段内没有被读或写就会被回收
- expireAfterWrite：当缓存项在指定的时间段内没有更新就会被回收。
- refreshAfterWrite：当缓存项上一次更新操作之后的多久会被刷新。

# 源码分析

对于expire和refresh，CacheBuilder里有下面三个字段：

      long expireAfterWriteNanos = UNSET_INT;
      long expireAfterAccessNanos = UNSET_INT;
      long refreshNanos = UNSET_INT;

因为CacheBuilder是Builder模式，有以下三个方法设置参数：

refreshAfterWrite方法：

      /**
       * Specifies that active entries are eligible for automatic refresh once a fixed duration has
       * elapsed after the entry's creation, or the most recent replacement of its value. The semantics
       * of refreshes are specified in {@link LoadingCache#refresh}, and are performed by calling
       * {@link CacheLoader#reload}.
       *
       * <p>As the default implementation of {@link CacheLoader#reload} is synchronous, it is
       * recommended that users of this method override {@link CacheLoader#reload} with an asynchronous
       * implementation; otherwise refreshes will be performed during unrelated cache read and write
       * operations.
       *
       * <p>Currently automatic refreshes are performed when the first stale request for an entry
       * occurs. The request triggering refresh will make a blocking call to {@link CacheLoader#reload}
       * and immediately return the new value if the returned future is complete, and the old value
       * otherwise.
       *
       * <p><b>Note:</b> <i>all exceptions thrown during refresh will be logged and then swallowed</i>.
       *
       * @param duration the length of time after an entry is created that it should be considered
       *     stale, and thus eligible for refresh
       * @param unit the unit that {@code duration} is expressed in
       * @throws IllegalArgumentException if {@code duration} is negative
       * @throws IllegalStateException if the refresh interval was already set
       * @since 11.0
       */
      @GwtIncompatible("To be supported (synchronously).")
      public CacheBuilder<K, V> refreshAfterWrite(long duration, TimeUnit unit) {
        checkNotNull(unit);
        checkState(refreshNanos == UNSET_INT, "refresh was already set to %s ns", refreshNanos);
        checkArgument(duration > 0, "duration must be positive: %s %s", duration, unit);
        this.refreshNanos = unit.toNanos(duration);
        return this;
      }

expireAfterWrite方法：

      /**
       * Specifies that each entry should be automatically removed from the cache once a fixed duration
       * has elapsed after the entry's creation, or the most recent replacement of its value.
       *
       * <p>When {@code duration} is zero, this method hands off to
       * {@link #maximumSize(long) maximumSize}{@code (0)}, ignoring any otherwise-specificed maximum
       * size or weight. This can be useful in testing, or to disable caching temporarily without a code
       * change.
       *
       * <p>Expired entries may be counted in {@link Cache#size}, but will never be visible to read or
       * write operations. Expired entries are cleaned up as part of the routine maintenance described
       * in the class javadoc.
       *
       * @param duration the length of time after an entry is created that it should be automatically
       *     removed
       * @param unit the unit that {@code duration} is expressed in
       * @throws IllegalArgumentException if {@code duration} is negative
       * @throws IllegalStateException if the time to live or time to idle was already set
       */
      public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit) {
        checkState(expireAfterWriteNanos == UNSET_INT, "expireAfterWrite was already set to %s ns",
            expireAfterWriteNanos);
        checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
        this.expireAfterWriteNanos = unit.toNanos(duration);
        return this;
      }

expireAfterAccess方法：

      /**
       * Specifies that each entry should be automatically removed from the cache once a fixed duration
       * has elapsed after the entry's creation, the most recent replacement of its value, or its last
       * access. Access time is reset by all cache read and write operations (including
       * {@code Cache.asMap().get(Object)} and {@code Cache.asMap().put(K, V)}), but not by operations
       * on the collection-views of {@link Cache#asMap}.
       *
       * <p>When {@code duration} is zero, this method hands off to
       * {@link #maximumSize(long) maximumSize}{@code (0)}, ignoring any otherwise-specificed maximum
       * size or weight. This can be useful in testing, or to disable caching temporarily without a code
       * change.
       *
       * <p>Expired entries may be counted in {@link Cache#size}, but will never be visible to read or
       * write operations. Expired entries are cleaned up as part of the routine maintenance described
       * in the class javadoc.
       *
       * @param duration the length of time after an entry is last accessed that it should be
       *     automatically removed
       * @param unit the unit that {@code duration} is expressed in
       * @throws IllegalArgumentException if {@code duration} is negative
       * @throws IllegalStateException if the time to idle or time to live was already set
       */
      public CacheBuilder<K, V> expireAfterAccess(long duration, TimeUnit unit) {
        checkState(expireAfterAccessNanos == UNSET_INT, "expireAfterAccess was already set to %s ns",
            expireAfterAccessNanos);
        checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
        this.expireAfterAccessNanos = unit.toNanos(duration);
        return this;
      }

这三个方法实现是设置这几个参数的值。

LoadingCache的get实现：

        // com.google.common.cache.LocalCache.Segment.get
        // loading
    
        V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
          checkNotNull(key);
          checkNotNull(loader);
          try {
            if (count != 0) { // read-volatile
              // don't call getLiveEntry, which would ignore loading values
              ReferenceEntry<K, V> e = getEntry(key, hash);
              if (e != null) {
                long now = map.ticker.read();
                V value = getLiveValue(e, now); // 1. 获取尚未过期的值（如果expired了，就expire掉）
                if (value != null) {
                  recordRead(e, now);
                  statsCounter.recordHits(1);
                  return scheduleRefresh(e, key, hash, value, now, loader); // 2. 触发refresh
                }
                ValueReference<K, V> valueReference = e.getValueReference();
                if (valueReference.isLoading()) {
                  return waitForLoadingValue(e, key, valueReference);
                }
              }
            }
    
            // at this point e is either null or expired;
            return lockedGetOrLoad(key, hash, loader); // 3
          } catch (ExecutionException ee) {
            Throwable cause = ee.getCause();
            if (cause instanceof Error) {
              throw new ExecutionError((Error) cause);
            } else if (cause instanceof RuntimeException) {
              throw new UncheckedExecutionException(cause);
            }
            throw ee;
          } finally {
            postReadCleanup();
          }
        }

这里有三个关键的点：

1. 1⃣️是判断是否有存活值，即根据expireAfterAccess和expireAfterWrite进行判断是否过期，如果过期，则value为null，跳到3⃣️
2. 2⃣️指不过期的情况下，根据refreshAfterWrite判断是否需要refresh
3. 3⃣️进行加载（load而非reload），原因是没有存活值，可能因为过期，可能根本就没有过该值。

所以，在get的时候，是先判断过期，再判断是否需要refresh。

        // com.google.common.cache.LocalCache.Segment.lockedGetOrLoad
        V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader)
            throws ExecutionException {
          ReferenceEntry<K, V> e;
          ValueReference<K, V> valueReference = null;
          LoadingValueReference<K, V> loadingValueReference = null;
          boolean createNewEntry = true;
    
          lock(); // 1. 加锁
          try {
            // re-read ticker once inside the lock
            long now = map.ticker.read();
            preWriteCleanup(now);
    
            int newCount = this.count - 1;
            AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
            int index = hash & (table.length() - 1);
            ReferenceEntry<K, V> first = table.get(index);
    
            for (e = first; e != null; e = e.getNext()) {
              K entryKey = e.getKey();
              if (e.getHash() == hash && entryKey != null
                  && map.keyEquivalence.equivalent(key, entryKey)) {
                valueReference = e.getValueReference();
                if (valueReference.isLoading()) { // 2. 如果已经在loading，则不需要再loading
                  createNewEntry = false;
                } else {
                  V value = valueReference.get();
                  if (value == null) {
                    enqueueNotification(entryKey, hash, valueReference, RemovalCause.COLLECTED);
                  } else if (map.isExpired(e, now)) {
                    // This is a duplicate check, as preWriteCleanup already purged expired
                    // entries, but let's accomodate an incorrect expiration queue.
                    enqueueNotification(entryKey, hash, valueReference, RemovalCause.EXPIRED);
                  } else { // 3. 已经被其他请求load进来了，可以正常返回value
                    recordLockedRead(e, now);
                    statsCounter.recordHits(1);
                    // we were concurrent with loading; don't consider refresh
                    return value;
                  }
    
                  // immediately reuse invalid entries
                  writeQueue.remove(e);
                  accessQueue.remove(e);
                  this.count = newCount; // write-volatile
                }
                break;
              }
            }
    
            if (createNewEntry) { // 4. 准备load，先创建LoadingValueReference
              loadingValueReference = new LoadingValueReference<K, V>();
    
              if (e == null) {
                e = newEntry(key, hash, first);
                e.setValueReference(loadingValueReference);
                table.set(index, e);
              } else {
                e.setValueReference(loadingValueReference);
              }
            }
          } finally {
            unlock(); // 5. 释放锁
            postWriteCleanup();
          }
    
          if (createNewEntry) { // 6. 开始loading
            try {
              // Synchronizes on the entry to allow failing fast when a recursive load is
              // detected. This may be circumvented when an entry is copied, but will fail fast most
              // of the time.
              synchronized (e) {
                return loadSync(key, hash, loadingValueReference, loader);
              }
            } finally {
              statsCounter.recordMisses(1);
            }
          } else {
            // The entry already exists. Wait for loading.
            return waitForLoadingValue(e, key, valueReference);
          }
        }

1. 获得锁
2. 获得key对应的valueReference，判断是否正在loading，如果loading，则不再进行load操作（通过设置createNewEntry为false），后续会等待获取新值。
3. 如果不是在loading，判断是否已经有新值了（被其他请求load完了），如果是则返回新值
4. 准备loading，创建为loadingValueReference。loadingValueReference 会使其他请求在步骤3的时候会发现正在loding。
5. 释放锁。
6. 如果真的需要load，则进行load操作。

通过分析发现，只会有1个load操作，其他get会先阻塞住。

下面看看scheduleRefresh方法：

        V scheduleRefresh(ReferenceEntry<K, V> entry, K key, int hash, V oldValue, long now,
            CacheLoader<? super K, V> loader) {
          if (map.refreshes() && (now - entry.getWriteTime() > map.refreshNanos)
              && !entry.getValueReference().isLoading()) { // 1. 判断是否需要refresh
            V newValue = refresh(key, hash, loader, true);
            if (newValue != null) {
              return newValue;
            }
          }
          return oldValue;
        }

1. 判断是否需要refresh，且当前非loading状态，如果是则进行refresh操作，并返回新值。
2. 如果需要refresh，如果有其他线程正在对该值进行refreshing，则返回旧值。

        // refresh
        /**
         * Refreshes the value associated with {@code key}, unless another thread is already doing so.
         * Returns the newly refreshed value associated with {@code key} if it was refreshed inline, or
         * {@code null} if another thread is performing the refresh or if an error occurs during
         * refresh.
         */
        @Nullable
        V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
          final LoadingValueReference<K, V> loadingValueReference =
              insertLoadingValueReference(key, hash, checkTime); // 让当前值进入loading状态
          if (loadingValueReference == null) {
            return null;
          }
    
          ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
          if (result.isDone()) {
            try {
              return Uninterruptibles.getUninterruptibly(result);
            } catch (Throwable t) {
              // don't let refresh exceptions propagate; error was already logged
            }
          }
          return null;
        }

1. 插入loadingValueReference，表示该值正在loading，其他请求根据此判断是需要进行refresh还是返回旧值。insertLoadingValueReference里有加锁操作，确保只有1个refresh穿透到后端。但是，这里加锁的范围比load时候加锁的范围要小，在expire->load的过程，所有的get一旦知道expire，则需要获得锁，直到得到新值为止，阻塞的影响范围会是从expire到load到新值为止；而refresh->reload的过程，一旦get发现需要refresh，会先判断是否有loading，再去获得锁，然后释放锁之后再去reload，阻塞的范围只是insertLoadingValueReference的一个小对象的new和set操作，几乎可以忽略不计，所以这是之前说refresh比expire高效的原因之一。
2. 进行refresh操作，这里不对loadAsync进行展开，它调用了CacheLoader的reload方法，reload方法支持重载去实现异步的加载，而当前线程返回旧值，这样性能会更好，其默认是同步地调用了CacheLoader的load方法实现。

到这里，我们知道了refresh和expire的区别了吧！refresh执行reload，而expire后会重新执行load，和初始化时一样。

# 参考

- [深入Guava Cache的refresh和expire刷新机制](https://blog.csdn.net/abc86319253/article/details/53020432)
- [CachesExplained](https://github.com/google/guava/wiki/CachesExplained)