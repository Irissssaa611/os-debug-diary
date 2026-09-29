## Async in Rust

1. **rust 异步的核心特点**

- **惰性的**，只有通过 `.await` 才会被第一次执行；

- **不强制依赖特定运行时**，这样用户可以根据需要选择合适的运行时（tokio、async-std、embassy）；
- **无栈协程**，通过编译时期将所有状态计算并排布生成状态机，实现无栈协程，内存开销极低；

2. **rust 异步的常见创建方式**

- async fn

```rust
async fn foo() -> u8{
    let x = bar().await;	// bar()也是异步函数
    x + 1
}
```

- async block

```rust
let foo = async {
    let x = bar().await;
    x + 1
};	// 立即创建 Future，只能调用1次
```

- async closure

```rust
let foo = async |x: u8|{
    bar(x).await
};	// 当调用 foo(x) 时才会被创建，可以多次调用进行创建，并且可以传参
```

- impl Future for（手动实现）

```rust
struct foo;
impl Future for foo {
    type Output = u8;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u8> {
        Poll:Ready(1)
    }
}
```

3. **调用/驱动 Future 的主要方式**

- 在 async 上下文中`.await`（同步函数中不能使用`.await`）

```rust 
async fn foo() -> u8 {
    let x = bar().await;	// bar()也是异步函数
    x + 1
}
```

- `#[tokio::main]`：异步入口

```rust
#[tokio::main]
async fn main(){
    let x = foo().await;
    println!("{x}");
}
```

- `block_on`：同步函数中驱动异步

```rust
fn main() {
    let x = future::executor::block_on(foo());
    println!("{x}");
}
// block_on 会阻塞运行异步函数
```

- `tokio::spawn`：交给运行时并发执行（可实现同步函数中驱动异步）

```rust
#[tokio::main]
fn main() {
    let foo = bar();
    tokio::spawn(foo);	// 将异步函数交给运行时，不会阻塞当前线程
}
```

- 还有一些其他的调用方式，这里不赘述（tokio::join!(a(), b())、futures::future::join_all等）。

3. **Rust 的无栈协程**

rust 的 future 没有自己独立的栈，它借用当前线程栈执行，一旦返回，线程栈上不可见，状态均保存在状态机中。对于其他语言的有栈协程来说，其调试接近于线程，而无栈协程的调用栈将不完整，需要工具。

4. **Rust 异步的运行时、调度器、reactor、Waker**

note：由于rust的广义运行时是轻量级的，所以在rust的语境下，运行时就是指异步运行时（tokio、async-std等）

- Runtime：顶级统筹容器，将调度器、定时器、线程池、reactor等打包为一个统一的运行上下文；
- Executor：Runtime 的一部分，维护一个任务就绪队列；
- Reactor：IO驱动，监听 epoll/kqueue/IOCP，IO 就绪时唤醒任务；
- Waker：Future 与运行时之间的唤醒桥梁；

```text
一个异步任务的整体流程：
executor poll 顶层 Future
  → 一路向下 poll
    → 最底层 IO Future 返回 Pending
      → 把 Waker 注册到 reactor
  ← 一路向上返回 Pending
executor 挂起任务，去跑别的
... IO 就绪，reactor 调用 Waker::wake ...
executor 再次 poll 顶层 Future
  → 从上次 .await 点恢复
  → 这次返回 Ready
```

















