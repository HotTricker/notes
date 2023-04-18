# markdown

- [markdown](#markdown)
  - [复习](#复习)
  - [instance](#instance)
    - [创建标题（多级）](#创建标题多级)

## 复习

> 1. 标题，段落，换行
> 2. 加粗、斜体、加粗+斜体、删除线、代码
> 3. 块引用，多个段落块引用，嵌套块引用
> 4. 列表，嵌套列表，无序列表，人物列表
> 5. 分割线
> 6. 链接，直接显示的链接，引用类型的链接，标签带格式的链接，图片，图片加链接
> 7. 转义字符
> 8. 表格
> 9. 围栏代码块
> 10. 数学公式：行内、行间、上下标

## instance

### 创建标题（多级）

创建段落要用
空白行

比如这样

两个空格然后回车为  
换行，仍为同一个段落
这样可以**加粗**，这样是*斜体*，这样是***同时***,这样是~~删除~~

>块引用
>
>多个段落块引用
>>嵌套块引用

1. 列表
1. 第二项（可以用1.开头）
1. 第三项
   1. 嵌套
   2. 嵌套2

- 无序列表（+ * 也可以）
  - 多级列表

- [ ] 任务列表
- [x] 完成的（可以用alt+c快速切换）

代码块  
`import python`

***

单独的一行上三个*** ___ --- 都可以作为分割线，前后最好都添加空白行

这是一个链接 [百度](https://www.baidu.com)  
这是一个直接显示的链接 <https://www.baidu.com> <874872938@qq.com>  
**[加粗的](database.md)**
*[斜体的](database.md)*
[`代码格式的`](database.md)  
引用类型的链接[instance][1]

[1]: https://www.baidu.com
这一部分可以放在任何地方

![图片alt](../assets/BingWallpaperx.jpg)

[![图片+链接](../assets/BingWallpaperx.jpg)](https://www.baidu.com)

\转义字符

| name   |  age  |  message |
| :--- | :---: | ---: |
| test |  18   | test-message |
| test another |  18   | test-message another |
| 冒号在左即左侧对齐 |  18   | 冒号在右即右侧对齐 |

``` json
{
  "语法高亮": ```
}
```

行内公式和行间公式：
$a^b$
$$a_b$$

{}用来表示一个整体：
$\sum_{n=1}^N {n+1}$

$\sum累加 \int积分 \prod连乘 \lim极限$
