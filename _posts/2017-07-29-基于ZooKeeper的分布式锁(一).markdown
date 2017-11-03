---
layout: post
title:  "基于ZooKeeper的分布式锁(一)"
date:   2017-07-29 08:00:00 +0800
categories: 中间件
keywords: 分布式锁,zookeeper,羊群效应
description: 介绍如何使用zookeeper实现分布式锁
---
借助ZooKeeper的临时节点，很容易实现分布式锁。为了获得一个锁，客户端尝试创建一个znode节点，如果znode节点创建成功，就表示客户端获得了锁并可以继续执行临界区中的代码；如果znode节点创建失败，就监听znode节点的变化，并在检测到znode节点被删除时再次创建节点来获得锁。如果要实现一个非阻塞锁的话，当znode节点创建失败时，就直接返回失败而不是去监听。

<br/>

```java
public boolean tryLock(String clientId, String resource)
        throws KeeperException, InterruptedException {
    try{
        zk.create(root + resource, clientId.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);

        return true;
    } catch(KeeperException e) {
        if (e.code().equals(KeeperException.Code.NODEEXISTS)) {
            return false;
        } else if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.tryLockWhenConnectionLoss(clientId, resource);
        } else {
            throw e;
        }
    }
}
```

<br/>

`tryLock`方法是非阻塞式锁的实现。参数`clientId`是客户端的标识，znode节点的内容，必须要全局唯一，作用后面会介绍，`resource`是znode的名称，用来区别不同的锁。这个方法的逻辑其实很简单，但有几点需要说明。

第一，创建的znode节点必须是临时的（CreateMode.EPHEMERAL），防止客户端崩溃时导致锁永远无法释放。

第二，如果znode节点已经存在（其它客户端已经持有锁），这时创建会失败，ZooKeeper API是以异常的形式来来处理的，所以我们需要捕获`NodeExistsException`异常，并返回false。

第三，ZooKeeper的会话与服务端是通过心跳保持连接的，当心跳超时或者链接丢失的时候客户端的请求会抛出Connection Loss异常，ZooKeeper客户端会进行自动重连，所以这种情况我们往往需要进行重试。链接丢失可能发生在客户端向服务端请求的时候，也可能发生在服务端向客户端响应的时候，所以这个时候我们并不知道前一次请求是否已经执行成功。正因为这个原因，在链接丢失的时候我们不能简单的进行重试，而是要先判断前一次请求是否成功，也就是`tryLockWhenConnectionLoss`方法要做的事情。

<br/>

```java
private boolean tryLockWhenConnectionLoss(String clientId, String resource)
        throws KeeperException, InterruptedException {

    try{
        zk.create(root + resource, clientId.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);

        return true;
    } catch(KeeperException e) {
        if (e.code().equals(KeeperException.Code.NODEEXISTS)) {
            return this.checkNode(clientId, resource);
        } else if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.tryLockWhenConnectionLoss(clientId, resource);
        } else {
            throw e;
        }
    }
}
```

<br/>

除了对`NodeExistsException`异常的处理不同外，`tryLockWhenConnectionLoss`方法与`tryLock`方法完全一样。前面提到过，在链接丢失的时候我们并不知道服务端是否已经处理成功。如果已经成功了，在进行重试的时候将会收到`NodeExistsException`异常，当然，失败也可能是因为其它的客户端已经持有锁，所以这时需要判断已经存在的znode节点是否是当前客户端前一次请求所创建的。也就是`checkNode`方法要做的事情。

<br/>

```java
private boolean checkNode(String clientId, String resource) throws KeeperException, InterruptedException {
    try {
        Stat stat = new Stat();
        byte[] data = zk.getData(root + resource, false, stat);
        if (clientId.equals(new String(data))) {
            return true;
        }

        return false;
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.NONODE)) {
            return this.tryLock(clientId, resource);
        } else if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            return this.checkNode(clientId, resource);
        } else {
            throw e;
        }
    }
}
```

<br/>

