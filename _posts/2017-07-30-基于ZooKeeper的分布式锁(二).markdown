---
layout: post
title:  "基于ZooKeeper的分布式锁(二)"
date:   2017-07-30 08:00:00 +0800
categories: 中间件
---
前一篇文章介绍了ZooKeeper实现简单的分布式锁算法，但是当并发量很高，锁冲突机率大的情况下这种算法会导致羊群效应。解决羊群问题的算法，其实Zookeeper官网就有:

<br/>

> 1. Call create( ) with a pathname of "_locknode_/lock-" and the sequence and ephemeral flags set.
> 2. Call getChildren( ) on the lock node without setting the watch flag (this is important to avoid the herd effect).
> 3. If the pathname created in step 1 has the lowest sequence number suffix, the client has the lock and the client exits the protocol.
> 4. The client calls exists( ) with the watch flag set on the path in the lock directory with the next lowest sequence number.
> 5. if exists( ) returns false, go to step 2. Otherwise, wait for a notification for the pathname from the previous step before going to step 2.

<br/>

基本思想是利用ZooKeeper的临时节点和有序节点，每个客户端在znode节点下面创建一个唯一子节点，每个子节点都有一个唯一的序列号，然后约定具有最小序列号的客户端持有锁；未获得锁的客户端，只需要监听比它自己的序列号小的前一个子节点即可。这样当锁释放的时候只会有一个客户端被唤醒，避免了羊群效应的发生。

<br/>

```java
private static final String SEPARATOR = "/";
private static final String NODE_NAME = "lock-";

public String lock(String clientId, String resource) throws KeeperException, InterruptedException {
    this.ensureResource(clientId, resource);

    try{
        String path = zk.create(root + resource + SEPARATOR + NODE_NAME, clientId.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

        this.waitUntilLocked(path, clientId, resource);
        return path;
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.lockOnConnectionLoss(clientId, resource);
        } else {
            throw e;
        }
    }
}

private String lockOnConnectionLoss(String clientId, String resource) throws KeeperException, InterruptedException {
    List<String> pathes = this.getChildren(resource);

    String myPath = findMyNodeOnConnectionLoss(pathes, clientId);

    if (null == myPath) {
        return this.lock(clientId, resource);
    }

    this.waitUntilLocked(myPath, clientId, resource);
    return myPath;
}
```

<br/>

resource参数是要加锁的资源标识，客户端会在它下面创建子节点。因为ZooKeeper不允许临时节点下面有子节点，所以`ensureResource`方法用于确保resource参数对应的znode节点必须存在。

`lock`方法里面创建的是一个EPHEMERAL_SEQUENTIAL的znode节点，它总是会成功的，所以不会发生`NodeExistsException`异常。不过链接丢失的情况依然需要单独处理，`lockOnConnectionLoss`方法会取出所有的子节点，然后判断前一次请求是否成功，如果没有成功的话，就进行重试；否则就进入下一步，判断自己是持有锁、还是等待。

<br/>

```java
private void waitUntilLocked(String myPath, String clientId, String resource)
        throws KeeperException, InterruptedException {
    while (true) {
        List<String> pathes = this.getChildren(resource);
        if (this.isLocked(myPath, pathes)) {
            return;
        } else {
            String target = this.findWatchPath(myPath, pathes, resource);
            this.watchAndWait(target);
        }
    }
}
```

<br/>

`waitUntilLocked`方法的作用就是判断自己是加锁成功，还是等待。`getChildren`方法返回所有的子节点，`isLocked`方法判断当前客户端创建的子节点myPath是否具有最小的序列号，如果是的话表示当前客户端加锁成功，直接返回。否则的话就通过`findWatchPath`方法找到需要监视的子节点，即找到比当前客户端创建的子节点的序列号小的前一个子节点，然后阻塞并监听该节点的删除事件，`watchAndWait`方法与简单锁的`listenLock`完全一样，只是方法命名不同而已。

锁的释放相关于简单锁的实现更容易些，因为每个客户端创建的都是唯一的子节点，所以链接丢失时只需要进行重试即可，不需要额外的判断了。

<br/>

