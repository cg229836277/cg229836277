---
layout: post
title:  "html 九-从对象到 iframe — 其他嵌入技术（非原创）"
date:   2021-2-20 11:09:22 +0800
categories: html
---

> 文章来源于[https://developer.mozilla.org/](https://developer.mozilla.org/)

## 1、嵌入简史

很久以前，在Web上，使用框架创建网站非常流行-网站的一小部分存储在单独的HTML页面中。这些被嵌入到称为框架集的主文档中，该文档使您可以指定屏幕上每个框架填充的区域，就像调整表的列和行的大小一样。这些被认为是90年代中期至后期最酷的，并且有证据表明，将网页分成这样的较小块对于提高下载速度更好-尤其是在那时网络连接如此缓慢的情况下，这种情况尤为明显。但是，它们确实存在许多问题，随着网络速度的加快，这些问题远远超过了任何积极的方面，因此您不会再看到它们的使用。

不久之后（90年代末，2000年代初），插件技术变得非常流行，例如Java Applets和Flash-这些使Web开发人员能够将丰富的内容嵌入到网页中，例如视频和动画，而这些内容仅靠HTML不能提供。 。嵌入这些技术是通过\<object>和较少使用的\<embed>之类的元素实现的，在当时它们非常有用。此后，由于许多问题（包括可访问性，安全性，文件大小等），它们已经过时了。如今，大多数移动设备不再支持此类插件，并且台式机支持正在逐渐消失。

最后，出现了\<iframe>元素（以及其他嵌入内容的方法，例如\<canvas>，\<video>等）。这提供了一种将整个Web文档嵌入到另一个内部的方法，就好像它是< img>或其他此类元素，并且今天经常使用。

简单介绍完了之后，让我们继续看看如何使用他们。

## 2、详细了解iframe

\<iframe>元素旨在允许您将其他Web文档嵌入到当前文档中。这非常适合将第三方内容合并到您的网站中，而您可能无法直接控制这些内容，并且不想实现自己的版本-例如来自在线视频提供商的视频，诸如Disqus之类的评论系统，来自在线的地图地图提供商，广告横幅等。您在本课程中一直使用的实时可编辑示例是使用\<iframe>实现的。

正如我们将在下面讨论的那样，\<iframe>存在一些严重的安全问题，但这并不意味着您不应该在您的网站中使用它们-它只需要一些知识和仔细的考虑即可。让我们更详细地研究代码。假设您想在您的一个网页中包含MDN词汇表-您可以尝试执行以下操作：

```html
<iframe src="https://developer.mozilla.org/en-US/docs/Glossary"
        width="100%" height="500" frameborder="0"
        allowfullscreen sandbox>
  <p>
    <a href="/en-US/docs/Glossary">
       Fallback link for browsers that don't support iframes
    </a>
  </p>
</iframe>
```

此示例包括使用\<iframe>所需的基本要素：

**allowfullscreen**

如果设置了\<iframe>，则可以使用Fullscreen API（在本文范围之外）以全屏模式放置。

**frameborder**

如果设置为1，则告诉浏览器在此框架和其他框架之间绘制边框，这是默认行为。

0表示删除边框，但不再建议使用此功能，因为在您的CSS中使用border可以更好地实现相同的效果。

**src**

与\<video> / \<img>一样，此属性包含指向要嵌入文档的URL的路径。

**width** 和 **height**

这些属性指定您希望iframe成为的宽度和高度。

**后备内容**

与\<video>之类的其他类似元素一样，您可以在\<iframe> \</iframe>标记之间添加回退内容，如果浏览器不支持\<iframe>，则会显示这些内容。在这种情况下，我们包含了指向页面的链接。如今，您不太可能会遇到任何不支持\<iframe>的浏览器。

**sandbox**

与其他\<iframe>功能（例如IE 10及更高版本）相比，此属性在更现代的浏览器中工作，要求提高安全性设置；我们将在下一部分中对此进行详细说明。

### 2.1 安全问题

上面我们提到了安全问题-现在让我们更详细地讨论这一点。我们不希望您第一次就完全理解所有这些内容。我们只是想让您意识到这一问题，并提供一个参考，以便您在有经验时可以重新考虑并开始考虑在实验和工作中使用\<iframe>。另外，也不必害怕并且不使用\<iframe>-您只需要小心。继续阅读...

浏览器制造商和网络开发人员已经了解到，很难将iframe用作网络上的恶意用户（通常称为黑客，或更准确地说是黑客）进行攻击的常见目标（官方术语：攻击向量），如果他们试图恶意修改您的网页，或诱使人们做自己不想做的事情，例如泄露用户名和密码之类的敏感信息。因此，规范工程师和浏览器开发人员已经开发了各种安全机制来提高\<iframe>的安全性，并且还需要考虑一些最佳做法-我们将在下面介绍其中的一些。

> 点击劫持是一种常见的iframe攻击，黑客将不可见的iframe嵌入到您的文档中（或将您的文档嵌入到自己的恶意网站中），并使用它来捕获用户的互动。这是一种误导用户或窃取敏感数据的常用方法。

不过，首先提供一个快速示例-尝试将上面显示的上一个示例加载到浏览器中-您可以在GitHub上实时找到它（也请参见源代码。）您实际上不会在页面上看到任何显示，并且如果您查看在浏览器开发人员工具中的控制台上，您会看到一条消息，说明原因。在Firefox中，您会被告知X-Frame-Options拒绝了Load：<https://developer.mozilla.org/en-US/docs/Glossary>不允许框架。这是因为构建MDN的开发人员已在服务器上提供了一个设置，该设置可为网站页面提供服务，以防止将其嵌入\<iframe>内（请参阅下面的Configure CSP指令。）这很有意义-整个MDN页面都没有。除非要进行诸如将其嵌入到网站中并声称自己拥有所有权之类的操作，或者尝试通过单击劫持来窃取数据，这都是非常不好的事情，否则将其嵌入其他页面中确实没有任何意义。另外，如果每个人都开始这样做，那么所有额外的带宽将开始使Mozilla花费很多钱。

## 3、仅在必要时嵌入

有时嵌入第三方内容（例如youtube视频和地图）是很有意义的，但是如果仅在完全必要时才嵌入第三方内容，则可以避免很多麻烦。 Web安全的一个好的经验法则是：“永远不要太谨慎。如果确实做到了，则无论如何都要对其进行仔细检查。如果有人做到了，除非有其他证明，否则就认为它很危险。”

除了安全性，您还应该注意知识产权问题。大多数内容都是受版权保护的，离线和在线的内容，甚至是您可能不希望看到的内容（例如，Wikimedia Commons上的大多数图像）。除非您拥有网页内容或拥有者已给予您明确的书面许可，否则切勿在网页上显示内容。侵犯版权的处罚是严厉的。同样，您永远不能太谨慎。

如果内容已获得许可，则必须遵守许可条款。例如，MDN上的内容是根据CC-BY-SA许可的。这意味着，即使您进行了重大更改，您在引用我们的内容时也必须正确地相信我们。

## 4、使用 HTTPS

HTTPS是HTTP的加密版本。您应该尽可能使用HTTPS服务您的网站：

- 1. HTTPS减少了远程内容在传输过程中被篡改的机会，
- 2. HTTPS阻止嵌入的内容访问父文档中的内容，反之亦然。

使用HTTPS需要一个安全证书，该证书可能很昂贵（尽管“让我们加密”使事情变得容易）–如果无法获得证书，则可以使用HTTP为父文档提供服务。但是，由于上述HTTPS的第二个好处，无论花费多少，您都绝不能在HTTP中嵌入第三方内容。 （在最佳情况下，用户的Web浏览器会给他们发出可怕的警告。）所有通过\<iframe>嵌入内容的知名公司都将通过HTTPS使其可用-查看\<iframe> src属性内的URL，当您嵌入Google Maps或YouTube中的内容。

## 5、一律使用sandbox属性

您想给攻击者尽可能少的权力，使他们可以在网站上做坏事，因此，您应该只给嵌入式内容执行其工作所需的权限。当然，这也适用于您自己的内容。可以适当使用（或用于测试）但不会对其余代码库（意外或恶意）造成任何损害的代码容器称为沙盒。

未沙盒化的内容可能会做太多事情（执行JavaScript，提交表单，弹出窗口等）。默认情况下，您应该使用不带参数的sandbox属性来施加所有可用的限制，如前面的示例所示。

如果绝对需要，您可以一个接一个地添加权限（在sandbox =“”属性值内）—有关所有可用选项，请参见[sandbox](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox)参考条目。一个重要的注意事项是，您绝对不要在您的沙盒属性中同时添加allow-scripts和allow-same-origin-在这种情况下，嵌入的内容可能会绕过Same-origin策略，该策略会阻止网站执行脚本，并使用JavaScript完全关闭沙箱。

## 6、配置CSP指令

CSP是内容安全策略的缩写（content security policy），它提供了一组HTTP标头（从Web服务器提供服务时随网页发送的元数据），旨在提高HTML文档的安全性。在保护\<iframe>时，可以配置服务器以发送适当的X-Frame-Options标头。这可以防止其他网站将您的内容嵌入其网页中（这将启用点击劫持和其他攻击），正如我们前面所看到的，这正是MDN开发人员所做的。

## 7、<embed> 和 <object> 元素

\<embed>和\<object>元素与\<iframe>的功能不同-这些元素是用于嵌入多种外部内容的通用嵌入工具，其中包括Java Applets和Flash，PDF等插件技术（可在一个带有PDF插件的浏览器），甚至包括视频，SVG和图像之类的内容！

但是，您不太可能经常使用这些元素-Applets已经使用多年了，由于多种原因，Flash不再非常流行（请参阅下面的插件案例），PDF往往可以更好地链接而不是嵌入式，其他内容（例如图像和视频）具有更好，更容易处理的元素。插件和这些嵌入方法实际上是一种旧技术，我们主要在提及它们的情况下，以防您在某些情况下（例如Intranet或企业项目）遇到它们。

如果您发现自己需要嵌入插件内容，则至少需要以下信息：

<table class="standard-table">
 <thead>
  <tr>
   <th scope="col"></th>
   <th scope="col"><a href="/en-US/docs/Web/HTML/Element/embed"><code>&lt;embed&gt;</code></a></th>
   <th scope="col"><a href="/en-US/docs/Web/HTML/Element/object"><code>&lt;object&gt;</code></a></th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td><a href="/en-US/docs/Glossary/URL">URL</a> of the embedded content</td>
   <td><a href="/en-US/docs/Web/HTML/Element/embed#attr-src"><code>src</code></a></td>
   <td><a href="/en-US/docs/Web/HTML/Element/object#attr-data"><code>data</code></a></td>
  </tr>
  <tr>
   <td><em>accurate </em><a href="/en-US/docs/Glossary/MIME_type">media type</a> of the embedded content</td>
   <td><a href="/en-US/docs/Web/HTML/Element/embed#attr-type"><code>type</code></a></td>
   <td><a href="/en-US/docs/Web/HTML/Element/object#attr-type"><code>type</code></a></td>
  </tr>
  <tr>
   <td>height and width (in CSS pixels) of the box controlled by the plugin</td>
   <td><a href="/en-US/docs/Web/HTML/Element/embed#attr-height"><code>height</code></a><br>
    <a href="/en-US/docs/Web/HTML/Element/embed#attr-width"><code>width</code></a></td>
   <td><a href="/en-US/docs/Web/HTML/Element/object#attr-height"><code>height</code></a><br>
    <a href="/en-US/docs/Web/HTML/Element/object#attr-width"><code>width</code></a></td>
  </tr>
  <tr>
   <td>names and values, to feed the plugin as parameters</td>
   <td>ad hoc attributes with those names and values</td>
   <td>single-tag <a href="/en-US/docs/Web/HTML/Element/param"><code>&lt;param&gt;</code></a> elements, contained within <code>&lt;object&gt;</code></td>
  </tr>
  <tr>
   <td>independent HTML content as fallback for an unavailable resource</td>
   <td>not supported (<code>&lt;noembed&gt;</code> is obsolete)</td>
   <td>contained within <code>&lt;object&gt;</code>, after <code>&lt;param&gt;</code> elements</td>
  </tr>
 </tbody>
</table>

这是一个使用<embed>元素嵌入Flash电影的示例：

```html
<embed src="whoosh.swf" quality="medium"
       bgcolor="#ffffff" width="550" height="400"
       name="whoosh" align="middle" allowScriptAccess="sameDomain"
       allowFullScreen="false" type="application/x-shockwave-flash"
       pluginspage="http://www.macromedia.com/go/getflashplayer">
```

太可怕了，不是吗？ Adobe Flash工具生成的HTML甚至更糟，使用嵌入了\<embed>元素的\<object>元素嵌入到里面。Flash曾经成功的为HTML5视频配置了后备内容，但有一段时间之后，这变得越来越没有必要了。

现在，让我们看一个将PDF嵌入页面的\<object>示例:

```html
<object data="mypdf.pdf" type="application/pdf"
        width="800" height="1200" typemustmatch>
  <p>You don't have a PDF plugin, but you can
    <a href="mypdf.pdf">download the PDF file.
    </a>
  </p>
</object>
```

PDF是纸张和数字化之间必不可少的垫脚石，但它们带来了许多可访问性挑战，并且在小屏幕上很难阅读。它们确实在某些圈子中仍然很受欢迎，但是链接到它们更好，这样它们可以在单独的页面上下载或阅读，而不是将其嵌入网页中。

### 7.1 避免插件提示

从前，插件在网络上必不可少。还记得曾经为了观看在线电影而不得不安装Adobe Flash Player的日子吗？然后，您会不断收到有关更新Flash Player和Java Runtime Environment的烦人警报。自那时以来，Web技术已经变得更加强大，并且那些日子已经过去了。对于几乎所有应用程序，是时候停止提供依赖于插件的内容，而是开始利用Web技术了。

扩大您对每个人的影响。每个人都有浏览器，但是插件越来越少，尤其是在移动用户中。由于无需任何插件即可轻松使用Web，因此人们宁愿只是访问竞争对手的网站，也不愿安装插件。
让自己摆脱Flash和其他插件带来的额外的可访问性难题。
远离其他安全隐患。众所周知，即使经过无数次修补，Adobe Flash也不安全。 2015年，Facebook当时的首席安全官Alex Stamos要求Adobe停止使用Flash。

> 注意：由于其固有的问题以及缺少对Flash的支持，Adobe宣布他们将在2020年底停止支持它。从2020年1月开始，大多数浏览器默认情况下会阻止Flash内容，并且到2020年12月31日，所有浏览器将完全删除所有Flash功能。在此日期之后，将无法访问任何现有的Flash内容。

那你该怎么办？如果需要交互性，HTML和JavaScript可以轻松完成任务，而无需Java程序或过时的ActiveX / BHO技术。不应依赖Adobe Flash，而应使用HTML5视频来满足媒体需求，使用SVG来实现矢量图形，并使用Canvas来实现复杂的图像和动画。彼得·埃尔斯特（Peter Elst）早在几年前就已经在写书，认为Adobe Flash很少是完成这项工作的正确工具。至于ActiveX，甚至Microsoft的Edge浏览器也不再支持它。