`checkNode`方法查询znode节点的内容并与当前的客户端标枳进行比较，如果相同，则说明加锁成功，反之表示其它客户端已经持有锁。如果这时znode节点不存在，说明其它客户端已经释放了锁，我们需要重新尝试加锁，也就是上面对`NoNodeException`的处理。这也就是为什么我们在加锁的时候，需要把客户端的标识作为参数的原因。

至此，一个非阻塞式锁的实现已经完成了。阻塞锁的实现要稍微复杂一点，在加锁失败的时候，需要去监听znode的变化，并在znode节点删除时重新进行尝试。

<br/>

```java
public void lock(String clientId, String resource) throws KeeperException, InterruptedException {
    while (true) {
        if (this.tryLock(clientId, resource)) {
            return;
        }

        this.listenLock(resource);
    }
}

private void listenLock(String resource) throws InterruptedException, KeeperException {
    Semaphore s = new Semaphore(0);

    try {
        Stat stat = zk.exists(root + resource, new Watcher() {
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

    } catch (KeeperException e) {
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            this.listenLock(resource);
            return;
        } else {
            throw e;
        }
    }
}
```

<br/>

阻塞锁实现的重点是`listenLock`方法，通过ZooKeeper.exists方法在znode节点上设置一个监视点，监听znode节点的删除事件，并通过一个信号量Semaphore来实现阻塞的效果（这里其实也可以使用CountDownLatch）。如果监视点设置成功，就阻塞等待锁的释放；如果设置失败，说明其它持有锁的客户端在当前客户端设置监视点的时候已经释放了锁，这时只需重新尝试加锁操作，即stat为null时，直接从`listenLock`方法返回`lock`方法。

锁的释放操作也很简单，只需要删除znode节点即可。

<br/>

```java
public void release(String clientId, String resource) throws KeeperException, InterruptedException {
    try{
        zk.delete(root + resource, -1);
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            this.checkRelease(clientId, resource);
        } else {
            throw e;
        }
    }
}

private void checkRelease(String clientId, String resource) throws KeeperException, InterruptedException {
    try {
        Stat stat = new Stat();
        byte[] data = zk.getData(root + resource, false, stat);
        if (clientId.equals(new String(data))) {
            this.release(clientId, resource);
        }
    } catch(KeeperException e){
        if (e.code().equals(KeeperException.Code.NONODE)) {
            return;
        } else if (e.code().equals(KeeperException.Code.CONNECTIONLOSS)) {
            this.checkRelease(clientId, resource);
        } else {
            throw e;
        }
    }
}
```

<br/>

ZooKeeper.delete同样可能发生链接丢失的情况，与之前的情况类似，在重试的时候时候需要判断之前的请求是否成功。在`checkRelease`方法中，通过查询znode节点的内容并与当前客户端的标识进行比较，如果相同，说明之前的请求没有成功，需要重新删除；如果不同，说时之前的请求已经成功，但是当前锁已经被另一个客户端持有，直接返回即可。如果znode节点不存在，说明之前请求已经成功，当前锁牌空闲状态，直接返回即可。

---

### 羊群效应

对于并发量小，或者锁冲突小的情况下，本文中介绍的锁实现可以很好的工作。但是当并发量很高，锁冲突机率会变大，这种锁的实现就会产生问题了。如果有大量的客户端都尝试对同一个资源加锁，即对同一个znode节点设计监视点，当锁被释放、znode节点被删除时，ZooKeeper服务端会产生一个尖峰的通知，该尖峰可能会导致网络的阻塞，引起一系列的问题，这种现象就是**羊群效应**。另一方面，唤醒全部的客户端，而实际上它们之间只会有一个能成功加锁，也是一种不合理的方式。

使用另一种算法可以成功避免羊群效应的发生，请参考[基于ZooKeeper的分布式锁(二)]({{site.baseurl}}/2017/07/基于ZooKeeper的分布式锁(二).html)。

*附上源码地址*：[https://github.com/OuYangLiang/ZK-lock](https://github.com/OuYangLiang/ZK-lock)