```java
public void release(String path) throws InterruptedException, KeeperException {
    try{
        zk.delete(path, -1);
    }
    catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            this.release(path);
        } else {
            throw e;
        }
    }
}
```

<br/>

下面给出其它几个方法的实现，方便大家查看:

<br/>

```java
private List<String> getChildren(String resource) throws KeeperException, InterruptedException {
    try{
        List<String> rlt = zk.getChildren(root + resource, false);

        Collections.sort(rlt, new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                int i1 = DistributedLock.this.parseSequence(o1);
                int i2 = DistributedLock.this.parseSequence(o2);
                return i1 - i2;
            }
        });

        return rlt;
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.getChildren(resource);
        } else {
            throw e;
        }
    }
}

private boolean isLocked(String myPath, List<String> pathes) {
    int mySeq = this.parseSequence(myPath);
    int firstSeq = this.parseSequence(pathes.get(0));

    return mySeq == firstSeq;
}

private String findWatchPath(String myPath, List<String> pathes, String resource) {
    String prefix = root + resource + SEPARATOR;
    int start = prefix.length();

    String path;
    for (int i = pathes.size() -1; i >= 0; i--) {
        path = pathes.get(i);
        if (path.equals(myPath.substring(start))) {
            return prefix + pathes.get(i-1);
        }
    }

    throw new IllegalStateException();
}

private void watchAndWait(String target)
        throws KeeperException, InterruptedException {
    Semaphore s = new Semaphore(0);
    try{
        Stat stat = zk.exists(target, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType().equals(EventType.NodeDeleted)) {
                    s.release();
                }
            }
        });

        if (null != stat) {
            s.acquire();
        }
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            this.watchAndWait(target);
            return;
        } else {
            throw e;
        }
    }
}

private String findMyNodeOnConnectionLoss(List<String> pathes, String clientId)
        throws KeeperException, InterruptedException {
    if (null == pathes || pathes.isEmpty()) {
        return null;
    }

    String path;
    for (int i = pathes.size() -1; i >= 0; i--) {
        path = pathes.get(i);
        if (this.isClientIdMatch(path, clientId)) {
            return path;
        }
    }

    return null;
}

private boolean isClientIdMatch(String path, String clientId) throws KeeperException, InterruptedException {

    try{
        Stat stat = new Stat();
        byte[] data = zk.getData(path, false, stat);
        if (clientId.equals(new String(data))) {
            return true;
        }

        return false;
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.NONODE)) {
            return false;
        } else if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.isClientIdMatch(path, clientId);
        } else {
            throw e;
        }
    }
}
```

---

### 读写锁

将上面的实现稍作修改，就可以实现读写锁，这里引用ZooKeeper官网的描述。

加读锁的算法：

> 1. Call create( ) to create a node with pathname "locknode/read-". This is the lock node use later in the protocol. Make sure to set both the sequence and ephemeral flags.
> 2. Call getChildren( ) on the lock node without setting the watch flag - this is important, as it avoids the herd effect.
> 3. If there are no children with a pathname starting with "write-" and having a lower sequence number than the node created in step 1, the client has the lock and can exit the protocol.
> 4. Otherwise, call exists( ), with watch flag, set on the node in lock directory with pathname staring with "write-" having the next lowest sequence number.
> 5. If exists( ) returns false, goto step 2.
Otherwise, wait for a notification for the pathname from the previous step before going to step 2

加写锁的算法：

> 1. Call create( ) to create a node with pathname "locknode/write-". This is the lock node spoken of later in the protocol. Make sure to set both sequence and ephemeral flags.
> 2. Call getChildren( ) on the lock node without setting the watch flag - this is important, as it avoids the herd effect.
> 3. If there are no children with a lower sequence number than the node created in step 1, the client has the lock and the client exits the protocol.
> 4. Call exists( ), with watch flag set, on the node with the pathname that has the next lowest sequence number.
> 5. If exists( ) returns false, goto step 2. Otherwise, wait for a notification for the pathname from the previous step before going to step 2.

*附上源码地址*：[https://github.com/OuYangLiang/ZK-lock](https://github.com/OuYangLiang/ZK-lock)
