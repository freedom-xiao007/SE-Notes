# Markdown转HTML
***
```python
import markdown
extensions = [ #根据不同教程加上的扩展
    'markdown.extensions.extra',
    'markdown.extensions.codehilite', #代码高亮扩展
    'markdown.extensions.toc',
    'markdown.extensions.tables',
    'markdown.extensions.fenced_code',
]
if __name__=="__main__":
    str_data = 你的markdowm文本 (这里加入markdown文章里不好显示就没写了，自己找一段就行)
    print(markdown.markdown(str_data,extensions=extensions))
```

## 参考链接
- [python使用markdown库把markdown转html时关于代码块高亮时遇到的大坑](https://blog.csdn.net/wjlhanhan/article/details/111404676)
- [用Python-Markdown和google-prettify来处理Markdown和代码高亮](https://www.the5fire.com/python-markdown-code-prettify.html)