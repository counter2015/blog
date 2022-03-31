

# Scalatra in Action 3: 你要去哪里？

title: Scalatra in Action 3. 你要去哪里？
date: 2019-03-23 16:06:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------


## 路由概览

之前我们有提到，Scalatra的核心，就是处理http请求，这里需要仔细弄明白请求是怎么被处理的。



![](https://counter2015.com/picture/scalatra-3-1.jpg)

▲ 图片来源：scalatra in action



Scalatra的路由 由三部分构成。

- *http method* , http只支持8种方法，如下表所示

![](https://counter2015.com/picture/scalatra-3-2.jpg)

- *route matcher*, 组成路径表达式的路由匹配灵活多变，下面会详细说明
- *action*,  该部分代码块表示执行到相应路由的情况下该执行的操作

<del>CRUD code boy</del>



## http method

http协议不关心数据是存储在关系型数据库的表中，还是存放在MongoDB的文件里，又或者是其他的数据存储实现里，我们在Scalatra的服务中谈论CRUD时，指的是*资源*。

**GET**

GET是最常见的http操作，当我们在地址栏键入网址时，按下回车，其实这时GET请求就已经被发送出去了

```scala
// fetching a artist with GET method
class RecordStore extends ScalatraServlet{
    get("/artists/:name/info"){
        Artist.find(params("name")) match {
            case Some(artist) => artist.toXml
            case None => status = 404
        }
    }
}
```

GET适用于以下几个场景

- 只读操作
- 请求可以重复提交，<del>比如我用F5一直刷新最新的新闻</del>
- 回应为标签或者搜索结果时



**POST**

WEB表单的默认操作是POST,但是POST能做的事不止这一个，POST可以处理json,xml,图片，文件等各种资源。

```scala
// adding a new artist with post
class RecordStore extends ScalatraServlet{
    post("/artist/new"){
        val artist = parseArtist(request)
        Artist.save(artist)
        val location = s"/artists/${artist.name}"
       
        //"Ctreated" will retrun http 201 code, and created with location header
        Created(artist, headers = "Location" -> location)
    }
    
    // read the artist from the request parameters 
    def parseArtist(implicit request: HttpServletRequest): Artist = {
        Artist(
            name = Artist.fromParam(params("name")),
            nationality = params("nationality"),
            isActive = params("isActive").toBoolean
        )
    }
}
```

POST的典型操作：解析请求的正文(body)，在服务器上创建一个新的资源并返回它的指针。

POST适用于以下几个场景：

- 创建操作
- 为创建的实体建立url
- 非幂等的写操作



**PUT**

PUT和POST最相似，PUT的正文中修改特定位置的资源

```scala
class RecordStore extends ScalatraServlet {
    put("/artists/:name"){
        val artist = parseArtist(request)
        Artist.update(parmas("name"), artist)
        //return the http code 204
        NoContent()
    }
    
    // read the artist from the request parameters
    def parseArtist(implicit request: HttpServletRequest): Artist = {
        Artist(
       		name = Artist.fromparam(params("name")),
            nationality = params("nationality"),
            isActive = params("	isActive").toBoolean
        )
    }
}
```



关于POST和PUT的区别，可以参考[这里](https://stackoverflow.com/questions/630453/put-vs-post-in-rest)

简单来说PUT执行的是幂等的操作。

PUT适用于以下几个场景：

- 更新操作
- 创建操作（当客户端完全了解资源的URL时）



**DELETE**

顾名思义，就是删除

```scala
class RecordStore extends ScalatraServlet {
    delete("/artists/:name"){
        if (Artists.delete(params("name")).isDefined){
            //if an artist was deleted, return http code 204
            NoContent()
        } else {
            // if no artist was found, return http code 404
        	NotFound()
        }
    }
}
```



以上是比较常见的http method.



## 匹配

之前的例子中，我们其实已经接触过了静态路径和参数变量

- get("/artists")
- get("/artists/:name")



在静态路径中，部分特殊字符需要经过[URL编码](http://en.wikipedia.org/wiki/Percent-encoding)，下面是一个例子。

![](https://counter2015.com/picture/scalatra-3-3.jpg)

在参数变量中，我们可以这样获取

```scala
/articles/52?foo=uno&bar=dos&baz=three&foo=anotherfoo

get("/articles/:id") {
  params("id") // => "52"
  params("foo") // => "uno" (discarding the second "foo" parameter value)
  params("unknown") // => generates a NoSuchElementException
  params.get("unknown") // => None - this is what Scala does with unknown keys in a Map

  multiParams("id") // => Seq("52")
  multiParams("foo") // => Seq("uno", "anotherfoo")
  multiParams("unknown") // => an empty Seq
}
```

参见[官方文档](http://scalatra.org//guides/2.6/http/routes.html)

### 可选参数

在匹配过程中，`?`代表在其之前的字符或者参数变量是*可选的*

```scala
get("/artists/?"){
   <artists>${Artist.fetchAll().map(_.toXml)}</artists>
}
```

上边的`/artists/?`会匹配以下两个请求

- http://localhost:8080/artists
- http://localhost:8080/artists/



```scala
class RecordStore extends ScalatraServlet {
    get("/artists/:name/info.?:format?") {
        Artist.find(params("name")) match {
            case Some(artist) => params.get("format") match {
				case Some("json") => artist.toJson
				case _ => artist.toXml
    			}
           	case None => NotFound()
        }
    }
}
```

以上这段代码可以匹配如下请求

| URL                                                  | Format param |
| ---------------------------------------------------- | ------------ |
| http://localhost:8080/artists/Otis_Redding/info      | undefined    |
| http://localhost:8080/artists/Otis_Redding/info.json | "json"       |
| http://localhost:8080/artists/Otis_Redding/info.xml  | "xml"        |
| http://localhost:8080/artists/Otis_Redding/info.     | undefined    |
| http://localhost:8080/artists/Otis_Redding/infojson  | "json"       |

### 通配符

通配符为"*",下面是一个简单的例子,其中`splat`是一个保留字，用来匹配通配符。

```scala
get("/say/*/to/*") {
  // Matches "GET /say/hello/to/world"
  multiParams("splat") // == Seq("hello", "world")
}

get("/download/*.*") {
  // Matches "GET /download/path/to/file.xml"
  multiParams("splat") // == Seq("path/to/file", "xml")
}
```

### 正则

如果你对正则有一定了解的话，那么看了之前的`?`和`*`,你一定会联想到*元字符*，没错，Scalatra当然支持正则匹配

> Some people, when confronted with a problem, think “I know, I’ll use regular
> expressions.” Now they have two problems.
> —Jamie Zawinski

<del>上次我看到这个笑话的时候说的是多线程</del>

```scala
get("""^/f(.*)/b(.*)""".r) {
  // Matches "GET /foo/bar"
  multiParams("captures") // == Seq("oo", "ar")
}
```

我们来详细讲解下`"""^/f(.*)/b(.*)""".r`这段和天书一样的代码

如果把它看成`"""<pattern>""".r`，`<pattern>`为`^\/f(.*)/b(.*)`

外层三引号加点r说明内部的表达式是正则格式，内部正则则是说明需要匹配的文本

`^`表示匹配 *开始*

这里有两个括号，是用来捕获的`(.*)`代表捕获任意字符(这里当然不包括`/`)

如果你想了解更多，推荐阅读[精通正则表达式](https://book.douban.com/subject/2154713/)

这里还有一个点要注意，`captures`是Scalatra的保留字，类似于`for`在各种语言中的地位，是有其特殊含义的。

mutiParams(”captures“)的作用就是获取所有被捕获的参数。

Scalatra将路径表达式编译成正则表达式，下面是部分映射关系表格

| 路径表达式 | 等效正则  |
| ---------- | --------- |
| :param     | ([/*#?]+) |
| ?          | ?         |
| *          | (.*?)     |
| ()         | \\(\\)     |



### 布尔表达式

和路径表达式、正则表达式不同，布尔表达式不能提取出URL中的某个字段，它的作用更像是一个门卫，判断哪些请求可以通过而哪些请求不可以通过。

```scala
class RecordStore extends ScalatraServlet {
	get("/listen/*".r, isMobile(request))) {
		StreamService.mobile(params("splat"))
	}
	
    get("/listen/*".r, !isMobile(request))) {
		StreamService.desktop(params("splat"))
	}
	
    def isMobile(request: HttpServletRequest): Boolean = {
		val lower = request.getHeader("User-Agent")
		lower.contains("android") || lower.contains("iphone")
	}
}
```



## 匹配顺序

自底向上,首次匹配,一图流

```scala
class MyScalatraServlet extends ScalatraServlet {
  get("/:name"){
    "name"
  }
  
  get("/:key"){
    "key"
  }
}
```

![](https://counter2015.com/picture/scalatra-3-4.jpg)

其他详细的使用，参考[官方文档](http://scalatra.org//guides/2.6/http/routes.html)

## 小结

了解路由匹配的基本规则