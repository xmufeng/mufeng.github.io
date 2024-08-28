---
title: Beanstalkd初探
---
## 简介
Beanstalkd是一个高性能、轻量级的分布式内存队列系统。Beanstalkd支持`指定任务的优先级（priority）`，`延迟处理（delay）`，`持久化（persistent data）`，`消息预留（buried）`，`任务超时重发（time to run）`等特性。
## 安装
以Ubuntu为例
```bash
apt install beanstalkd
```
安装完成后，服务默认监听`:11300`
## 示例
### 生产者
```golang
package bslearn

import (
	"fmt"
	"testing"
	"time"

	"github.com/beanstalkd/go-beanstalk"
)

func TestProducer(t *testing.T) {
	conn, err := beanstalk.Dial("tcp", "127.0.0.1:11300")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer conn.Close()
    
	tube := beanstalk.NewTube(conn, "mytube")
    // 向mytube中发送延迟任务
	if id, err := tube.Put([]byte("Hello beanstalk"), 0, time.Second*10, time.Second); err != nil {
		fmt.Println(err)
	} else {
		fmt.Println(id)
	}
}
```

### 消费者
```golang
package bslearn

import (
	"fmt"
	"testing"

	"github.com/beanstalkd/go-beanstalk"
)

func TestConsumer(t *testing.T) {
	conn, err := beanstalk.Dial("tcp", "127.0.0.1:11300")
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()

	if tubes, err := conn.ListTubes(); err != nil {
		fmt.Println(err)
	} else {
		for _, tube := range tubes {
			fmt.Println(tube)
		}
        // 使用mytube接收任务
		mytube := beanstalk.NewTube(conn, "mytube")
        // 获取处理延迟状态的任务
		if id, _, err := mytube.PeekDelayed(); err != nil {
			fmt.Println(err)
		} else {
			stats, _ := conn.StatsJob(id)
			for k, v := range stats {
				fmt.Println(k, "=", v)
			}
		}
        // 获取处理就绪状态的任务
		if id, body, err := mytube.PeekReady(); err != nil {
			fmt.Println(err)
		} else {
			fmt.Println("id=", id, ";body=", string(body))
            // 任务处理完成后，删除
			conn.Delete(id)
		}
	}
}
```
