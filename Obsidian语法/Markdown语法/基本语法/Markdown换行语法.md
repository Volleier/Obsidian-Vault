在一行的末尾添加两个或多个空格，然后按回车键,即可创建一个换行(`<br>`)。

|Markdown语法|HTML|预览效果|
|---|---|---|
|`This is the first line.     And this is the second line.`|`<p>This is the first line.<br>   And this is the second line.</p>`|This is the first line.  <br>And this is the second line.|

# 换行（Line Break）用法的最佳实践
为了兼容性，请在行尾添加“结尾空格”或 HTML 的 `<br>` 标签来实现换行。

还有两种其他方式我并不推荐使用。CommonMark 和其它几种轻量级标记语言支持在行尾添加反斜杠 (`\`) 的方式实现换行，但是并非所有 Markdown 应用程序都支持此种方式，因此从兼容性的角度来看，不推荐使用。并且至少有两种轻量级标记语言支持无须在行尾添加任何内容，只须键入回车键（`return`）即可实现换行。

|✅  Do this|❌  Don't do this|
|---|---|
|`First line with two spaces after.  `<br>`And the next line.`<br><br>`First line with the HTML tag after. <br> And the next line.`|`First line with a backslash after.\   And the next line.`<br><br>`First line with nothing after.   And the next line.`|