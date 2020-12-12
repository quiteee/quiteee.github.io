# Go语言实战 | 并发模式 - Pool

> 《Go语言实战》笔记 

这篇文章展示如何使用有缓冲的通道实现资源池，来管理在goroutine之间共享的资源。

这种模式在需要共享一组静态资源的情况下非常有用，如共享数据库连接或者内存缓冲区。


## 自定义Pool
使用该资源池获取资源时，会优先使用之前被释放的资源，可获取资源的数量没有上线。

---

下面开始进入正题，先构建一个资源结构体

```go
// 定义Pool
type Pool struct {
    m         sync.Mutex
    resources chan io.Closer
    factory   func() (io.Closer, error) // 资源的创建工厂
    closed    bool
}
```

`m`是一个互斥锁，保证在多个goroutine访问资源池时，池内的值是安全的。

`resources`是一个接口类型的通道，用来保存共享的资源。

`factory`是一个函数类型，用于创建资源，具体的实现细节由Pool的使用者提供。

`closed`标志Pool是否被关闭

定义一个Error

```go
var ErrPoolClosed = errors.New("Pool has been closed.")
```

单独定义这个error的目的是为了让调用者从返回的多种error中可以识别出这个特定的error。

接下来是初始化Pool的方法，即New函数，由调用者指定构建资源的函数，以及资源池的大小

```go
func New(fn func() (io.Closer, error), size int) (*Pool, error) {
    if size <= 0 {
        return nil, errors.New("Size value too small.")
    }
    return &Pool{
        m:         sync.Mutex{},
        resources: make(chan io.Closer, size),
        factory:   fn,
    }, nil
}
```

下面是Acquire方法，这个方法可以让调用方从Pool中获取资源

```go
func (p *Pool) Acquire() (io.Closer, error) {
    select {
    // 优先返回空闲资源
    case r, ok := <-p.resources:
        if !ok {
            return nil, ErrPoolClosed
        }
        return r, nil
    // 没有空闲资源就只能新建
    default:
        log.Println("Resources is empty, create new")
        return p.factory()
    }
}
```

使用完资源后需要释放资源，下面是Release方法

```go
func (p *Pool) Release(r io.Closer) {
    p.m.Lock()
    defer p.m.Unlock()

    // 在Pool已经关闭的情况下，直接关闭资源
    if p.closed {
        r.Close()
        return 
    }

    select {
    // 如果通道没满，作为空闲资源放入通道
    case p.resources <- r:
        log.Println("Release: to pool")
    // 如果通道满了，直接关闭资源
    default:
        log.Println("Release: close")
        r.Close()
    }
    return
}
```

Release的时候，要先判断Pool是否已经关闭。

`closed`字段不是线程安全的，在下面介绍的`Close()`方法中可能被其他线程修改，所以需要通过互斥锁保证线程安全。

一旦程序不再使用资源池，需要调用Close方法关闭资源池。

```go
func (p *Pool) Close() {
    p.m.Lock()
    defer p.m.Unlock()

    //如果Pool已经被关闭，直接返回
    if p.closed {
        return
    }

    p.closed = true // 关闭池

    // 先关闭通道，否则容易产生死锁
    close(p.resources)

    // 逐个关闭通道内的空闲资源
    for r := range p.resources {
        r.Close()
    }
}
```

在实现Pool的时候，需要考虑到一些边界条件，例如
1. 初始化的时候，`size <= 0`
2. Acquire资源的时候，资源池已经关闭
3. Release资源的时候，资源池已经关闭 or 资源池的缓冲通道已经满了

## 自定义Pool的使用
---

下面介绍一下如何使用上面实现的Pool

```go

const (
    // 使用的goroutine数量上限
    maxGoroutines = 10
    // 资源池中的资源数量
    poolResources = 2
)

type dbConnection struct {
    ID int32
}

func (dbConn *dbConnection) Close() error {
    log.Println("Close: Connection", dbConn.ID)
    return nil
}

var idCounter int32

func createConnection() (io.Closer, error) {
    // 确保每个新建的conn有一个唯一ID
    id := atomic.AddInt32(&idCounter, 1)
    log.Println("Create: Connection", id)
    return &dbConnection{id}, nil
}

// 模拟数据库查询操作
func query(q int, pool *Pool) {
    // 获取资源
    conn, err := pool.Acquire()
    if err != nil {
        log.Println(err)
        return
    }
    defer pool.Release(conn)

    log.Printf("Query [%d] Get Conn [%d]\n", q, conn.(*dbConnection).ID)
    time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
    log.Printf("Conn [%d] Release\n", conn.(*dbConnection).ID)
}

func main() {
    var wg sync.WaitGroup
    wg.Add(maxGoroutines)

    p, err := New(createConnection, poolResources)
    if err != nil {
        log.Println(err)
        return
    }

    for i := 0; i < maxGoroutines; i++ {
        go func(i int) {
            query(i, p)
            wg.Done()
        }(i)
        // 这里设定sleep，是为了不让所有的goroutine几乎同时去query，这会导致在资源Release之前，所有goroutine已经通过create的方式拿到资源
        time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
    }
    
    // 等待线程执行结束，然后close
    wg.Wait()
    log.Println("Shutdown Program.")
    p.Close()
}

```

