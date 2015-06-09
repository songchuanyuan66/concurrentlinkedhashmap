# Introduction #

A [Map](http://java.sun.com/javase/6/docs/api/java/util/Map.html) provides the ability to work with key->value pairs and treats each entry independently. A common use-case is that the same value could be retrieved by multiple keys. In most usages the map is unbounded and used for a short duration, so the value can be safely referenced in multiple key->value pairs.

In caches the map is bounded and multiple entries for the same value would impact the size and statistical information. An alternative approach is to store a primaryKey->value mapping in the cache and store the key->primaryKey mapping separately. When the value is added, removed, or evicted the key mapping must be updated accordingly.

## Example ##

An application may retrieve a user's profile by different lookup approaches. The primary usage may be by a unique identifier, while alternative flows may be by the user's email address or login name. In complex applications there may be many alternative lookup approaches based on multiple properties.

| **Key** | **Property** | **Type** |
|:--------|:-------------|:---------|
| id      | user.getId() | Primary  |
| email   | user.getEmail() | Secondary |
| loginName | user.getLoginName() | Secondary |

An interesting aspect of the keys is that they are all subsets of the properties on the value (its domain model). In most cases there is a relationship between the key and value rather than an ad hoc grouping. This can be leveraged to allow automatic extraction of the keys from the value.

## Usage ##

The following shows how indexing can be leveraged by a caching facade. The value contains metadata describing the cache information to allow automatic key extraction and association to a cache region. A benefit is that separate key classes are not required, but rather the domain object is treated like a template for retrieving the full value. This can be thought of as a "named lookup" style similar to Hibernate's "named query" approach.

```
public class UserProfileService {
    @Autowired
    private Cache cache;

    public User getUserById(long id) {
      User user = new User();
      user.setId(id);
      return cache.get(user, "byId");
    }

    public User getUserByEmail(String email) {
      User user = new User();
      user.setEmail(email);
      return cache.get(user, "byEmail");
    }

    public User getUserByLoginName(String loginName) {
      User user = new User();
      user.setLoginName(loginName);
      return cache.get(user, "byLoginName");
    }

    // etc
}
```

# Implementation #

The following shows an implementation of an indexable caching decorator. It uses a data map to store the association from the primaryKey to the value. A secondary map stores a key to the index. An entry is retrieved by using the lookup table to determine the primary key for retrieval from the data map.

## Concurrency ##

The IndexMap must perform multiple operations when a insertion, update, or removal occurs to keep both mappings in sync. Rather than using a single lock, _lock striping_ can be used as each entry must have a unique set of keys. This allows a high level of concurrency while maintaining correctness by blocking concurrent operations for the same entry.

A traditional _static_ model is adopted by using an array of locks. This approach is performant, but requires a good sizing as independent entry operations may contend due to sharing a lock instance. An alternative _dynamic_ model leverages the garbage collector to acheive per entry locks, but will likely not offer a significant performance improvement in practice.

```
package com.reardencommerce.kernel.concurrent.shared.locks;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * A <tt>StripedLock</tt> provides mutual exclusion on an individual lock segment and high concurrency by allowing
 * multiple segments to be operated on independently. A selection algorithm assigns a resource to a {@link Lock}
 * instance so that shared access can be controlled and should provide a good spreading strategy to evenly distribute
 * the lock contention across all segments.
 * <p>
 * The characteristics and usage pattern should follow the same guidelines when working with the {@link Lock} instances
 * directly. The lock segment can be used indirectly through operations similar to those found in {@link Lock}. In most
 * cases the following idiom should be used:
 *
 * <pre><tt>
 *     Object o = ...;
 *     StripedLock lock = ...;
 *     lock.lock(o);
 *     try {
 *         // access the resource protected by this lock
 *     } finally {
 *         lock.unlock(o);
 *     }
 * </tt></pre>
 * <p>
 * Except where noted by an implementation, passing a <tt>null</tt> value for any parameter will result in a
 * {@link NullPointerException} or {@link IllegalStateException} being thrown.
 *
 * @see    Lock
 * @author <a href="mailto:ben.manes@reardencommerce.com">Ben Manes</a>
 */
public interface StripedLock {

    /**
     * Acquires the lock.
     * <p>
     * If the lock is not available then the current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired.
     *
     * @param o The object to select the lock for.
     * @see     Lock#lock()
     **/
    void lock(Object o);

    /**
     * Acquires the lock unless the current thread is {@link Thread#interrupt interrupted}.
     * <p>
     * Acquires the lock if it is available and returns immediately.
     * <p>
     * If the lock is not available then the current thread becomes disabled for thread scheduling
     * purposes and lies dormant until one of two things happens:
     *
     * @param o                     The object to select the lock for.
     * @throws InterruptedException If the current thread is interrupted while acquiring the lock
     *                              (and interruption of lock acquisition is supported).
     * @see                         Lock#lockInterruptibly()
     * @see                         Thread#interrupt
     **/
    void lockInterruptibly(Object o) throws InterruptedException;

    /**
     * Acquires the lock only if it is free at the time of invocation.
     * <p>
     * Acquires the lock if it is available and returns immediately with the value <tt>true</tt>.
     * If the lock is not available then this method will return immediately with the value <tt>false</tt>.
     * <p>
     * A typical usage idiom for this method would be:
     * <pre>
     *      Object o = ...;
     *      StripedLock lock = ...;
     *      if (lock.tryLock(o)) {
     *          try {
     *              // manipulate protected state
     *          } finally {
     *              lock.unlock(o);
     *          }
     *      } else {
     *          // perform alternative actions
     *      }
     * </pre>
     * This usage ensures that the lock is unlocked if it was acquired, and doesn't try to unlock if the
     * lock was not acquired.
     *
     * @param o The object to select the lock for.
     * @return  <tt>true</tt> if the lock was acquired and <tt>false</tt> otherwise.
     * @see     Lock#tryLock()
     **/
    boolean tryLock(Object o);

    /**
     * Acquires the lock if it is free within the given waiting time and the current thread has not been
     * {@link Thread#interrupt interrupted}.
     * <p>
     * If the lock is available this method returns immediately with the value <tt>true</tt>.
     * If the lock is not available then the current thread becomes disabled for thread scheduling purposes
     * and lies dormant until one of three things happens:
     *
     * @param time                  The maximum time to wait for the lock.
     * @param unit                  The time unit of the <tt>time</tt> argument.
     * @return                      <tt>true</tt> if the lock was acquired and <tt>false</tt> if the waiting
     *                              time elapsed before the lock was acquired.
     * @throws InterruptedException If the current thread is interrupted while acquiring the lock
     *                              (and interruption of lock acquisition is supported).
     * @see                         Lock#tryLock()
     * @see                         Thread#interrupt
     **/
    boolean tryLock(Object o, long time, TimeUnit unit) throws InterruptedException;

    /**
     * Releases the lock.
     *
     * @param o The object to select the lock for.
     * @see     Lock#unlock()
     **/
    void unlock(Object o);

    /**
     * Returns a new {@link Condition} instance that is bound to the selected <tt>Lock</tt> instance.
     * <p>
     * Before waiting on the condition the lock must be held by the current thread. A call to
     * {@link Condition#await()} will atomically release the lock before waiting and
     * re-acquire the lock before the wait returns.
     *
     * @param o                              The object to select the lock for.
     * @return                               A new {@link Condition} instance for this <tt>Lock</tt> instance.
     * @throws UnsupportedOperationException If this <tt>Lock</tt> implementation does not support conditions.
     * @see                                  Lock#newCondition()
     **/
    Condition newCondition(Object o);
}
```
```
package com.reardencommerce.kernel.concurrent.shared.locks;

import static com.google.common.base.ReferenceType.STRONG;
import static com.google.common.base.ReferenceType.WEAK;
import static com.reardencommerce.kernel.utilities.shared.Assertions.greaterThan;
import static com.reardencommerce.kernel.utilities.shared.Assertions.notNull;
import static com.reardencommerce.kernel.utilities.shared.ObjectUtil.hash;
import static java.lang.Math.abs;

import java.util.Arrays;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

import com.google.common.collect.ReferenceMap;

/**
 * A reentrant mutual exclusive {@link StripedLock} with a selection algorithm based on the object's identity
 * ({@link #hashCode()} and {@link #equals(Object)} operations).
 * <p>
 * The individual locks supports a maximum of 2147483648 recursive locks by the same thread.
 *
 * @author <a href="mailto:ben.manes@reardencommerce.com">Ben Manes</a>
 */
public final class ReentrantStripedLock extends ForwardingStripedLock {
    private final AbstractReentrantStripedLock delegate;

    /**
     * Creates a {@link StripedLock} using a default number of stripes.
     *
     * @return A reentrant mutual exclusive {@link StripedLock}.
     */
    public static ReentrantStripedLock createFixed() {
        return createFixed(256);
    }

    /**
     * Creates a {@link StripedLock} using the specified number of stripes.
     *
     * @param size The number of locks.
     * @return     A reentrant mutual exclusive {@link StripedLock}.
     */
    public static ReentrantStripedLock createFixed(int size) {
        return new ReentrantStripedLock(new FixedStripedLock(size));
    }

    /**
     * Creates a {@link StripedLock} using a dynamic number of stripes.
     *
     * @return A reentrant mutual exclusive {@link StripedLock}.
     */
    public static ReentrantStripedLock createDynamic() {
        return new ReentrantStripedLock(new DynamicStripedLock());
    }

    /**
     * Creates a reentrant mutual exclusive {@link StripedLock} with the given selection algorithm.
     *
     * @param delegate The lock selection algorithm.
     */
    private ReentrantStripedLock(AbstractReentrantStripedLock delegate) {
        this.delegate = notNull(delegate);
    }

    /**
     * Queries if this lock is held by any thread. This method is designed for use in monitoring of the system
     * state, not for synchronization control.
     *
     * @param o The object to select the lock for.
     * @return  <tt>true</tt> if any thread holds this lock and <tt>false</tt> otherwise.
     * @see     ReentrantLock#isLocked()
     */
    public boolean isLocked(Object o) {
        return delegate().isLocked(o);
    }

    /**
     * Queries if this lock is held by the current thread.
     *
     * @param o The object to select the lock for.
     * @return  <tt>true</tt> if current thread holds this lock and <tt>false</tt> otherwise.
     * @see     ReentrantLock#isHeldByCurrentThread()
     */
    public boolean isHeldByCurrentThread(Object o) {
        return delegate().isHeldByCurrentThread(o);
    }

    /**
     * Queries the number of holds on this lock by the current thread.
     *
     * @param o The object to select the lock for.
     * @return  The number of holds on this lock by the current thread, or zero if this lock is not held
     *          by the current thread.
     * @see     ReentrantLock#getHoldCount()
     */
    public int getHoldCount(Object o) {
        return delegate().getHoldCount(o);
    }

    /**
     * Queries whether any threads are waiting to acquire this lock. Note that because cancellations may occur at any
     * time, a <tt>true</tt> return does not guarantee that any other thread will ever acquire this lock.
     * This method is designed primarily for use in monitoring of the system state.
     *
     * @param o The object to select the lock for.
     * @return  <tt>true</tt> if there may be other threads waiting to acquire the lock.
     * @see     ReentrantLock#hasQueuedThreads()
     */
    public boolean hasQueuedThreads(Object o) {
        return delegate().hasQueuedThreads(o);
    }

    /**
     * Queries whether the given thread is waiting to acquire this lock. Note that because cancellations may occur at
     * any time, a <tt>true</tt> return does not guarantee that this thread will ever acquire this lock.  This method
     * is designed primarily for use in monitoring of the system state.
     *
     * @param o      The object to select the lock for.
     * @param thread The thread,
     * @return       <tt>true</tt> if the given thread is queued waiting for this lock.
     * @see          ReentrantLock#hasQueuedThread(Thread)
     */
    public boolean hasQueuedThread(Object o, Thread thread) {
        return delegate().hasQueuedThread(o, thread);
    }

    /**
     * Returns an estimate of the number of threads waiting to acquire this lock.  The value is only an estimate
     * because the number of threads may change dynamically while this method traverses internal data structures.
     * This method is designed for use in monitoring of the system state, not for synchronization control.
     *
     * @param o The object to select the lock for.
     * @return  The estimated number of threads waiting for this lock
     * @see     ReentrantLock#getQueueLength()
     */
    public int getQueueLength(Object o) {
        return delegate().getQueueLength(o);
    }

    /**
     * Queries whether any threads are waiting on the given condition associated with this lock. Note that because
     * timeouts and interrupts may occur at any time, a <tt>true</tt> return does not guarantee that a future
     * <tt>signal</tt> will awaken any threads.  This method is designed primarily for use in monitoring of
     * the system state.
     *
     * @param o         The object to select the lock for.
     * @param condition The condition
     * @return          <tt>true</tt> if there are any waiting threads.
     * @see             ReentrantLock#hasWaiters(Condition)
     */
    public boolean hasWaiters(Object o, Condition condition) {
        return delegate().hasWaiters(o, condition);
    }

    /**
     * Returns an estimate of the number of threads waiting on the given condition associated with this lock.
     * Note that because timeouts and interrupts may occur at any time, the estimate serves only as an upper
     * bound on the actual number of waiters. This method is designed for use in monitoring of the system
     * state, not for synchronization control.
     *
     * @param o         The object to select the lock for.
     * @param condition The condition.
     * @return          The estimated number of waiting threads.
     * @see             ReentrantLock#getWaitQueueLength(Condition)
     */
    public int getWaitQueueLength(Object o, Condition condition) {
        return delegate().getWaitQueueLength(o, condition);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected AbstractReentrantStripedLock delegate() {
        return delegate;
    }

    /**
     * The base class for a reentrant striped lock selection algorithm.
     */
    private static abstract class AbstractReentrantStripedLock implements StripedLock {

        /**
         * Retrieves the lock instance for the object.
         *
         * @param o The object to select the lock for.
         * @return  The shared lock instance.
         */
        protected abstract ReentrantLock getLock(Object o);

        /**
         * @see ReentrantStripedLock#isLocked(Object)
         */
        public boolean isLocked(Object o) {
            return getLock(o).isLocked();
        }

        /**
         * @see ReentrantStripedLock#isHeldByCurrentThread(Object)
         */
        public boolean isHeldByCurrentThread(Object o) {
            return getLock(o).isHeldByCurrentThread();
        }

        /**
         * @see ReentrantStripedLock#getHoldCount(Object)
         */
        public int getHoldCount(Object o) {
            return getLock(o).getHoldCount();
        }

        /**
         * @see ReentrantStripedLock#hasQueuedThreads(Object)
         */
        public boolean hasQueuedThreads(Object o) {
            return getLock(o).hasQueuedThreads();
        }

        /**
         * @see ReentrantStripedLock#hasQueuedThread(Object, Thread)
         */
       public boolean hasQueuedThread(Object o, Thread thread) {
           return getLock(o).hasQueuedThread(thread);
       }

       /**
        * @see ReentrantStripedLock#getQueueLength(Object)
        */
       public int getQueueLength(Object o) {
           return getLock(o).getQueueLength();
       }

       /**
        * @see ReentrantStripedLock#getWaitQueueLength(Object, Condition)
        */
       public int getWaitQueueLength(Object o, Condition condition) {
           return getLock(o).getWaitQueueLength(condition);
       }

       /**
        * @see ReentrantStripedLock#hasWaiters(Object, Condition)
        */
       public boolean hasWaiters(Object o, Condition condition) {
           return getLock(o).hasWaiters(condition);
       }
    }

    /**
     * Selects the lock based on the object's {@link #hashCode()}. A universal hashing algorithm is applied to
     * improve the spreading across the lock instances.
     */
    private static final class FixedStripedLock extends AbstractReentrantStripedLock {
        private final ReentrantLock[] locks;

        private FixedStripedLock(int size) {
            greaterThan(0, size);
            locks = new ReentrantLock[size];
            for (int i=0; i<size; i++) {
                locks[i] = new ReentrantLock();
            }
        }
        @Override
        protected ReentrantLock getLock(Object o) {
            return locks[abs(hash(notNull(o)) % locks.length)];
        }
        public void lock(Object o) {
            getLock(o).lock();
        }
        public void lockInterruptibly(Object o) throws InterruptedException {
            getLock(o).lockInterruptibly();
        }
        public boolean tryLock(Object o) {
            return getLock(o).tryLock();
        }
        public boolean tryLock(Object o, long time, TimeUnit unit) throws InterruptedException {
            return getLock(o).tryLock(time, unit);
        }
        public void unlock(Object o) {
            getLock(o).unlock();
        }
        public Condition newCondition(Object o) {
            return getLock(o).newCondition();
        }
        @Override
        public String toString() {
            return Arrays.toString(locks);
        }
    }

    /**
     * Selects the lock by associating the object to a unique lock instance. The instances are discarded by a weak
     * reference eviction policy so that mutual exclusion is guaranteed for the duration of the lock's shared usage.
     * If multiple requests for the lock occur, the operations will be performed serially as only one thread can
     * lock the mutex. When no threads reference the lock instance then it is eligible for garbage collection.
     */
    private static final class DynamicStripedLock extends AbstractReentrantStripedLock {
        private final ConcurrentMap<ReentrantLock, Integer> references;
        private final ConcurrentMap<Object, ReentrantLock> locks;

        private DynamicStripedLock() {
            references = new ConcurrentHashMap<ReentrantLock, Integer>();
            locks = new ReferenceMap<Object, ReentrantLock>(STRONG, WEAK);
        }

        /**
         * {@inheritDoc}
         */
        @Override
        protected ReentrantLock getLock(Object o) {
            ReentrantLock lock = new ReentrantLock();
            ReentrantLock current = locks.putIfAbsent(o, lock);
            return (current == null) ? lock : current;
        }

        /**
         * Increments a strong reference to the lock while it is in use.
         *
         * @param lock The shared lock instance that is being used by this thread.
         */
        private void increment(ReentrantLock lock) {
            for (;;) {
                Integer current = references.putIfAbsent(lock, 1);
                if ((current == null) || references.replace(lock, current, ++current)) {
                    return; // inserted first use or incremented usage count
                }
            }
        }

        /**
         * Decrements a strong reference to the lock. When it is no longer shared and in use then
         * it becomes eligible for garbage collection.
         *
         * @param lock The shared lock instance that was previously used by this thread.
         */
        private void decrement(ReentrantLock lock) {
            for (;;) {
                Integer current = references.get(lock);
                if (current.intValue() == 1) {
                    if (references.remove(lock, current)) {
                        return; // removed only usage of lock
                    }
                    continue; // retry
                } else if (references.replace(lock, current, --current)) {
                    return; // decremented shared usage of lock
                }
            }
        }
        public void lock(Object o) {
            ReentrantLock lock = getLock(o);
            increment(lock);
            lock.lock();
        }
        public void lockInterruptibly(Object o) throws InterruptedException {
            ReentrantLock lock = getLock(o);
            increment(lock);
            lock.lockInterruptibly();
        }
        public boolean tryLock(Object o) {
            ReentrantLock lock = getLock(o);
            if (lock.tryLock()) {
                increment(lock);
                return true;
            }
            return false;
        }
        public boolean tryLock(Object o, long time, TimeUnit unit) throws InterruptedException {
            ReentrantLock lock = getLock(o);
            if (lock.tryLock(time, unit)) {
                increment(lock);
                return true;
            }
            return false;
        }
        public void unlock(Object o) {
            ReentrantLock lock = getLock(o);
            lock.unlock();
            decrement(lock);
        }
        public Condition newCondition(final Object o) {
            return new DynamicCondition(getLock(o));
        }
        @Override
        public boolean hasWaiters(Object o, Condition condition) {
            if (!(condition instanceof DynamicCondition)) {
                throw new IllegalArgumentException("not owner");
            }
            return getLock(o).hasWaiters(((DynamicCondition) condition).condition);
        }
        @Override
        public int getWaitQueueLength(Object o, Condition condition) {
            if (!(condition instanceof DynamicCondition)) {
                throw new IllegalArgumentException("not owner");
            }
            return getLock(o).getWaitQueueLength(((DynamicCondition) condition).condition);
        }
        @Override
        public String toString() {
            return locks.toString();
        }

        /**
         * A condition that while strongly reachable its associated lock is strongly reachable too.
         */
        private static final class DynamicCondition extends ForwardingCondition {
            @SuppressWarnings("unused")
            private final ReentrantLock lock;
            private final Condition condition;

            public DynamicCondition(ReentrantLock lock) {
                this.lock = lock;
                this.condition = lock.newCondition();
            }
            @Override
            protected Condition delegate() {
                return condition;
            }
        }
    }
}
```

## IndexMap ##

When used with a cache as the data store, it is assumed that on an eviction a listener performs a call-back to notify the decorator. With a bit of care, the SelfPopulatingMap and ExpirableMap can decorate the IndexMap.

```
package com.reardencommerce.kernel.collections.shared;

import static com.google.common.collect.Iterators.emptyIterator;
import static com.google.common.collect.Maps.immutableEntry;
import static com.reardencommerce.kernel.collections.shared.Maps2.asMap;
import static com.reardencommerce.kernel.collections.shared.Sets2.union;
import static com.reardencommerce.kernel.concurrent.shared.locks.ReentrantStripedLock.createFixed;
import static com.reardencommerce.kernel.utilities.shared.Assertions.equalTo;
import static com.reardencommerce.kernel.utilities.shared.Assertions.isFalse;
import static com.reardencommerce.kernel.utilities.shared.Assertions.isTrue;
import static com.reardencommerce.kernel.utilities.shared.Assertions.notNull;
import static java.lang.String.format;
import static java.util.Collections.unmodifiableSet;

import java.util.AbstractMap;
import java.util.AbstractSet;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

import com.google.common.base.Function;
import com.google.common.collect.ForwardingCollection;
import com.google.common.collect.ForwardingSet;
import com.google.common.collect.ImmutableSet;
import com.reardencommerce.kernel.concurrent.shared.locks.StripedLock;

/**
 * A {@link ConcurrentMap} where multiple keys index to the same value and removal of the value causes the removal
 * of all of its associated keys. When a value is added it can be retrieved by using any of the applicable keys.
 * It is assumed that keys-value mappings do not overlap. All operations that update associations in the map and
 * accept both a key and a value parameter will assert that the given key is a valid association.
 * <p>
 * This implementation provides a different definition of {@link #size()}, which reflects the number of values rather
 * than the number of key-value pairs. The collections retrieved from {@link #keySet()}, {@link #values()}, and
 * {@link #entrySet()} are not naturally aligned as they reflect different aspects of the data structure.
 * <p>
 * The entries in this map must conform to the following restrictions.
 * <ul>
 *   <li> The key-value mappings may not overlap.
 *   <li> The primary key cannot change over the lifetime of the value.
 *   <li> The secondary keys may change over the lifetime of the value.
 * </ul>
 *
 * @author <a href="mailto:ben.manes@reardencommerce.com">Ben Manes</a>
 */
public class IndexMap<K, V> extends AbstractMap<K, V> implements ConcurrentMap<K, V> {
    private final ConcurrentMap<K, Index<K>> indexes;
    private final Function<V, Index<K>> extractor;
    private final ConcurrentMap<K, V> store;
    protected final StripedLock lock;

    /**
     * An {@link IndexMap} backed by a {@link ConcurrentHashMap}.
     *
     * @param extractor Extracts the keys that are associated with the given value.
     */
    public static <K, V> IndexMap<K, V> create(Function<V, Index<K>> extractor) {
        return create(new ConcurrentHashMap<K, V>(), extractor);
    }

    /**
     * An {@link IndexMap} backed by the specified map.
     *
     * @param store     The backing data map to decorate.
     * @param extractor Extracts the keys that are associated with the given value.
     */
    public static <K, V> IndexMap<K, V> create(ConcurrentMap<K, V> store, Function<V, Index<K>> extractor) {
        return create(store, new ConcurrentHashMap<K, Index<K>>(), extractor);
    }

    /**
     * An {@link IndexMap} backed by the specified maps.
     *
     * @param store     The backing data map to decorate.
     * @param indexes   The backing key index map to decorate.
     * @param extractor Extracts the keys that are associated with the given value.
     */
    public static <K, V> IndexMap<K, V> create(ConcurrentMap<K, V> store, ConcurrentMap<K, Index<K>> indexes,
                                               Function<V, Index<K>> extractor) {
        return new IndexMap<K, V>(store, indexes, extractor);
    }

    /**
     * An implementation backed by the specified map.
     *
     * @param store     The backing data map to decorate.
     * @param indexes   The backing key index map to decorate.
     * @param extractor Extracts the keys that are associated with the given value.
     */
    IndexMap(ConcurrentMap<K, V> store, ConcurrentMap<K, Index<K>> indexes, Function<V, Index<K>> extractor) {
        this.extractor = notNull(extractor);
        this.indexes = notNull(indexes);
        this.store = notNull(store);
        this.lock = createFixed();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean containsKey(Object key) {
        return indexes.containsKey(key);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean containsValue(Object value) {
        return store.containsValue(value);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean isEmpty() {
        return store.isEmpty();
    }

    /**
     * Retrieves the number of values in the map. The number of keys can be retrieved through {@link #keySet()}.
     *
     * @return The number of values in the map.
     */
    @Override
    public int size() {
        return store.size();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void clear() {
        for (K key : store.keySet()) {
            remove(key);
        }
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public V get(Object key) {
        Index<K> index = indexes.get(key);
        return (index == null) ? null : store.get(index.getPrimary());
    }

    /**
     * Places the values into the map and associates its keys.
     *
     * @param values The values to put into the map.
     */
    public void putAllValues(Iterable<? extends V> values) {
        for (V value : values) {
            putValue(value);
        }
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public V put(K key, V value) {
        Index<K> index = checkIndex(key, value);
        return put(index, value);
    }

    /**
     * Places the value into the map and associates its keys.
     *
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    public V putValue(V value) {
        notNull(value);
        Index<K> index = extractor.apply(value);
        return put(index, value);
    }

    /**
     * Places the value into the map and associates its keys.
     *
     * @param index The key associations.
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    private V put(Index<K> index, V value) {
        K primary = index.getPrimary();
        lock.lock(primary);
        try {
            V old = store.put(primary, value);
            if (old == null) {
                addIndex(index);
            } else {
                updateIndex(indexes.get(primary), index);
            }
            return old;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * {@inheritDoc}
     */
    public V putIfAbsent(K key, V value) {
        Index<K> index = checkIndex(key, value);
        return putIfAbsent(index, value);
    }

    /**
     * Places the value into the map if absent and associates its keys.
     *
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    public V putIfAbsentValue(V value) {
        notNull(value);
        Index<K> index = extractor.apply(value);
        return putIfAbsent(index, value);
    }

    /**
     * Places the value into the map if absent and associates its keys.
     *
     * @param index The key associations.
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    private V putIfAbsent(Index<K> index, V value) {
        K primary = index.getPrimary();
        lock.lock(primary);
        try {
            V old = store.putIfAbsent(primary, value);
            if (old == null) {
                addIndex(index);
            }
            return old;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * {@inheritDoc}
     * <p>
     * The key must be associated with the replacement value. If it is only specified on the value being replaced then
     * the association validation will fail.
     */
    public V replace(K key, V value) {
        Index<K> index = checkIndex(key, value);
        return replace(index, value);
    }

    /**
     * Replaces the entry based on the primary key only if it is currently mapped to some value.
     *
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    public V replaceValue(V value) {
        notNull(value);
        Index<K> index = extractor.apply(value);
        return replace(index, value);
    }

    /**
     * Replaces the entry based on the primary key only if it is currently mapped to some value.
     *
     * @param index The key associations.
     * @param value The value to put into the map.
     * @return      The previous value, or <tt>null</tt> if there was no mapping.
     */
    private V replace(Index<K> index, V value) {
        K primary = index.getPrimary();
        lock.lock(primary);
        try {
            V old = store.replace(primary, value);
            if (old != null) {
                updateIndex(indexes.get(primary), index);
            }
            return old;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * {@inheritDoc}
     */
    public boolean replace(K key, V oldValue, V newValue) {
        notNull(key);
        notNull(oldValue);
        notNull(newValue);
        Index<K> oldIndex = extractor.apply(oldValue);
        Index<K> newIndex = extractor.apply(newValue);
        Set<K> keys = union(oldIndex.getAll(), newIndex.getAll());
        isTrue(keys.contains(key), null, "Invalid association: %s not in %s or %s", key, oldIndex, newIndex);
        return replace(oldIndex, oldValue, newIndex, newValue);
    }

    /**
     * Replaces the entry based on the primary key only if it is currently mapped to the given value.
     *
     * @param oldValue The expected value in the map.
     * @param newValue The value to put into the map.
     * @return         Whether the value was replaced.
     */
    public boolean replaceValue(V oldValue, V newValue) {
        notNull(oldValue);
        notNull(newValue);
        Index<K> oldIndex = extractor.apply(oldValue);
        Index<K> newIndex = extractor.apply(newValue);
        return replace(oldIndex, oldValue, newIndex, newValue);
    }

    /**
     * Replaces the entry based on the primary key only if it is currently mapped to the given value.
     *
     * @param oldIndex The old key associations.
     * @param oldValue The expected value in the map.
     * @param newValue The value to put into the map.
     * @return         Whether the value was replaced.
     */
    private boolean replace(Index<K> oldIndex, V oldValue, Index<K> newIndex, V newValue) {
        equalTo(oldIndex.getPrimary(), newIndex.getPrimary(),
                "The primary keys do not match: %s != %s", oldIndex.getPrimary(), newIndex.getPrimary());
        K primary = oldIndex.getPrimary();
        lock.lock(primary);
        try {
            if (store.replace(primary, oldValue, newValue)) {
                updateIndex(oldIndex, newIndex);
                return true;
            }
            return false;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public V remove(Object key) {
        notNull(key);
        Index<K> index = indexes.get(key);
        if (index == null) {
            return null;
        }
        K primary = index.getPrimary();
        lock.lock(primary);
        try {
            // if the store is bounded then the value may be evicted and a call-back performed to cleanup
            // the index keys. Therefore remove them if found, regardless of whether the value still exists
            V value = store.remove(primary);
            index = indexes.get(primary);
            if (index != null) {
                removeIndex(index);
            }
            return value;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * {@inheritDoc}
     */
    public boolean remove(Object key, Object value) {
        notNull(key);
        notNull(value);
        Index<K> index = indexes.get(key);
        if (index == null) {
            return false;
        }
        K primary = index.getPrimary();
        lock.lock(primary);
        try {
            if (store.remove(primary, value)) {
                removeIndex(indexes.get(primary));
                return true;
            }
            return false;
        } finally {
            lock.unlock(primary);
        }
    }

    /**
     * Removes the value and its associates keys from the map.
     *
     * @param value The value to remove from the map.
     * @return      Whether the value was removed.
     */
    public boolean removeValue(V value) {
        notNull(value);
        Index<K> index = extractor.apply(value);
        return remove(index.getPrimary(), value);
    }

    /**
     * Extracts the index from the value and asserts that the given key is a valid association.
     *
     * @param key   The key.
     * @param value The value.
     * @return      The index.
     */
    private Index<K> checkIndex(K key, V value) {
        notNull(key);
        notNull(value);
        Index<K> index = extractor.apply(value);
        return isTrue(index.getAll().contains(key), index, "Invalid association: %s not in %s", key, index);
    }

    /**
     * Adds the indexed keys. It is assumed that the lock is held.
     *
     * @param index The key associations.
     */
    protected void addIndex(Index<K> index) {
        indexes.putAll(asMap(index.getAll(), index));
    }

    /**
     * Removes the indexed keys. It is assumed that the lock is held.
     *
     * @param index The key associations.
     */
    protected void removeIndex(Index<K> index) {
        for (K key : index.getAll()) {
            indexes.remove(key);
        }
    }

    /**
     * Updates the index associations for a replaced value. It is assumed that the lock is held.
     * <p>
     * It is expected that the primary key does not change, but that secondary key associations may have become stale.
     * If so, the stale indexes are removed and the new indexes added to the mapping.
     *
     * @param oldIndex The old key associations.
     * @param newIndex The new key associations.
     */
    protected void updateIndex(Index<K> oldIndex, Index<K> newIndex) {
        // Adds the new keys, or updates old ones, to map to the new index
        addIndex(newIndex);

        // Removes the stale keys that no longer map to the old index
        Set<K> removed = new HashSet<K>(oldIndex.getSecondaries());
        removed.removeAll(newIndex.getSecondaries());
        for (K key : removed) {
            indexes.remove(key);
        }
    }

    /**
     * Retrieves the backing map of the keys to their primary.
     *
     * @return The key-primary mapping.
     */
    protected ConcurrentMap<K, Index<K>> keyDelegate() {
        return indexes;
    }

    /**
     * Retrieves the backing map of the primary key to the value.
     *
     * @return The primary-value mapping.
     */
    protected ConcurrentMap<K, V> valueDelegate() {
        return store;
    }

    /**
     * Retrieves the function that extracts the keys from a value.
     *
     * @return The key generating function.
     */
    public Function<V, Index<K>> extractor() {
        return extractor;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Set<K> keySet() {
        return new KeySet();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Collection<V> values() {
        return new Values();
    }


    /**
     * {@inheritDoc}
     */
    @Override
    public Set<Entry<K, V>> entrySet() {
        return new EntrySet();
    }

    /**
     * The index associated with a value.
     *
     * @author <a href="mailto:ben.manes@reardencommerce.com">Ben Manes</a>
     */
    public static final class Index<K> {
        private final K primary;
        private final Set<K> all;
        private final Set<K> secondaries;

        /**
         * The index keys associated with a single value.
         *
         * @param primary     The primary key to the value.
         * @param secondaries The secondary keys to the value.
         */
        public Index(K primary, K... secondaries) {
            this(primary, ImmutableSet.of(secondaries));
            equalTo(secondaries.length, getSecondaries().size(), "Duplicate keys");
        }

        /**
         * The index keys associated with a single value.
         *
         * @param primary     The primary key to the value.
         * @param secondaries The secondary keys to the value.
         */
        public Index(K primary, Set<K> secondaries) {
            this.primary = notNull(primary);
            this.secondaries = unmodifiableSet(secondaries);
            this.all = union(getPrimary(), getSecondaries());
            isFalse(secondaries.contains(primary), "Secondaries contain primary key");
        }

        /**
         * Retrieves the primary key.
         */
        public K getPrimary() {
            return primary;
        }

        /**
         * Retrieves the secondary keys to index with.
         */
        public Set<K> getSecondaries() {
            return secondaries;
        }

        /**
         * Retrieves all of the keys.
         */
        public Set<K> getAll() {
            return all;
        }

        /**
         * {@inheritDoc}
         */
        @Override
        public String toString() {
            return format("primary=%s, secondaries=%s", getPrimary(), getSecondaries());
        }
    }

    /**
     * An adapter to safely externalize the keys.
     */
    private final class KeySet extends ForwardingSet<K> {
        private final Set<K> keys = indexes.keySet();

        @Override
        public void clear() {
            IndexMap.this.clear();
        }
        @Override
        public Iterator<K> iterator() {
            return new KeyIterator(keys.iterator());
        }
        @Override
        public boolean remove(Object key) {
            return (IndexMap.this.remove(key) != null);
        }
        @Override
        protected Set<K> delegate() {
            return keys;
        }
    }

    /**
     * An adapter to safely externalize the keys.
     */
    private final class KeyIterator implements Iterator<K> {
        private final Iterator<K> keys;
        private K current;

        public KeyIterator(Iterator<K> keys) {
            this.keys = keys;
        }
        public boolean hasNext() {
            return keys.hasNext();
        }
        public K next() {
            current = keys.next();
            return current;
        }
        public void remove() {
            IndexMap.this.remove(current);
        }
    }

    /**
     * An adapter to safely externalize the values.
     */
    private final class Values extends ForwardingCollection<V> {
        private final Collection<V> values = store.values();

        @Override
        public void clear() {
            IndexMap.this.clear();
        }
        @Override
        public Iterator<V> iterator() {
            return new ValueIterator(values.iterator());
        }
        @Override
        @SuppressWarnings("unchecked")
        public boolean remove(Object value) {
            return IndexMap.this.removeValue((V) value);
        }
        @Override
        protected Collection<V> delegate() {
            return values;
        }
    }

    /**
     * An adapter to safely externalize the values.
     */
    private final class ValueIterator implements Iterator<V> {
        private final Iterator<V> values;
        private V current;

        public ValueIterator(Iterator<V> values) {
            this.values = values;
        }
        public boolean hasNext() {
            return values.hasNext();
        }
        public V next() {
            current = values.next();
            return current;
        }
        public void remove() {
            IndexMap.this.removeValue(current);
        }
    }

    /**
     * An adapter that represents the association of multiple keys to a single value.
     */
    private final class EntrySet extends AbstractSet<Entry<K, V>> {
        @Override
        public void clear() {
            IndexMap.this.clear();
        }
        @Override
        public int size() {
            return IndexMap.this.indexes.size();
        }
        @Override
        public Iterator<Entry<K, V>> iterator() {
            return new EntryIterator(store.values().iterator());
        }
        @Override
        public boolean contains(Object obj) {
            if (!(obj instanceof Entry)) {
                return false;
            }
            Entry<?, ?> entry = (Entry<?, ?>) obj;
            V value = get(entry.getKey());
            return (value != null) && (value.equals(entry.getValue()));
        }
        @Override
        public boolean add(Entry<K, V> entry) {
            return (putIfAbsent(entry.getKey(), entry.getValue()) == null);
        }
        @Override
        public boolean remove(Object obj) {
            if (!(obj instanceof Entry)) {
                return false;
            }
            Entry<?, ?> entry = (Entry<?, ?>) obj;
            return IndexMap.this.remove(entry.getKey(), entry.getValue());
        }
        @Override
        public Object[] toArray() {
            // avoid 3-expression 'for' loop when using concurrent collections
            Collection<Entry<K, V>> entries = new ArrayList<Entry<K, V>>(size());
            for (Entry<K, V> entry : this) {
                entries.add(entry);
            }
            return entries.toArray();
        }
        @Override
        public <T> T[] toArray(T[] array) {
            // avoid 3-expression 'for' loop when using concurrent collections
            Collection<Entry<K, V>> entries = new ArrayList<Entry<K, V>>(size());
            for (Entry<K, V> entry : this) {
                entries.add(entry);
            }
            return entries.toArray(array);
        }
    }

    /**
     * An adapter that represents the association of multiple keys to a single value.
     */
    private final class EntryIterator implements Iterator<Entry<K, V>> {
        private final Iterator<V> values;
        private Iterator<K> keys;
        private V value;

        public EntryIterator(Iterator<V> values) {
            this.keys = emptyIterator();
            this.values = values;
        }
        public boolean hasNext() {
            return keys.hasNext() || values.hasNext();
        }
        public Entry<K, V> next() {
            if (!keys.hasNext()) {
                value = values.next();
                Index<K> index = extractor.apply(value);
                keys = index.getAll().iterator();
            }
            return immutableEntry(keys.next(), value);
        }
        public void remove() {
            IndexMap.this.removeValue(value);
        }
    }
}
```
```
package com.reardencommerce.kernel.collections.shared;

import static com.google.common.collect.Sets.difference;
import static com.google.common.collect.Sets.intersection;
import static com.reardencommerce.kernel.collections.shared.Maps2.entry;
import static com.reardencommerce.kernel.concurrent.shared.ConcurrentTestHarness.timeTasks;
import static com.reardencommerce.kernel.utilities.shared.Assertions.greaterThan;
import static com.reardencommerce.kernel.utilities.shared.Assertions.notBlank;
import static java.lang.String.format;
import static java.util.Arrays.asList;
import static java.util.Collections.emptyList;
import static java.util.Collections.emptySet;
import static java.util.Collections.shuffle;
import static org.testng.Assert.assertEquals;
import static org.testng.Assert.assertFalse;
import static org.testng.Assert.assertNotNull;
import static org.testng.Assert.assertNotSame;
import static org.testng.Assert.assertNull;
import static org.testng.Assert.assertTrue;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Queue;
import java.util.Random;
import java.util.Set;
import java.util.Map.Entry;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;

import org.testng.annotations.BeforeMethod;
import org.testng.annotations.Test;

import com.google.common.base.Function;
import com.google.common.collect.Iterables;
import com.reardencommerce.kernel.collections.shared.IndexMap.Index;
import com.reardencommerce.kernel.concurrent.shared.locks.ReentrantStripedLock;

/**
 * A unit test for the {@link IndexMap}.
 *
 * @author <a href="mailto:ben.manes@reardencommerce.com">Ben Manes</a>
 */
public final class IndexMapTest {
    private final User alice = new User(1, "alice", "alice@gmail.com", "alice123");  // original profile
    private final User alice2 = new User(1, "alice", "alice@gmail.com", "alice456"); // updated profile
    private final User bob = new User(2, "bob", "bob@yahoo.com", "bob");
    private final User charlie = new User(3, "charlie", "charlie@hotmail.com", "charlie1960");
    private final IndexMap<Object, User> map = IndexMap.create(new UserKeyExtractor());
    private final Random random = new Random();

    @BeforeMethod
    public void reset() {
        map.clear();
        assertEmpty();
    }

    @Test
    public void index() {
        Index<Object> index = map.extractor().apply(alice);
        assertTrue(index.getAll().contains(index.getPrimary()));
        assertTrue(index.getAll().containsAll(index.getSecondaries()));
        assertFalse(index.getSecondaries().contains(index.getPrimary()));
        assertEquals(index.toString(), format("primary=%s, secondaries=%s", index.getPrimary(), index.getSecondaries()));
    }

    @Test
    public void internals() {
        assertTrue(map.keyDelegate().isEmpty());
        assertTrue(map.valueDelegate().isEmpty());
        map.putAllValues(asList(alice, bob, charlie));
        assertEquals(map.keyDelegate().size(), 9);
        assertEquals(map.valueDelegate().size(), 3);
    }

    @Test
    public void getPutRemove() {
        // insert
        assertNull(map.putValue(alice));
        assertContainsOnly(alice);
        assertNotFound(bob, charlie);

        // update
        assertEquals(map.put("alice456", alice2), alice);
        assertUpdate(alice, alice2);
        assertEquals(map.put("alice123", alice), alice2);
        assertUpdate(alice2, alice); // restored

        // add
        assertEquals(map.putIfAbsentValue(alice), alice);
        assertNull(map.putIfAbsent("bob@yahoo.com", bob));
        assertContainsOnly(alice, bob);
        assertNotFound(charlie);

        // remove
        assertNull(map.remove("charlie1960"));
        assertFalse(map.removeValue(charlie));
        assertFalse(map.remove("charlie@hotmail.com", charlie));
        assertEquals(map.remove("bob@yahoo.com"), bob);
        assertTrue(map.remove("alice123", alice));

        assertEmpty();
    }

    @Test
    public void replace() {
        assertNull(map.replaceValue(alice));
        assertNull(map.replace("alice123", alice));
        assertFalse(map.replaceValue(alice, alice2));
        assertFalse(map.replace("alice123", alice, alice2));
        map.putAllValues(asList(alice, bob, charlie));

        // v1 -> v2
        assertEquals(map.replaceValue(alice2), alice);
        assertUpdate(alice, alice2);

        // v2 -> v1
        assertEquals(map.replace(1, alice), alice2);
        assertUpdate(alice2, alice);

        // v1 -> v2
        assertTrue(map.replaceValue(alice, alice2));
        assertUpdate(alice, alice2);

        // v2 -> v1
        assertTrue(map.replace(1, alice2, alice));
        assertUpdate(alice2, alice);

        assertContainsOnly(alice, bob, charlie);
    }

    @Test
    public void rawObjectMethods() {
        map.putAllValues(asList(alice, bob, charlie));
        Object o = new Object();
        assertNull(map.get(o));
        assertFalse(map.containsKey(o));
        assertFalse(map.containsValue(o));
        assertFalse(map.remove(o, o));
    }

    @Test
    public void clear() {
        map.putAllValues(asList(alice, bob, charlie));
        assertContainsOnly(alice, bob, charlie);
        map.clear();
        assertEmpty();
    }

    @Test
    public void entrySet() {
        // contains, add, remove
        Set<Entry<Object, User>> set = map.entrySet();
        assertFalse(set.contains(entry((Object) 1, alice)));
        assertFalse(set.containsAll((asList(entry(1, alice)))));
        assertTrue(set.add(entry((Object) 1, alice)));
        assertFalse(set.add(entry((Object) 1, alice)));
        assertEquals(set.size(), 3); // 1 value -> 3 keys
        assertTrue(set.contains(entry((Object) 1, alice)));
        assertTrue(set.containsAll((asList(entry(1, alice)))));
        assertFalse(set.contains(alice));
        assertFalse(set.remove(new Object()));
        assertTrue(set.remove(entry((Object) 1, alice)));
        assertFalse(set.remove(entry((Object) 1, alice)));
        assertTrue(set.isEmpty());
        assertEmpty();

        // clear
        map.putAllValues(asList(alice, bob, charlie));
        assertFalse(set.isEmpty());
        set.clear();
        assertTrue(set.isEmpty());
        assertEmpty();

        // toArray
        assertEquals(set.toArray().length, 0);
        assertEquals(set.toArray(new Entry[set.size()]).length, 0);
        map.putAllValues(asList(alice, bob, charlie));
        Object[] array1 = set.toArray();
        Entry<?, ?>[] array2 = set.toArray(new Entry[set.size()]);
        assertEquals(array1, array2);
        assertEquals(set.toArray().length, 9); // 3 values -> 9 keys
        for (User user : asList(alice, bob, charlie)) {
            for (Object key : map.extractor().apply(user).getAll()) {
                assertTrue(asList(array1).contains(entry(key, user)));
            }
        }

        // iterator
        map.putAllValues(asList(alice, bob, charlie));
        assertContainsOnly(alice, bob, charlie);
        for (Iterator<Entry<Object, User>> i=map.entrySet().iterator(); i.hasNext();) {
            assertNotNull(i.next());
            i.remove();
        }
        assertEmpty();
    }

    /*
     * Negative tests (get)
     */
    @Test(expectedExceptions=RuntimeException.class)
    public void getWithNullKey() {
        map.get(null);
    }

    /*
     * Negative tests (containsKey)
     */
    @Test(expectedExceptions=RuntimeException.class)
    public void containsKeyWithNullKey() {
        map.containsKey(null);
    }

    /*
     * Negative tests (containsKey)
     */
    @Test(expectedExceptions=RuntimeException.class)
    public void containsValueWithNullValue() {
        map.containsValue(null);
    }

    /*
     * Negative tests (putAllValues)
     */
    @Test(expectedExceptions=RuntimeException.class)
    public void putAllValuesWithNullList() {
        map.putAllValues(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putAllValuesWithNullElement() {
        map.putAllValues(asList(alice, null, charlie));
    }

    /*
     * Negative tests (put)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void putWithBadAssociation() {
        map.put(1, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putWithNullKey() {
        map.put(null, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putWithNullValue() {
        map.put(1, null);
    }

    /*
     * Negative tests (putValue)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void putValueWithNullValue() {
        map.putValue(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putValueWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        try {
            indexMap.putValue(alice);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Negative tests (putIfAbsent)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void putIfAbsentWithBadAssociation() {
        map.putIfAbsent(1, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putIfAbsentWithNullKey() {
        map.putIfAbsent(null, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putIfAbsentWithNullValue() {
        map.putIfAbsent(1, null);
    }

    /*
     * Negative tests (putValue)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void putIfAbsentValueWithNullValue() {
        map.putIfAbsentValue(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void putIfAbsentValueWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        try {
            indexMap.putIfAbsentValue(alice);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Negative tests (replace)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceWithBadAssociation() {
        map.replace(1, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceWithNullKey() {
        map.replace(null, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceWithNullValue() {
        map.replace(1, null);
    }

    /*
     * Negative tests (replaceValue)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueWithNullValue() {
        map.replaceValue(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        map.putValue(alice);
        indexMap.keyDelegate().putAll(map.keyDelegate());
        indexMap.valueDelegate().putAll(map.valueDelegate());
        try {
            indexMap.replaceValue(alice);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Negative tests (conditional replace)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceConditionallyWithBadAssociation() {
        map.replace(1, bob, charlie);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceConditionallyWithNullKey() {
        map.replace(null, bob, charlie);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceConditionallyWithNullOldValue() {
        map.replace(1, null, charlie);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceConditionallyWithNullNewValue() {
        map.replace(1, bob, null);
    }

    /*
     * Negative tests (conditional replaceValue)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueConditionallyWithNullOldValue() {
        map.replaceValue(null, charlie);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueConditionallyWithNullNewValue() {
        map.replaceValue(bob, null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueConditionallyWithDifferentPrimaryKeys() {
        map.replaceValue(bob, alice);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void replaceValueConditionallyWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        map.putValue(alice);
        indexMap.keyDelegate().putAll(map.keyDelegate());
        indexMap.valueDelegate().putAll(map.valueDelegate());
        try {
            indexMap.replaceValue(alice, alice2);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Negative tests (remove)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeWithNullKey() {
        map.remove(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        map.putValue(alice);
        indexMap.keyDelegate().putAll(map.keyDelegate());
        indexMap.valueDelegate().putAll(map.valueDelegate());
        try {
            indexMap.remove(alice.id);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Negative tests (conditional remove)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeConditionallyWithNullKey() {
        map.remove(null, bob);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeConditionallyWithNullValue() {
        map.remove(1, null);
    }

    /*
     * Negative tests (removeValue)
     */
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeValueWithNullValue() {
        map.removeValue(null);
    }
    @Test(expectedExceptions=IllegalStateException.class)
    public void removeValueWithFailure() {
        IndexMap<Object, User> indexMap = createFailingIndexMap();
        map.putValue(alice);
        indexMap.keyDelegate().putAll(map.keyDelegate());
        indexMap.valueDelegate().putAll(map.valueDelegate());
        try {
            indexMap.removeValue(alice);
        } finally {
            assertFalse(((ReentrantStripedLock) indexMap.lock).isLocked(alice.id));
        }
    }

    /*
     * Concurrent tests
     */
    @Test
    public void concurrent() throws InterruptedException {
        final int nThreads = 10;
        final int variations = 15;
        final Queue<User> users = createUsers(nThreads, variations);
        timeTasks(nThreads, new Runnable() {
            public void run() {
                for (int i=0; i<variations/3; i++) {
                    map.putValue(users.poll());
                    map.removeValue(users.poll());
                    Set<Object> keys = map.extractor().apply(users.poll()).getAll();
                    map.get(Iterables.get(keys, random.nextInt(3)));
                }
            }
        });
        for (User user : map.values()) {
            Set<Object> keys = map.extractor().apply(user).getAll();
            assertKeysFound(keys, user);
            map.removeValue(user);
        }
        assertEmpty();
    }

    private Queue<User> createUsers(int users, int variations) {
        List<User> list = new ArrayList<User>();
        for (int i=1; i<=users; i++) {
            for (int j=1; j<=variations; j++) {
                String name = "name-" + i + (random.nextBoolean() ? "" : ":" + j);
                String email = "email-" + i + (random.nextBoolean() ? "" : ":" + j);
                String login = "login-" + i + (random.nextBoolean() ? "" : ":" + j);
                list.add(new User(i, name, email, login));
            }
        }
        shuffle(list);
        return new ConcurrentLinkedQueue<User>(list);
    }

    private void assertEmpty() {
        assertTrue(map.isEmpty());
        assertEquals(map.size(), 0);
        assertEquals(map.values(), emptyList());
        assertEquals(map.keySet(), emptySet());
        assertEquals(map.entrySet(), emptySet());
        assertNotFound(alice, bob, charlie);
    }

    private void assertContainsOnly(User... users) {
        for (User user : users) {
            assertTrue(map.containsValue(user));
            Set<Object> keys = map.extractor().apply(user).getAll();
            assertEquals(keys.size(), 3);
            assertKeysFound(keys, user);
            assertTrue(map.keySet().containsAll(keys));
            assertTrue(map.values().contains(user));
        }
        assertEquals(map.size(), users.length);
        assertEquals(map.values().size(), users.length);
        assertEquals(map.keySet().size(), 3 * users.length);
        assertEquals(map.entrySet().size(), 3 * users.length);
        for (Entry<Object, User> entry : map.entrySet()) {
            assertEquals(map.get(entry.getKey()), entry.getValue());
        }
    }

    private void assertNotFound(User... users) {
        for (User user : users) {
            assertFalse(map.containsValue(user));
            Set<Object> keys = map.extractor().apply(user).getAll();
            assertEquals(keys.size(), 3);
            assertKeysNotFound(keys);
        }
    }

    private void assertUpdate(User oldUser, User newUser) {
        assertNotSame(oldUser, newUser);
        Index<Object> oldIndex = map.extractor().apply(oldUser);
        Index<Object> newIndex = map.extractor().apply(newUser);
        assertEquals(newIndex.getPrimary(), oldIndex.getPrimary());
        assertEquals(map.get(newIndex.getPrimary()), newUser);

        Set<Object> diff = difference(oldIndex.getAll(), newIndex.getAll());
        assertKeysNotFound(diff);

        Set<Object> union = intersection(oldIndex.getAll(), newIndex.getAll());
        assertKeysFound(union, newUser);
    }

    private void assertKeysFound(Set<Object> keys, User user) {
        for (Object key : keys) {
            assertTrue(map.containsKey(key), "Key not found: " + key);
            assertEquals(map.get(key), user);
        }
    }

    private void assertKeysNotFound(Set<Object> keys) {
        for (Object key : keys) {
            assertNull(map.get(key));
            assertFalse(map.containsKey(key));
        }
    }

    /**
     * Creates an index map that fails during internal processing. This is used to assert that the lock is released.
     */
    private IndexMap<Object, User> createFailingIndexMap() {
        return new IndexMap<Object, User>(new ConcurrentHashMap<Object, User>(),
                                          new ConcurrentHashMap<Object, Index<Object>>(),
                                          new UserKeyExtractor()) {
            @Override
            protected void addIndex(Index<Object> index) {
                throw new IllegalStateException();
            }
            @Override
            protected void removeIndex(Index<Object> index) {
                throw new IllegalStateException();
            }
            @Override
            protected void updateIndex(Index<Object> oldIndex, Index<Object> newIndex) {
                throw new IllegalStateException();
            }
        };
    }

    private static final class User {
        final int id;
        final String name;
        final String email;
        final String loginName;

        public User(int id, String name, String email, String loginName) {
            this.id = greaterThan(0, id);
            this.name = notBlank(name);
            this.email = notBlank(email);
            this.loginName = notBlank(loginName);
        }
        @Override
        public String toString() {
            return String.format("%d: name=%s, loginName=%s, email=%s", id, name, loginName, email);
        }
    }

    private static final class UserKeyExtractor implements Function<User, Index<Object>> {
        public Index<Object> apply(User user) {
            return new Index<Object>(user.id, user.email, user.loginName); // 3 keys
        }
    }
}
```