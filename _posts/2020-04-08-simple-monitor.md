---
layout: default
title: 用Golang实现简单的网页版监控
---
# 目标
自己动手实现一个可在网页上查看系统命令执行过程的监控。

为了实现该监控功能，有几个问题需要解决：

    1.怎样在程序中执行系统命令？
    2.怎样把结果返回到网页端？
    3.怎样处理异步的情形？

# 1.怎样在程序中执行系统命令？
## 1.1 os/exec package
golang 提供了 os/exec 包，可以用于执行系统命令。

```golang
// main
// 仅演示Linux环境下的情景
func main() {
	var stdout = new(bytes.Buffer)
	var stderr = new(bytes.Buffer)
	var err error

	// Linux环境
	fmt.Println("first command")
	cmd := exec.Command("ls")
	cmd.Stdout = stdout
	cmd.Stderr = stderr
	if err = cmd.Run(); err != nil {
		fmt.Printf("%v", err)
	}
	fmt.Println(stdout.String())

	stdout.Reset()

	fmt.Println("second command")
	cmd = exec.Command("ls", "-a")
	cmd.Stdout = stdout
	cmd.Stderr = stderr
	if err = cmd.Run(); err != nil {
		fmt.Printf("%v", err)
	}
	fmt.Println(stdout.String())

	stdout.Reset()

	fmt.Println("third command")
	// 错误的方式
	cmd = exec.Command("ls -a")
	cmd.Stdout = stdout
	cmd.Stderr = stderr
	if err = cmd.Run(); err != nil {
		fmt.Printf("%v", err)
	}

	fmt.Println(stdout.String())
}
```
我们可以写一个短小的例子来测试 exec.Command。
```golang
// echohello
func main() {
    fmt.Println("hello world")
}
```
如果这个小程序编译成可执行文件 echohello 与上面的例子 main 在同一个文件夹中，main 需要在路径上做一些处理，才能正确执行 echohello。
```golang
// 先获取 main 的执行路径
execPath, err := os.Executable()
if err != nil {
    fmt.Printf("os.Executable error %v", err)
    return
}

// 拼接出 echohello 的执行路径
fullPath := filepath.Join(filepath.Dir(execPath), "echohello")
cmd = exec.Command(fullPath)
```
## 1.2 处理多个分批次返回的情况
我们把 echohello 做一些小改动
```golang
// echohello
func main() {
    for i := 1; i < 3; i++ {
        fmt.Println("hello world")
        time.Sleep(time.Second)
    }
}
```
我们再次用 main 来执行 echohello，发现等待大约3秒钟之后，3行 "hello world" 同时显示出来，有时候我们想要这样的效果，echohello 每次输出 "hello world"都能在屏幕上及时展示，这需要对 main 程序做一些小改动。
```golang
func AsyncCommand(ch chan string, Args []string, isLocal bool) (err error) {
	if isLocal {
		if execPath, err := os.Executable(); err != nil {
			fmt.Printf("os.Executable error %v", err)
		} else {
			Args[0] = filepath.Join(filepath.Dir(execPath), Args[0])
		}
	}

	cmd := exec.Command(Args[0], Args[1:]...)

	// 命令的错误输出和标准输出都连接到同一个管道
	stdout, err := cmd.StdoutPipe()
	cmd.Stderr = cmd.Stdout

	if err != nil {
		return err
	}

	if err = cmd.Start(); err != nil {
		return err
	}
	// 从管道中实时获取输出并打印到终端
	tmp := make([]byte, 1024)
	for {
		if n, err := stdout.Read(tmp); n == 0 || err != nil {
			break
		} else {
			ch <- string(tmp)
		}
	}

	err = cmd.Wait()
	return
}

func main() {
	ch := make(chan string)
	go func() {
		for {
			if s, ok := <- ch; ok {
				fmt.Print(s)
			} else {
				break
			}
		}
	}()

	_ = AsyncCommand(ch, []string{"echohello2"}, true)
}
```
用 StdoutPipe 方法，可以及时获得执行结果。

## 1.3 要点
    1.多个 args 需要用数组传入，一个字符串中包含多个 args 传入无效。
    2.用 StdoutPipe 结合 Channel 可以实现及时输出的效果。
    3.stdout.Read(tmp) 这一行代码，如果 tmp 空间太小，会执行多次 Read。