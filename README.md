# Dfinity_Chinese_Guidelines

## Dfinity中文指引

- Motoko中文开发指引
- Inside the Internet Computer中文研究报告
- ……

## 获取最新文章请移步我的博客：[秋夜Dx の ブログ](https://qiuyedx.com/)



WP HTML转Markdown通配符：

^<!--.*



代码块转换：

先把```匹配到的全部换成三个点加go，

然后把```go\n\n匹配到的换成三个点即可。


还得匹配一下****，全部删除;匹配一下 span>\n\n\星\星 ，选择性地or全部换为 span>** 。
(如有发现其他格式异常，请提交issue告诉我，感谢！)

最后的最后，匹配^\n$，全部删除，即可删除连续的多余的空行。