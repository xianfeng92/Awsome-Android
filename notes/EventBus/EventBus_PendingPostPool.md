## PendingPost 对象池

subscription 和 event 的实体类，其内部维护着一个 pendingPost 对象池。

通过其静态方法 obtainPendingPost 可以获取 PendingPost 对象，源码如下所示：

```
    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        return new PendingPost(event, subscription);
    }
```

1. 如果 pendingPostPool 中有 pendingPost 对象，则直接从 pendingPostPool 中获取 pendingPost。

2. 只有当 pendingPostPool 中没有 pendingPost 对象时，才会去重新创建一个 PendingPost 对象。


通过对象池缓存不用的对象，减少下次创建的性能消耗。


PendingPost 中还有一个静态方法 releasePendingPost，其源码如下：

```
    static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // Don't let the pool grow indefinitely
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }
```

当一个 PendingPost 对象不需要使用时，可以通过该方法将 PendingPost 放到 pendingPostPool 中，以便下次调用
obtainPendingPost 来重复利用该对象。









































