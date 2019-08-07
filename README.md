## 背景

某个工作日的下午，一个前端小伙伴找我说：“我的线上的页面原来是好好的，今天访问的时候怎么变成了下载了，我什么都没做呀？”正在这时，又有人反馈所有的 web 网页访问的时候都变成下载了。

经过紧急的排查，发现问题的原因是阿里云某个 `CDN` 节点的回源请求的构建出现了问题，导致 `OSS` 源站无法识别 `CDN` 请求，并对其插入 `content-disposition`头，最终影响了浏览器对文件的处理逻辑。(本应该浏览器渲染的，被当做附件下载)。

所以今天要和大家聊的就是导致这起线上问题的 `Content-disposition` 到底是何方神圣。

## Content-Disposition

`Content-Disposition` 有两种应用场景。

### 用在 HTTP 响应头中

场景一是用在 HTTP 的响应头中，指示响应的内容该以何种形式展示。是以内联的形式（即网页或者页面的一部分），还是以附件的形式下载并保存到本地。

`Content-Disposition` 的第一个参数的值有两个：

- `inline` 默认值，表示响应中的消息体会以页面的一部分或者整个页面的形式展示。
- `attachment` 表示响应的消息体应被下载到本地；大多数浏览器会出现一个“保存为”的对话框。

第二个参数是可选的 `filename`。当响应的内第一个参数指定为 `attachment` 时，浏览器会将响应中的内容下载下来，`filename` 可以指定下载后的文件名。

`Content-Disposition` 在响应头中可能会这样出现：

```
// 正常解析渲染
Content-Disposition: inline
// 下载文件
Content-Disposition: attachment
// 下载文件，并将文件保存为filename.jpg
Content-Disposition: attachment; filename="filename.jpg"
```

### 用在 multiparty/form-data 的请求体中

还有一种场景是，当页面上有表单，并且我们选择的表单提交方式为 `multiparty/form-data` 时，`Content-Disposition` 会出现在请求体中。其作用是说明对应的表单项的字段名是什么，表单中上传的文件名是什么。在该场景下第一个参数总是固定不变的 `form-data`，另外有两个可选参数。`name` 表示对应的表单项的字段名，`filename` 表示对应的上传的文件名。参数之间使用 `;` 进行分割。

`Content-Disposition` 在 `multiparty/form-data` 的请求体头中可能会这样出现：

```
Content-Disposition: form-data
Content-Disposition: form-data; name="fieldName"
Content-Disposition: form-data; name="fieldName"; filename="filename.jpg"
```

## 示例

接下来，我们通过一个示例来加深一下对它的印象。该示例主要演示两部分功能。

- 通过设置 `Content-Disposition: attachment` 在浏览器中下载保存请求的文件。
- 通过实现一个 post 请求，来说明 `Content-Disposition: form-data` 在请求体中的展示形式。

创建 `contentDispositionDemo` 目录，执行:

```
npm install express multiparty
```

服务端代码我们使用 `express`来实现，同时使用 `multiparty` 模块来处理 `multiparty/form-data` 的请求。 服务端代码如下：

```
// app.js
const express = require("express");
const multiparty = require("multiparty");
const app = express();
const port = 3000;
app.use(express.static("public"));

// 访问的时候，弹窗提示下载attachment.html，并保存为ddd.html
app.get("/attach", (req, res, next) => {
  const options = {
    root: __dirname,
    headers: {
      "Content-Disposition": "attachment;filename=ddd.html"
    }
  };
  res.sendFile("/attachment.html", options, err => {
    if (err) {
      next(err);
    } else {
      console.log("Sent:", "attachment.html");
    }
  });
});

// form-data表单提交
app.post("/user", (req, res, next) => {
  const form = new multiparty.Form();
  var name;
  var image;
  form.on("error", next);
  form.on("close", function(err) {
    console.log(err);
    console.log(name);
    console.log(image);
    res.send("资料提交成功！");
  });

  form.on("field", function(key, val) {
    if (key !== "image") name = val;
  });

  form.on("part", function(part) {
    if (!part.filename) return;
    if (part.name !== "image") return part.resume();
    image = {};
    image.filename = part.filename;
    image.size = 0;
    part.on("data", function(buf) {
      image.size += buf.length;
    });
  });

  form.parse(req);
});

app.listen(port, () => console.log(`App listening on port ${port}!`));

```

接下来我们创建一个简单的表单提交页面，用户需要填写姓名和一张照片。我们将该页面放在 `public` 目录下，页面代码如下：

```
<!--public/index.html-->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>content-disposition 示例</title>
    <style>
      form {
        padding: 20px;
        border: 1px gray solid;
        background: yellowgreen;
      }
      .form-item {
        padding: 10px 0;
      }
      #commit {
        display: block;
        width: 80px;
        height: 30px;
        text-align: center;
        line-height: 30px;
        border: 1px solid #666;
        cursor: pointer;
        background: #eee;
      }
    </style>
  </head>
  <body>
    <h1>请提交您的个人资料</h1>
    <form>
      <div class="form-item">
        <label for="name">姓名：</label>
        <input type="text" name="name" id="name" />
      </div>
      <div class="form-item">
        <label for="pic">照片：</label>
        <input type="file" name="image" id="image" />
      </div>
      <div class="form-item"><span id="commit">提交</span></div>
    </form>
  </body>
  <script>
    function upload() {
      var formData = new FormData();
      var name = document.querySelector("#name").value;
      var image = document.querySelector("#image").files[0];
      formData.append("name", name);
      formData.append("image", image);

      var xhr = new XMLHttpRequest();
      xhr.open("POST", "/user");
      xhr.onreadystatechange = function() {
        if (xhr.readyState === XMLHttpRequest.DONE && xhr.status === 200) {
          console.log(xhr.responseText);
          alert(xhr.responseText);
        }
      };
      xhr.send(formData);
    }

    document.querySelector("#commit").addEventListener(
      "click",
      function() {
        upload();
      },
      false
    );
  </script>
</html>

```

另外，再创建一个 HTML 页面，该页面用于浏览器访问的时候下载。具体代码内容随意：

```
<!--attachment.html-->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>保存文件</title>
  </head>
  <body>
    保存文件
  </body>
</html>

```

执行 `node app.js` 启动服务，当我们访问 `localhost:3000/attach`的时候，可以发现浏览器直接下载了 `attachment.html`，并将其保存为了 `ddd.html`。

以下为页面下载的截图：

![](https://static.daojia.com/assets/project/tosimple-pic/02_1565169034452.png)

响以下为响应头的截图：

![](https://static.daojia.com/assets/project/tosimple-pic/03_1565169078337.png)

访问`localhost:3000`，在首页的表单中输入信息并提交，然后我们看到下图中 `Content-Disposition` 的展示形式：

![](https://static.daojia.com/assets/project/tosimple-pic/04_1565169566322.png)

## 总结

由于当我们使用 `Content-Disposition` 默认值为 `inline`的时候，在请求的响应头中是不显示的，所以我猜很多前端的小伙伴可能对 `Content-Disposition` 了解不够。其主要用途还是用来告知浏览器如何处理响应中的内容。当然，出现文章开篇的事故是个意外。通过该文章对其有了较为详细的了解后，相信前端小伙伴们都可以熟练应用它来达到不同的目的了。