程序执行结果如下

```go
2020/12/12 15:52:25 Resources is empty, create new
2020/12/12 15:52:25 Create: Connection 1
2020/12/12 15:52:25 Query [0] Get Conn [1]
2020/12/12 15:52:25 Resources is empty, create new
2020/12/12 15:52:25 Create: Connection 2
2020/12/12 15:52:25 Query [1] Get Conn [2]            // 先创建了两个 conn
2020/12/12 15:52:26 Conn [2] Release
2020/12/12 15:52:26 Release: to pool                  // conn 2 释放后，进入pool
2020/12/12 15:52:26 Query [2] Get Conn [2]            // conn 2 被下一个query获取到
2020/12/12 15:52:26 Resources is empty, create new
2020/12/12 15:52:26 Create: Connection 3
2020/12/12 15:52:26 Query [3] Get Conn [3]            // 创建 conn 3
2020/12/12 15:52:26 Conn [2] Release
2020/12/12 15:52:26 Release: to pool
2020/12/12 15:52:26 Conn [1] Release
2020/12/12 15:52:26 Release: to pool                  // conn 2 和 conn 1 被释放, 此时 pool:【2，1】
2020/12/12 15:52:26 Query [4] Get Conn [2]            // conn 2 被获取，此时 pool:【1】
2020/12/12 15:52:26 Conn [3] Release
2020/12/12 15:52:26 Release: to pool                  // conn 3 被释放，此时 pool:【1，3】
2020/12/12 15:52:27 Conn [2] Release
2020/12/12 15:52:27 Release: close                    // conn 2 被释放，此时 pool 已满，所以 conn 2 被关闭
2020/12/12 15:52:27 Close: Connection 2
2020/12/12 15:52:27 Query [5] Get Conn [1]
2020/12/12 15:52:27 Query [6] Get Conn [3]            // conn 1 和 conn 3 被获取，此时 pool:【】
2020/12/12 15:52:27 Resources is empty, create new
2020/12/12 15:52:27 Create: Connection 4
2020/12/12 15:52:27 Query [7] Get Conn [4]            // 创建新资源 conn 4
2020/12/12 15:52:27 Conn [3] Release
2020/12/12 15:52:27 Release: to pool
2020/12/12 15:52:27 Conn [1] Release
2020/12/12 15:52:27 Release: to pool                  // conn 3 和 conn 1 被释放，此时 pool: 【3，1】
2020/12/12 15:52:27 Query [8] Get Conn [3]            // conn 3 被获取，此时 pool:【1】
2020/12/12 15:52:27 Conn [4] Release
2020/12/12 15:52:27 Release: to pool                  // conn 4 被释放，此时 pool: 【1，4】
2020/12/12 15:52:27 Query [9] Get Conn [1]
2020/12/12 15:52:28 Conn [1] Release
2020/12/12 15:52:28 Release: to pool                  // conn 1 被获取，又被释放，此时 pool: 【4，1】
2020/12/12 15:52:28 Conn [3] Release
2020/12/12 15:52:28 Release: close                    // conn 3 被释放，此时 pool 已满，所以 conn 3 被关闭
2020/12/12 15:52:28 Close: Connection 3
2020/12/12 15:52:28 Shutdown Program.                 // 程序结束，依次关闭 pool 内的资源
2020/12/12 15:52:28 Close: Connection 4
2020/12/12 15:52:28 Close: Connection 1
```

---

## sync.Pool

`sync.Pool` 在Go1.6及之后的版本中，标准库里自带的资源池的实现。

`sync.Pool`的资源池是无上限的（受内存大小影响），在高负载下可以动态地扩容，在不活跃时会对象池会收缩。

下面是用`sync.Pool`实现我们上面的例子的代码

```go
const (
    // 使用的goroutine数量上限
    maxGoroutines = 10
    // 资源池中的资源数量
    poolResources = 2
)

type dbConnection struct {
    ID int32
}

func (dbConn *dbConnection) Close() error {
    log.Println("Close: Connection", dbConn.ID)
    return nil
}

var idCounter int32

func createConnection() interface{} {
    id := atomic.AddInt32(&idCounter, 1)
    log.Println("Create: Connection", id)
    return &dbConnection{id}
}

func query(q int, pool *sync.Pool) {
    conn := pool.Get()
    log.Printf("Query [%d] Get Conn [%d]\n", q, conn.(*dbConnection).ID)

    defer pool.Put(conn)
    time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
    log.Printf("Conn [%d] Release\n", conn.(*dbConnection).ID)
}

func main() {
    var wg sync.WaitGroup
    wg.Add(maxGoroutines)
    p := &sync.Pool{
        New: createConnection,
    }
    for i := 0; i < maxGoroutines; i++ {
        go func(i int) {
            query(i, p)
            wg.Done()
        }(i)
        time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
    }
    wg.Wait()
    log.Println("Shutdown Program.")
}
```

---
## END

