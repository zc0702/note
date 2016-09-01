
### 使用简洁明了的语法进行排版

---

### 标题

在 Markdown 文件的文字前面添加“#”号。

    # 一级标题
    ## 二级标题
    ### 三级标题
    #### 四级标题
    ##### 五级标题
    ###### 六级标题

效果

# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题

---

### 列表

无序列表：在 Markdown 文件的文字前面添加“-”或者“*”号。

有序列表：在 Markdown 文件的文字前面添加“1. 2. 3. ”这样的数字序号。

符号或者数字，与正文之间使用空格分割。

    - aaa
    - bbb
    - ccc

效果

- aaa
- bbb
- ccc

---

### 引用块

在文本前增加“>”

    > aaa

效果

>aaa

---

### 粗体

使用两个 * 包含文本

    **test**

效果

**test**

---

### 斜体

使用一个 * 包含文本

    *test*

效果

*test*

---

### 代码块

代码前放入 4 个空格或是 1 个制表符

        int main()
        {
            print 1
        }

效果

    int main()
    {
        print 1
    }

---

### 表格

这个表格用起来真的很崩溃 ~.~

举个栗子

    | title1        | title2   | title3  |
    | ------------- |:--------:| -----:|
    | 1-1           | 2-1      | 3-1 |
    | 1-2           | 2-2      | 3-2 |
    | 1-3           | 2-3      | 3-3 |

效果

| title1        | title2   | title3  |
| ------------- |:--------:| -----:|
| 1-1           | 2-1      | 3-1 |
| 1-2           | 2-2      | 3-2 |
| 1-3           | 2-3      | 3-3 |

个人感觉还不如直接使用html的table标签方便

    <table>
        <tr>
            <th>title1</th>
            <th>title2</th>
            <th>title3</th>
        </tr>
        <tr>
            <td>1-1</td>
            <td>2-1</td>
            <td>3-1</td>
        </tr>
        <tr>
            <td>1-2</td>
            <td>2-2</td>
            <td>3-2</td>
        </tr>
        <tr>
            <td>1-3</td>
            <td>2-3</td>
            <td>3-3</td>
        </tr>
    </table>

效果

<table>
    <tr>
        <th>title1</th>
        <th>title2</th>
        <th>title3</th>
    </tr>
    <tr>
        <td>1-1</td>
        <td>2-1</td>
        <td>3-1</td>
    </tr>
    <tr>
        <td>1-2</td>
        <td>2-2</td>
        <td>3-2</td>
    </tr>
    <tr>
        <td>1-3</td>
        <td>2-3</td>
        <td>3-3</td>
    </tr>
</table>

---

### 图片

图片格式：\!\[Alt text\]\(/path/to/img.jpg "Optional title"\)

    ![图片加载不出来的替换文字](http://imgsrc.baidu.com/forum/w%3D580/sign=c88ee8d20fd79123e0e0947c9d355917/ad651930e924b899ab0fc1e26d061d950b7bf692.jpg "葫芦娃")

效果

![图片加载不出来的替换文字](http://imgsrc.baidu.com/forum/w%3D580/sign=c88ee8d20fd79123e0e0947c9d355917/ad651930e924b899ab0fc1e26d061d950b7bf692.jpg "葫芦娃")

---

### 链接

链接格式：\[\]\(\)
注：比图片格式少一个!号

    [百度](http://www.baidu.com)

效果

[百度](http://www.baidu.com)

---

