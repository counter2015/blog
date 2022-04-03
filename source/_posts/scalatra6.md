

# Scalatra in Action 6: 处理文件

title: Scalatra in Action 6. 处理文件
date: 2019-06-30 10:47:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------





## 提供文件

当文件服务器搭建完成后，预计是一个这样的界面。

![](http://counter2015.com/picture/scalatra-6-0.png)

###  搭建文件服务器

将文件写入响应时，需要先在头部声明正文中包含的文件类型，包括但不限于

- text/plain 文本文件
- image/jpeg JPEG图像
- application/octet-stream 二进制流

当未显示地声明文件类型时，会分析部分文件数据来推断文件的类型。

下面是一个文件服务应用程序的代码示例，它应该有用于访问存储位置进行读写的 WEB API。文件中需要包含文件的元信息(meta information)

文件的ID,名称，描述等内容。文件的存储需要能支持创建，查找，查看文件,以下是原书的代码。


```scala
case class Document (
  id: Long,
  name: String,
  contentType: Option[String],
  description: String)

case class DocumentStore(base: String) {
  private val fileNameIndex = collection.concurrent.TrieMap[Long, Document]()
  
  private val idCounter = new AtomicLong(0) 
  
  // add a new document
  def add(name: String,
          in: InputStream,
          contentType: Option[String],
          description: String): Long = {
    copyStream(in, new FileOutputStream(getFile(id)))
    val id = idCounter.getAndIncrement
    fileNameIndex(id) = Document(id, name, contentType, description)
    id
  }
  
  // return a sequence of all documents
  def list: Seq[Document] = fileNameIndex.values.toSeq
  
  // return a document for a given ID
  def findById(id: Long): Option[Document] = fileNameIndex.get(id)
  
  // return a file for a given ID
  def asFile(id: Long): File = new File(f"$base/$id")
  
  // write a input stream to an output stream
  private def copyStream(input: InputStream, output: OutputStream) {
    val buffer = Array.ofDim[Byte](1024)
    var bytesRead: Int = 0
    while (bytesRead != -1){
      bytesRead = input.read(buffer)
      if (bytesRead > 0) output.write(buffer, 0, bytesRead)
    }
  }
}
```

这里有两个可选的响应头字段(response header fields)， 

- `Content-Disposition`
- `Content-Description`

`Content-Disposition`中包含有关处理响应中包含的文件的信息，如果该字段设置为`inline`,那么返回的文件应该直接展示给用户

该字段的默认值为`attachment`,这会要求用户选择如何对文件做进一步的操作，如，浏览器通常会提示是直接打开还是另存为本地文件。

此外，在这个字段中还能设置文件名参数，用来设置存储文件时的默认文件名称。

`Content-Description`字段可以包含请求负载(payload)的简短描述。



下面的代码展示了如何从文档存储区提供文档服务，以及如何通过设置HTTP头来包含元信息。可以按文档ID来查询文件。

如果可以找到，则返回HTTP Header 集合	(headers set)；否则返回404错误。



``` scala
class DocumentsApp(store: DocumentStore) 
  extends ScalatraServlet with FileUploadSupport with ScalateSupport {
  
    get("/documentes/:documentId") {
    	val id = params.as[Long]("documentId")
      
      val doc = store.getDocument(id).getOrElse(halt(404, reason = "could not find document"))
      
      doc.contentType.foreach{ct => contentType = ct}
      response.setHeader("Content-Disposition", f"""attachment; filename="${doc.name}"""")
      
     	response.setHeader("Content-Description", 	doc.description)
     	store.getFile(id)
    }
}
```

接下来我们深入文件存储的部分，`DocumentStore`在`ScalatraBootstrap`这个类中，当程序启动时创建。



```scala
import org.scalatra._
import javax.servlet.ServletContext
import org.scalatra.book.chapter05.{DocumentStore, DocumentApp}

class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    val store = DocumentStore("data")
    store.add("strategy.jpg",
              new FileInputStream(new File("data/strategy")),
              Some("image/jpeg"),
              "bulletproof business strategy")
    
    store.add("manual.pdf",
              new FileInputStream(new File("data/manual.pdf")),
              Some("application/pdf"),
              "the manual about foos")
    
    val app = new DocumentsApp(store)
    context.mount(app, "/*")
  }
}


```

此时如果启动程序，从浏览器URL

http://localhost:8080/documents/0,就能成功下载文件了

###  静态资源

网络应用程序通常会包含各种网络资源，如：图像，CSS,HTML,通常它们都被放在`src/main/webapp`的资源目录下，

这些资源可以用`serveStaticResource`的方法提供给用户使用。

`serveStaticResource`方法从请求URL解析资源，如果可以找到资源，则会把它写入响应，这个资源可以通过

`ServletContext.getResource`方法来使用。如果找不到对应的资源，将会返回404错误。

这部分由`notFound`的handle来处理

```scala
notFound {
    contentType = null
    serveStaticResource() getOrElse halt(404, <h1>Not found.</h1>)
}
```



###  使用gzip压缩

HTTP允许对响应体应用内容编码。实际上，这个通常是一种压缩算法，可以减少网站使用的带宽。

scalatra提供了使用gzip算法对传出响应进行编码的选项。

使用gzip压缩特性需要启用`ContentEncodingSupport`特性：

```scala
class DocumentsApp extends ScalatraServlet with ContentEncodingSupprot {
  get ("/sample") {
    new File("data/strategy.jpg")
  }
}
```



## 接受文件

### 支持文件上传

如果想支持文件上传，需要在应用中启用`FileUploadSupport`的特质，当该特质启用时，如果发现

有`multipart request`，将提取报文部分给应用程序。

```scala
import org.scalatra.ScalatraServlet
import org.scalatra.servlet.FileUploadSupport
import org.scalatra.scalate.ScalateSupport

class DocumentsApp extends ScalatraServlet
	with FileUploadSupport with ScalateSupport {
    
}
```

每个文件都表示为fileitem的一个实例,它描述文件的

多部分请求中的名称、大小和原始字段名。如果在请求中指定了，内容类型和字符集是可用的；

否则它们为None值。表6.1列出了文件项字段和方法。

![](https://counter2015.com/picture/scalatra-6-1.png)

▲ 图片来源：scalatra in action



下面是一段简单的上传文件代码的示例

```scala
post("/sample") {
    val file = fileParams("sample")
    val desc = params("description")
    <div>
      <h1>Received {file.getSize} bytes</h1>
      <p>Description: {desc}</p>
    </div>
}
```



接下来来试着把原来成程序添加上上传文件的功能，下面是表单的模板程序，这是用`scalate`的DSL模板语言写的，看起来有点陌生

```scala
  .col-lg-3
    %h4 Create a new document
    %form(enctype="multipart/form-data" method="post" action="/documents")
      %div.form-group
        %label File:
        %input(type="file" name="file")
      %div.form-group
        %label Description:
        %input.form-control(type="text" name="description" value="")
      %input.btn.btn-default(type="submit")
```



翻译成HTML

```html
<div class="col-lg-3">
  <h4>Create a new document</h4>
  <form enctype="multipart/form-data" method="post" action="/documents">
    <div class="form-group">
      <label>File:</label>
      <input type="file" name="file">
    </div>
    <div class="form-group">
      <label>Description:</label>
      <input class="form-control" type="text" name="description" value="">
    </div>
    <input class="btn btn-default" type="submit">
  </form>
</div>
```



### 为文件上传添加配置

如上文所说，这里构建的文件服务器是基于内存的，所以要考虑到这样一个情况：用户上传了一个超大的文件把内存挤爆了。

因此，我们需要人工设定一个最大值来限制文件大小。

下面的代码中，限制了单个文件的大小不能超过30MB，每批文件的大小不能超过100MB，

可以通过`configureMultipartHandling`方法来配置

```scala
import org.scalatra.ScalatraServlet
import org.scalatra.servlet.FileUploadSupport
import org.scalatra.servlet.MultipartConfig
import org.scalatra.scalate.ScalateSupport

class DocumentsApp extends ScalatraServlet with FileUploadSupport with ScalateSupport {
	configureMultipartHandling(MultipartConfig(
	maxFileSize = Some(30 * 1024 * 1024),
	maxRequestSize = Some(100 * 1024 * 1024),
  ))
// ...
}
```

关于其他可以配置的信息可以参照下表，其中`-1`代表没有限制

![](<http://counter2015.com/picture/scalatra-6-2.png>)

完整的代码可以从[这里](<https://github.com/counter2015/scalatra-file-server>)访问







