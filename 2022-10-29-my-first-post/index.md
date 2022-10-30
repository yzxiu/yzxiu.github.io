# My First Post




## 一、配置

参考：[https://hugoloveit.com/zh-cn/posts/](https://hugoloveit.com/zh-cn/posts/)





## 二、图片

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


