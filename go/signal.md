# signal包中的Notify和Stop函数
## signal.Notify
函数声明

```
func Notify(c chan<- os.Signal, sig ...os.Signal)
``` 
官方描述
> 1. Notify 让 signal 包将输入信号 sig 转发到 c (chan)，如果没有指定要转发的信号，会将所有信号转发
> 2. signal包向 c 转发消息时，不会阻塞（如果 c 阻塞了, signal 会放弃转发），因此需要保证 c 有足够的缓存空间
> 3. 允许同一个 c 多次调用 Notify 函数，会根据传入的信号扩展需要转发的信号（不会减少原来的信号，只有 Stop 方法可以减少）
> 4. 允许不同的 c 调用 Notify 函数，不会互相影响

## signal.Stop

函数声明
```
func Stop(c chan<- os.Signal)
``` 
官方描述
> 1. Stop 让 signal 包停止转发信号到 c
> 2. 会取消在此之前通过 Notify 对 c 指定的所有转发信号
> 3. 当 Stop 函数返回时，会确保 c 不再接收信号

##示例

```
func main() {
	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	for {
		select {
		case s := <- ch :
			signal.Stop(ch)
			fmt.Printf("Interrupted, %v\n", s)
			return
		default:
			fmt.Println("Running")
		}
		time.Sleep(3 * time.Second)
	}
}
```
补充说明
> 1. 函数每 3s 输出 "Running"，直到进程被中断
> 2. ch 缓存大小指定为1，这是考虑到上面讲的 Notify 官方描述第2条，否则很可能无法及时收到信息