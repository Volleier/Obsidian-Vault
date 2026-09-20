Markdown基本语法允许您通过将行缩进四个空格或一个制表符来创建代码块。如果发现不方便，请尝试使用受保护的代码块。根据Markdown处理器或编辑器的不同，您将在代码块之前和之后的行上使用三个反引号（(` ``` `）或三个波浪号（\~\~\~)

````
```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````

呈现的输出如下所示：

```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

# 语法高亮

许多Markdown处理器都支持受围栏代码块的语法突出显示。使用此功能，您可以为编写代码的任何语言添加颜色突出显示。要添加语法突出显示，请在受防护的代码块之前的反引号旁边指定一种语言。

````
```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```
````

呈现的输出如下所示：
```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

**常用程序标识关键字**

| 语言           | 关键字                       | 语言               | 关键字                     |
| ------------ | ------------------------- | ---------------- | ----------------------- |
| C            | cpp , c                   | GO               | go , golang             |
| Java         | java                      | AppleScript      | applescript             |
| Python       | py , python               | ActionScript 3.0 | actionscript3 , as3     |
| PHP          | php                       | ColdFusion       | coldfusion , cf         |
| Shell        | bash , shell              | Delphi           | delphi , pascal , pas   |
| C#           | c# , c-sharp , csharp     | diff&patch       | diff patch              |
| CSS          | css                       | Erlang           | erl , erlang            |
| JavaScript   | js , jscript , javascript | Groovy           | groovy                  |
| text         | text , plain              | JavaFX           | jfx , javafx            |
| XML          | xml , xhtml , xslt , html | Perl             | perl , pl , Perl        |
| R            | r , s , splus             | Ruby             | ruby , rails , ror , rb |
| Visual Basic | vb , vbnet                | SASS&SCSS        | sass , scss             |
| Objective C  | objc , obj-c              | Scala            | scala                   |
| F#           | f# f-sharp , fsharp       | SQL              | sql                     |
| matlab       | matlab                    | swift            | swift                   |


