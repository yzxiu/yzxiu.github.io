# My First Post




## 一、配置

参考：[https://hugoloveit.com/zh-cn/posts/](https://hugoloveit.com/zh-cn/posts/)





## 二、图片

### 2.1 图片上传
typora配置：

![image-20221030223715217](https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-223715.png "typora setting")

图片插入：

```markdown
# 使用typora直接拖拽即可
![image-20221030223715217](https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-223715.png "typora setting")
```



picgo-core插件：

```bash
# https://connor-sun.github.io/posts/38835.html
picgo install super-prefix
```

picgo-core配置：

```json
{
  "picBed": {
    "github": {
      "repo": "*****",
      "token": "*****",
      "path": "/",
      "customUrl": "",
      "branch": "blog"
    },
    "current": "github",
    "uploader": "github"
  },
  "picgoPlugins": {
    "picgo-plugin-super-prefix": true
  },
  "picgo-plugin-super-prefix": {
    "prefixFormat": "YYYY-MM/",
    "fileFormat": "YYYYMMDD-HHmmss"
  }
}
```

### 2.2 小图展示
小图片如果使用 `![image-20221030223715217](https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png "small image")`的方式插入，会拉伸宽屏，如下：

![image-20221030223715217](https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png "small image")

1. 可以改用 `<img>` 的方式 ，并使用 `{{</* style "text-align:center;" */>}}` 控制位置，如下：
```markdown
{{</* style "text-align:center;" */>}}
<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 80%;" >
{{</* /style */>}}
```
呈现的效果如下：
{{< style "text-align:center;" >}}
<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 80%;" >
{{< /style >}}


2) 图文混排，使用`style="zoom: 20%;"`控制大小，如下：
```markdown
图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 10%;" />图文混排

图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 20%;" />图文混排

图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 30%;" />图文混排
```

呈现的效果如下：

图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 10%;" />图文混排

图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 20%;" />图文混排

图文混排<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-10/20221030-235308.png" style="zoom: 30%;" />图文混排

## 三、代码

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


