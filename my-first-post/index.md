# My First Post

{{< load-photoswipe >}}

## My First Post

搞了一个下午，终于弄的差不多了，主要用了[ LoveIt](https://github.com/dillonzq/LoveIt) 这个主题，自己加了 `photoswipe` 用于浏览图片。

[配置参考](https://hugoloveit.com/zh-cn/posts/)



## 图片


![涂鸦](https://raw.githubusercontent.com/yzxiu/images/master/blog/graffiti-569265_1280.jpg "涂鸦")

![img](https://raw.githubusercontent.com/yzxiu/images/master/blog/architecture.png "containerd")





## 代码

```go
package main

import "fmt"

func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}

func main() {
	fmt.Println(split(17))
}
```



## 流程图

```markdown
{{< mermaid >}}
graph LR;
    A[Hard edge] -->|Link text| B(Round edge)
    B --> C{Decision}
    C -->|One| D[Result one]
    C -->|Two| E[Result two]
{{< /mermaid >}}
```

