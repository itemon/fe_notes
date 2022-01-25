# PostCSS和它的那些"插件"们

现在前端工程体系变得越来越复杂，工程内部分工越来越细，看一眼配置，postcss, autoprefix，eslint, prettier等等等等，错综复杂交织在一起，你甚至开始怀念jQuery时代的简单。

工程问题易被忽视，尤其是初入行当，遇到问题往往要多花费很多精力去解决，但工程问题又是提效之源，花时间去系统理解它们是一件值得去投入的事情。其中PostCSS和它的那些插件们便是首要面对的。

## 什么是PostCSS

官方给出的PostCSS的定义是

> PostCSS is a tool for transforming style with JS plugins

说白了，PostCSS就是一个样式转换工具，它的作用是在代码运行前，将CSS代码做一次转换，只不过PostCSS使用的是JS语言来完成这个工作。为什么要对CSS代码做转换呢，其主要原因有几点：

1. 语法转换需要，你可能使用了最新的语法标准，但大部分浏览器还不能直接运行这些新语法的样式

2. 标准语法扩展，比如支持sass的嵌套语法，函数，变量等等。

3. 解决平台兼容性问题，你只需要使用最标准的语法，PostCSS帮你转换成每家浏览器支持的语法，比如flex布局。

归根结底，因为有了一次代码转换的过程，所以就有无限种可能性。

## 理清楚PostCSS的内外部关系

PostCSS扎根于前端的工程体系内，拿webpack来说，PostCSS就是以loader的形式存在并且在编译器应用转换工作的。

### PostCSS的外部关系

和PostCSS一样，Sass、Less也是应用较广的CSS预编译工具，他们都可以转换最新的CSS语法，支持函数，mixin等CSS语言扩展特性。不同的是，Sass和Less本身是比较系统的CSS扩展语言，内建了编译转换工具。PostCSS是通过插件的机制，分散提供语言特性的转换工具，甚至，开发者可以自定义插件，提供新的定制语法转化，灵活度更高(PostCSS还提供了转化Sass、Less语法的插件)。

### PostCSS的内部结构

相比而言，PostCSS最大的设计优势在于其内部的插件机制，为扩展性和定制性提供了可能性。插件机制本身很好理解，PostCSS本身作为loader工作在类似webpack的工程编译流程中，PostCSS解析CSS文件生成AST Tree，其插件流水线处理AST Tree，很多前端工程工具都有类似的设计（如babel)。

<img src="file:///Users/huangwei/Documents/postcssdrawio.png" title="" alt="postcssdrawio.png" data-align="center">
