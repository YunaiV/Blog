title: Web API 文档生成工具 apidoc
date: 2018-08-01
tags:
categories: 精进
permalink: Fight/web-api-doc
author: 老梁
from_url: http://blog.720ui.com/2018/apidoc_use/
wechat_url:

-------

摘要: 原创出处 http://blog.720ui.com/2018/apidoc_use/ 「老梁」欢迎转载，保留摘要，谢谢！

- [开始入门](http://www.iocoder.cn/Fight/web-api-doc/)
- [代码注释](http://www.iocoder.cn/Fight/web-api-doc/)
- [完整的案例](http://www.iocoder.cn/Fight/web-api-doc/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

<!-- more -->

在服务端开发过程中，我们需要提供一份 API 接口文档给 Web 端和移动端使用。实现 API 接口文档编写工作，有很多种方式，例如通过 Word 文档编写，或者通过 MediaWiki 进行维护。此外，还有比较流行的方式是利用 Swagger 自动化生成文档。这里，笔者想分享另一个 Web API 文档生成工具 apidoc。

apidoc 是通过源码中的注释来生成 Web API 文档。因此，apidoc 对现有代码可以做到无侵入性。此外，apidoc 可以支持多种语言 C#, Go, Dart, Java, JavaScript, PHP, TypeScript (all DocStyle capable languages)，CoffeeScript，Erlang，Perl，Python，Ruby，Lua。通过 apidoc 可以非常方便地生成可交互地文档页面。

![](image.chenyongjun.vip/apidoc01.png)

## 开始入门

首先，我们需要 node.js 的支持。在搭建好 node.js 环境后，通过终端输入 npm 命名进行安装。

```Shell
npm install apidoc -g
```

接着，我们还需要添加 apidoc.json 文件到项目工程的根目录下。

```JSON
{
  "name": "example",
  "version": "0.1.0",
  "description": "apiDoc basic example",
  "title": "Custom apiDoc browser title",
  "url" : "https://api.github.com/v1"
}
```

这里，笔者主要演示 Java 注释如何和 apidoc 结合使用。现在，我们先来看一个案例。

```Java
/**
 *   @api {GET} logistics/policys 查询签收预警策略
 *   @apiDescription 查询签收预警策略
 *   @apiGroup QUERY
 *   @apiName logistics/policys
 *   @apiParam  {Integer} edition   平台类型
 *   @apiParam  {String} tenantCode 商家名称
 *   @apiPermission LOGISTICS_POCILY
 */
```

最后，我们在终端输入 apidoc 命令进行文档生成。这里，我们用自己的项目工程的根目录替代 myapp/,用需要生成文档的地址替代 apidoc/。

```Shell
apidoc -i myapp/ -o apidoc/
```

例如，笔者的配置是这样的。

```Shell
apidoc -i /Users/lianggzone/Documents/dev-space/git-repo -o /Users/lianggzone/Documents/dev-space/apidoc/
```

## 代码注释

### @api
@api 标签是必填的，只有使用 @api 标签的注释块才会被解析生成文档内容。格式如下：

```Shell
@api {method} path [title]
```

这里，有必要对参数内容进行讲解。

| 参数名 | 描述 |
| :--- | :--- |
| method | 请求方法, 如 POST，GET，POST，PUT，DELETE 等。 |
| path | 请求路径。 |
| title【选填】 | 简单的描述 |

### @apiDescription

@apiDescription 对 API 接口进行描述。格式如下：

<pre>
@apiDescription text
</pre>

### @apiGroup

@apiGroup 表示分组名称，它会被解析成一级导航栏菜单。格式如下：

<pre>
@apiGroup name
</pre>

### @apiName

@apiName 表示接口名称。注意的是，在同一个 @apiGroup 下，名称相同的 @api 通过 @apiVersion 区分，否者后面 @api 会覆盖前面定义的 @api。格式如下：

```Shell
@apiName name
```

### @apiVersion

@apiVersion 表示接口的版本，和 @apiName 一起使用。格式如下：

```Shell
@apiVersion version
```

### @apiParam
@apiParam 定义 API 接口需要的请求参数。格式如下：

```Shell
@apiParam [(group)] [{type}] [field=defaultValue] [description]
```

这里，有必要对参数内容进行讲解。

| 参数名 | 描述 |
| :--- | :--- |
| (group)【选填】 | 参数进行分组 |
| {type}【选填】 | 参数类型，包括{Boolean}, {Number}, {String}, {Object}, {String[]}， (array of strings), ... |
| {type{size}}【选填】 | 可以声明参数范围，例如{string{..5}}， {string{2..5}}， {number{100-999}} |
| {type=allowedValues}【选填】 | 可以声明参数允许的枚举值，例如{string=&quot;small&quot;,&quot;huge&quot;}， {number=1,2,3,99} |
| field | 参数名称 |
| [field] | 声明该参数可选 |
| =defaultValue【选填】 | 声明该参数默认值 |
| description【选填】 | 声明该参数描述 |

类似的用法，还有 @apiHeader 定义 API 接口需要的请求头，@apiSuccess 定义 API 接口需要的响应成功，@apiError 定义了 API 接口需要的响应错误。

这里，我们看一个案例。

```Java
/**
 *   @apiParam  {Integer} edition=1   平台类型
 *   @apiParam  {String} [tenantCode] 商家名称
 */
```

此外，还有 @apiParamExample，@apiHeaderExample， @apiErrorExample，@apiSuccessExample 可以用来在文档中提供相关示例。

### @apiPermission
@apiPermission 定义 API 接口需要的权限点。格式如下：

```Shell
@apiPermission name
```

此外，还有一些特别的注解。例如 @apiDeprecated 表示这个 API 接口已经废弃，@apiIgnore 表示忽略这个接口的解析。关于更多的使用细节，可以阅读官方文档：http://apidocjs.com/#demo

## 完整的案例

最后，我们用官方的案例，讲解下一个完整的配置。

首先，配置 apidoc.json，内容如下：

```Json
{
  "name": "example",
  "version": "0.1.0",
  "description": "A basic apiDoc example"
}
```

接着，我们配置相关的 Java 源码注释。

```Java
/**
 * @api {get} /user/:id Request User information
 * @apiName GetUser
 * @apiGroup User
 *
 * @apiParam {Number} id Users unique ID.
 *
 * @apiSuccess {String} firstname Firstname of the User.
 * @apiSuccess {String} lastname  Lastname of the User.
 *
 * @apiSuccessExample Success-Response:
 *     HTTP/1.1 200 OK
 *     {
 *       "firstname": "John",
 *       "lastname": "Doe"
 *     }
 *
 * @apiError UserNotFound The id of the User was not found.
 *
 * @apiErrorExample Error-Response:
 *     HTTP/1.1 404 Not Found
 *     {
 *       "error": "UserNotFound"
 *     }
 */
```

然后，执行命名生成文档。

```Shell
apidoc -i myapp/ -o apidoc/
```

生成的页面，如下所示。

![](image.chenyongjun.vip/apidoc02.jpeg)