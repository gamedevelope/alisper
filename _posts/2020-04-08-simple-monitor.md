---
layout: default
title: 用Golang实现简单的网页版监控
---
# 目标
自己动手实现一个可在网页上查看系统命令执行过程的监控。

为了实现该监控功能，我们一开始会有几点疑问：

    1.怎样在程序中执行系统命令？
    2.怎样把结果返回到网页端？
    3.怎样处理异步的情形？

# 1.怎样在程序中执行系统命令？
golang 提供了 os/exec 包，可以用于执行系统命令。

```golang
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
