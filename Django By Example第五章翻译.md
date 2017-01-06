（译者@ucag注：大家好，我是新来的翻译，希望大家多多交流。题目还是沿用老传统。有做的不严谨的地方还请大家指出来。）

(审校者@夜夜月注：人多力量大，第五章由@ucag两天内独立翻译完毕)

##**第五章**
##**在你的网站中分享内容**

在上一章中，你为你的网站建立了用户注册和认证系统。你学习了如何为用户创建定制化的个人资料模型以及如何将主流的社交网络的认证添加进你的网站。
在这一章中，你将学习如何通过创建一个  JavaScript 书签来从其他的站点分享内容到你的网站，你也将通过使用 jQuery 在你的项目中实现一些 AJAX 特性。
这一章涵盖了以下几点：

 * 创建一个`many-to-many`（多对多）关系
 * 定制表单（form）的行为
 * 在 Django 中使用 jQuery
 * 创建一个 jQuery 书签
 * 通过使用 sorl-thumbnail 来生成缩略图
 * 实现 AJAX 视图（views）并且使这些视图（views）和 jQuery 融合
 * 为视图（views）创建定制化的装饰器 （decorators）
 * 创建 AJAX 分页
##**建立一个能为图片打标签的网站**
我们将允许用户可以在我们网站中分享他们在其他网站发现的图片，并且他们还可以为这些图片打上标签。为了达到这个目的，我们将要做以下几个任务：

 * 定义一个模型来储存图片以及图片的信息
 * 新建一个表单（form）和视图（view）来控制图片的上传
 * 为用户创建一个可以上传他们在其他网站发现的图片的系统

首先，通过以下命令在你的 bookmarks 项目中新建一个应用：
    
    django-admin startapp images
    
像如下所示一样在你的 `settings.py` 文件中 `INSTALED_APPS` 设置项下添加 'images' :

    INSTALLED_APPS = [
        # ... 
        'images',
    ]

现在Django知道我们的新应用已经被激活了。
##**创建图像模型**
编辑 images 应用中的 `models.py`  文件，将以下代码添加进去：

```python
from django.db import models
from django.conf import settings
class Image(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL,
    related_name='images_created')
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200,blank=True)
    url = models.URLField()
    image = models.ImageField(upload_to='images/%Y/%m/%d')
    description = models.TextField(blank=True)
    created = models.DateField(auto_now_add=True,
                               db_index=True)
    def __str__(self):
        return self.title
```
我们将要使用这个模型来储存来自各个不同网站中被标记的图片。让我们来看看在这个模型中的字段：

 - `user`： 标记了这张图片 `User` 对象。这是一个 `ForeignKey`字段 (译者注：外键，即一对多字段)，因为它指定了一个一对多关系： 一个用户可以 post 多张图片， 但是每张图片只能由一个用户上传
 - `title`： 图片的标题
 - `slug`： 一个只包含字母、数字、下划线、和连字符的标签， 用于创建优美的 搜索引擎友好（SEO-friendly）的 URL（译者注：slug 这个词在中文没有很好的对应翻译，所以就请大家记住“slug 表示的是只有字母、数字、下划线和连字符的标签”。如果有仔细看过 Django 官方文档的读者就会知道： slug 是一个新闻术语， 而 Django 的开发目的也是为了更好的编辑新闻， 所以这里就不难理解为什么 Django 中会出现 slug 字段了）
 - `url`： 这张图片的源 URL
 - `image`： 图片文件
 - `description`： 一个可选的图片描述字段
 - `created`： 用于表明一个对象在数据库中创建时的时间和日期。由于我们使用了`auto_now_add` ,当对象被创建时候时间和日期将会被自动设置，我们使用了 `db_index=True` ，所以 Django 将会在数据库中为这个字段创建索引

> 数据库索引改善了查询的执行。考虑为这个字段设置 `db_index=True` 是因为你将要很频繁地使用 `filter()`，`exclude()`,`order_by()` 来执行查询。`ForeignKey` 字段或者带有`unique=True`的字段表明了一个索引的创建。你也可以使用`Meta.index_together`来为多个字段创建索引。

我们将要重写 Image 模型的 `save()`方法来自动的生成`slug`字段。这个 `slug`字段基于`title`字段的值。像下面这样导入`slugify()`函数， 然后在 Image 模型中添加一个 `save()` 方法：

```python
from django.utils.text import slugify
class Image(models.Model):
    # ...
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
            super(Image, self).save(*args, **kwargs)
```
在这段代码中，我们使用了 Django 提供的`slugify()`函数在没有提供`slug`字段时根据给定的图片标题自动生slug,然后，我们保存了这个对象。我们自动生成slug，这样的话用户就不用自己输入`slug`字段了。
##**建立多对多关系**
我们将要在 Image 模型中再添加一个字段来保存喜欢这张图片的用户。因此，我们需要一个多对多关系。因为一个用户可能喜欢很多张图片，一张图片也可能被很多用户喜欢。
在 Image 模型中添加以下字段：

```python
user_like = models.ManyToManyField(settings.AUTH_USER_MODEL,
                                   related_name='images_liked',
                                   blank=True)
```


当你定义一个`ManyToMany`字段时，Django 会用两张表主键（primary key）创建一个中介联接表（译者注：就是新建一张普通的表，只是这张表的内容是由多对多关系双方的主键构成的）。`ManyToMany`字段可以在任意两个相关联的表中创建。
同`ForeignKey`字段一样，`ManyToMany`字段的`related_name`属性使我们可以命名另模型回溯（或者是反查）到本模型对象的关系。`ManyToMany`字段提供了一个多对多管理器（manager），这个管理器使我们可以回溯相关联的对象比如：`image.users_like.all()`或者从一个`user`中回溯，比如：`user.images_liked.all()`。
打开命令行，执行下面的命令以创建首次迁移：

```
python manage.py makemigrations images
```
你能看见以下输出：

```
Migrations for 'images':
    0001_initial.py:
        - Create model Image
```
现在执行这条命令来应用你的迁移：

```
python manage.py migrate images
```
你将会看到包含这一行输出：

```
Applying images.0001_initial... OK
```
现在 Image 模型已经在数据库中同步了。
##**注册 Image 模型到管理站点中**
编辑 `images` 应用的 `admin.py` 文件，然后像下面这样将 `Image` 模型注册到管理站点中：

```python
from django.contrib import admin
from .models import Image
class ImageAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'image', 'created']
    list_filter = ['created']

admin.site.register(Image, ImageAdmin)
```
使用命令`python manage.py runserver`打开开发服务器，在浏览器中打开`http://127.0.0.1:8000/admin/`,可以看到`Image`模型已经注册到了管理站点中：

![Django-5-1][1]
##**从其他网站上传内容**
我们将使用户可以给从他们在其他网站发现的图片打上标签。用户将要提供图片的 URL ，标题，和一个可选的描述。我们的应用将要下载这幅图片，并且在数据库中创建一个新的 `Image` 对象。
我们从新建一个用于提交图片的表单开始。在images应用的路径下创建一个 `forms.py` 文件，在这个文件中添加如下代码：

```python
from django import forms
from .models import Image
class ImageCreateForm(forms.ModelForm):
    class Meta:
        model = Image
        fields = ('title', 'url', 'description')
        widgets = {
            'url': forms.HiddenInput,
        }
```
如你所见，这是一个通过`Image`模型创建的`ModelForm`（模型表单），但是这个表单只包含了 `title`,`url`,`description`字段。我们的用户不会在表单中直接为图片添加 URL。相反的，他们将会使用一个 JavaScropt 工具来从其他网站中选择一张图片然后我们的表单将会以参数的形式接收这张图片的 URL。我们覆写 `url` 字段的默认控件（widget）为一个`HiddenInput`控件，这个控件将会被渲染为属性是 `type="hidden"`的 HTML 元素。使用这个控件是因为我们不想让用户看见这个字段。
##**清洁表单字段**
（译者注：原文标题是：cleaning form fields,在数据处理中有个术语是“清洗数据”，但是这里的清洁还有“使其整洁”的含义，感觉更加符合`clean_url`这个方法的定位。）
为了验证提供的图片 URL 是否合法，我们将检查以`.jpg`或`.jpeg`结尾的文件名，来只允许JPG文件的上传。Django允许你自定义表单方法来清洁特定的字段，通过使用以`clean_<fieldname>`形式命名的方法来实现。这个方法会在你为一个表单实例执行`is_valid()`时执行。在清洁方法中，你可以改变字段的值或者为某个特定的字段抛出错误当需要的时候，将下面这个方法添加进`ImageCreateForm`:

```python
def clean_url(self):
    url = self.cleaned_data['url']
    valid_extensions = ['jpg', 'jpeg']
    extension = url.rsplit('.', 1)[1].lower()
    if extension not in valid_extensions:
        raise forms.ValidationError('The given URL does not ' \
                                   'match valid image extensions.')
    return url
```
在这段代码中，我们定义了一个`clean_url`方法来清洁`url`字段，这段代码的工作流程是：
 
 * 1. 我们从表单实例的`cleaned_data`字典中获取了`url`字段的值
 * 2. 我们分离了 URL 来获取文件扩展名，然后检查它是否为合法扩展名之一。如果它不是一个合法的扩展名，我们就会抛出`ValidationError`,并且表单也不会被认证。我们执行的是一个非常简单的认证。你可以使用更好的方法来验证所给的 URL 是否是一个合法的图片。
除了验证所给的 URL， 我们还需要下载并保存图片文件。比如，我们可以使用操作表单的视图来下载图片。不过，我们将采用一个更加通用的方法 ———— 通过覆写我们模型表单中`save()`方法来完成这个任务。

##**覆写模型表单中的save()方法**
如你所知，`ModelForm`提供了一个`save()`方法来保存目前的模型实例到数据库中，并且返回一个对象。这个方法接受一个布尔参数`commit`，这个参数允许你指定这个对象是否要被储存到数据库中。如果`commit`是`False`，`save()`方法将会返回一个模型实例但是并不会把这个对象保存到数据库中。我们将覆写表单中的`save()`方法，来下载图片然后保存它。
将以下的包在`foroms.py`中的顶部导入：

```python
from urllib import request
from django.core.files.base import ContentFile
from django.utils.text import slugify
```
把`save()`方法加入`ImageCreateForm`中：

```python
def save(self, force_insert=False,
         force_update=False,
         commit=True):
    image = super(ImageCreateForm, self).save(commit=False)
    image_url = self.cleaned_data['url']
    image_name = '{}.{}'.format(slugify(image.title),
    image_url.rsplit('.', 1)[1].lower())
# 从给定的 URL 中下载图片
    response = request.urlopen(image_url)
    image.image.save(image_name,
                    ContentFile(response.read()),
                    save=False)
    if commit:
        image.save()
    return image
```
我们覆写的`save()`方法保持了`ModelForm`中需要的参数、
这段代码：
 1. 我们通过调用`save()`方法从表单中新建了一个`image`对象，并且`commit=False`
 2. 我们从表单的`cleaned_data`字典中获取了 URL 
 3. 我们通过结合`image`的标题 slug 和源文件的扩展名生成了图片的名字
 4. 我们使用 Python 的 `urllib` 模块来下载图片，然后我们调用`save()`方法把图片传递给一个`ContentFiel`对象，这个对象被下载的文件所实例化。这样，我们就可以将我们的文件保存到项目中的 media 路径下。我们传递了参数`comiit=False`来避免对象被保存到数据库中。
 5. 为了保持和我们覆写的`save()`方法一样的行为，我们将在`commit`参数为`Ture`时保存表单到数据库中。

现在我们需要一个新的视图来控制我们的表单。编辑 iamges 应用的`views.py`文件，然后将以下代码添加进去:

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .forms import ImageCreateForm

@login_required
def image_create(request):
    """
    View for creating an Image using the JavaScript Bookmarklet.
    """
    if request.method == 'POST':
        # form is sent
        form = ImageCreateForm(data=request.POST)
        if form.is_valid():
            # form data is valid
            cd = form.cleaned_data
            new_item = form.save(commit=False)
            # assign current user to the item
            new_item.user = request.user
            new_item.save()
            messages.success(request, 'Image added successfully')
            # redirect to new created item detail view
            return redirect(new_item.get_absolute_url())
    else:
        # build form with data provided by the bookmarklet via GET
        form = ImageCreateForm(data=request.GET)

    return render(request, 'images/image/create.html', {'section': 'images',
                                                        'form': form})

```
我们给`image_create`视图添加了一个`login_required`装饰器，来阻止未认证的用户的连接。这段代码完成下面的工作：
 1. 我们先从 GET 中获取初始数据来创建一个表单实例。这个数据由来自外部网站图片的`url`和`title`属性构成，并且将由我们等会儿要创建的 JavaScript 工具提供。现在我们只是假设这里有初始数据。
 2. 如果表单被提交我们将检查它是否合法。如果这个表单是合法的，我们将新建一个`Image`实例，但是我们通过传递`commit=False`来保证这个对象将不会保存到数据库中。
 3. 我们将绑定当前用户（user）到一个新的`iamge`对象。这样我们就可以知道是谁上传了每一张图片。
 4. 我们把 iamge 对象保存到了数据库中
 5. 最后，我们使用 Django 的信息框架创建了一条上传成功的消息然后重定向用户到新图像的规范 URL 。我们没有在 Image 模型中实现`get_absolute_url()`方法，我们等会儿将编写它。

在你的 images 应用中创建一个叫做`urls.py`的新文件，然后添加如下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^create/$', views.image_create, name='create'),
]
```
像下面这样编辑在你项目文件夹中的主`urls.py`文件，将我们刚才为 images 应用创建的 url 模式添加进去：

```python
urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^account/', include('account.urls')),
    url(r'^images/', include('images.urls', namespace='images')),
]
```
最后，你需要创建一个模板来渲染你的表单。在你的 images 应用路径下创建如下路径结构：

```
templates/
   images/
       image/
          create.html
```
编辑新的 `create.html` 模板然后添加以下代码进去：

```html
{% extends "base.html" %}

{% block title %}Bookmark an image{% endblock %}

{% block content %}
    <h1>Bookmark an image</h1>
    <img src="{{ request.GET.url }}" class="image-preview">
    <form action="." method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <input type="submit" value="Bookmark it!">
    </form>
{% endblock %}

```
现在在你的浏览器中打开`http://127.0.0.1:8000/images/create/?title=...&url=...`，记得在 后面传递 GET 参数`title`和`url`来提供一个已存在的JPG图像的 URL 。
举个例子，你可以使用像下面这样的 URL：
```
http://127.0.0.1:8000/images/create/?title=%20Django%20and%20Duke&url=http://upload.wikimedia.org/wikipedia/commons/8/85/Django_Reinhardt_and_Duke_Ellington_%28Gottlieb%29.jpg
```
你可以看到一个带有图片预览的表单，就像下面这样：
![Django-5-2][2]
添加描述然后点击 **Bookmark it！**按钮。一个新的 `Image`对象将会被保存在你的数据库中。你将会得到一个错误，这个错误指示说`Image`模型没有`get_absolute_url()`方法。现在先不要担心这个，我们待会儿将添加这个方法、在你的浏览器中打开`http://127.0.0.1:8000/admin/images/image/`，确定新的图像对象已经被保存了。
##**用 jQuery 创建一个书签**
书签是一个保存在浏览器中包含 JavaScript 代码的标签，用来拓展浏览器功能。当你点击书签的时候， JavaScript 代码会在浏览器显示的网站中被执行。这是一个在和其它网站交互时非常有用的工具。

一些在线服务，比如 Pinterest 实现了他们自己的书签来让用户可以在他们的平台中分享来自其他网站的内容，我们将以同样的方式创建一个书签，让用户可以在我们的网站中分享来自其他网站的图片。
我们将使用 jQuery 来创建我们的书签。 jQuery 是一个流行的 JavaScript 框架， 这个框架允许你快速开发客户端的功能。你可以在官网中更多的了解 jQuery: http://jquery.com/ 

你的用户将会像下面这样在他们的浏览器中添加书签然后使用它：
 1. 用户从你的网站中拖拽一个链接到他的浏览器。这个链接在它的`href`属性中包含了 JavaScript 代码。这段代码将会被储存到书签当中。
 2. 用户访问任意一个网站，然后点击这个书签， 这个书签的 JavaScript 代码就被执行了。

由于 JavaScript 代码将会以书签的形式被储存，之后你将不能更新它。这是个很显著的缺点，但是你可以通过实现一个简单的激活脚本来解决这个问题，这个脚本从一个 URL 中加载 JavaScript。你的用户将会以书签的形式来保存这个激活脚本，这样你就能在任何时候更新书签代码的内容了。我们将会采用这个方法来创建我们的书签。我们开始吧！
（译者注：上面这一段似乎有一点难以理解，其实很简单，就是把 JavaScript 保存在后端，只让用户保存一个能获取这段 JavaScript 的 url，url 是由书签来获取的。用户保存的就是这个含有获取 url 的 JavaScript 书签。）

在 image/templates/ 下创建一个新的模板，把它命名为 `bookmarklet_launcher.js`。这个就是我们的激活脚本了。将以下 JavaScript 代码添加进这个文件

```JavaScript
(function(){
    if(window.myBookmarklet!==undefined){
        myBookmarklet();
    }
    else{
        document.body.appendChild(document.createElement('script')).src='http://127.0.0.1:8000/static/js/bookmarklet.js?r='+Math.floor(Math.random()*99999999999999999999);
    }
})();
```
这段脚本通过检查 `myBookmarklet`变量是否被定义来检测书签是否被加载。这样，我们就可以避免在用户重复点击书签时重复加载。如果 `myBookmarklet` 没有被定义，我们就再加载一个 JavaScript 文件来在文档中添加一个`<script>`元素。 这个 `script` 标签加载 `bookmarklet_launcher.js`脚本，将一个随机数作为参数来防止加载浏览器缓存中的文件。

我们当前的 bookmarklet 代码位于 `bookmarklet.js` 静态文件中。这使我们在不要求用户更新书签的情况下更新我们代码。让我们把书签添加进 dashboard 页，我们的用户就可以将它拷贝到他们的书签中。

编辑  account/dashboard.html 模板，像如下一样更改它：

```html
{% extends "base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    <h1>Dashboard</h1>
    
    {% with total_images_created=request.user.images_created.count %}
        <p>Welcome to your dashboard. You have bookmarked {{ total_images_created }} image{{ total_images_created|pluralize }}.</p>
    {% endwith %}
    
    <p>Drag the following button to your bookmarks toolbar to bookmark images from other websites → <a href="javascript:{% include "bookmarklet_launcher.js" %}" class="button">Bookmark it!</a><p>
    
    <p>You can also <a href="{% url "edit" %}">edit your profile</a> or <a href="{% url "password_change" %}">change your password</a>.<p>
{% endblock %}
```
这个 danshboard 展示了用户所标记的图片总数。我们使用`{% with %}`模板标签来设置一个带有用户标记图片总数的参数。我们也引入了一个带有`href`属性的链接，这个链接含有我们的书签激活脚本。我们从`  bookmarklet_launcher.js`模板中引入 JavaScript 脚本。

在你的浏览器中打开` http://127.0.0.1:8000/account/ `,你可以看到如下页面：
![此处输入图片的描述][3]


拖拽`Bookmark it!`链接到你的浏览器的书签工具栏中。

现在创建下面几个路径和文件在 images 应用路径中：

 - static/
 - js/
 - bookmarklet.js

你会在本章示例代码文件夹中的images 应用路径下找到 `static/css/` 路径。复制 `css/` 路径到你的代码文件夹下的`static/`中。` css/bookmarklet.css`文件为我们的 JavaScript 书签提供了样式。
编辑`  bookmarklet.js`静态文件，然后添加以下 JavaScript 代码：

```JavaScript
(function(){
  var jquery_version = '2.1.4';
  var site_url = 'http://127.0.0.1:8000/';
  var static_url = site_url + 'static/';
  var min_width = 100;
  var min_height = 100;

  function bookmarklet(msg) {
      // Here goes our bookmarklet code
);
 // Check if jQuery is loaded
  if(typeof window.jQuery != 'undefined') {
    bookmarklet();
  } else {
    // Check for conflicts
    var conflict = typeof window.$ != 'undefined';
    // Create the script and point to Google API
    var script = document.createElement('script');
    script.setAttribute('src','http://ajax.googleapis.com/ajax/libs/jquery/' + jquery_version + '/jquery.min.js');
    // Add the script to the 'head' for processing
    document.getElementsByTagName('head')[0].appendChild(script);
    // Create a way to wait until script loading
    var attempts = 15;
    (function(){
      // Check again if jQuery is undefined
      if(typeof window.jQuery == 'undefined') {
        if(--attempts > 0) {
          // Calls himself in a few milliseconds
          window.setTimeout(arguments.callee, 250)
        } else {
          // Too much attempts to load, send error
          alert('An error ocurred while loading jQuery')
        }
      } else {
          bookmarklet();
      }
    })();
  }

})()
```
这是主要的 jQuery 加载脚本，当脚本已经加载到当前网站中时，它负责调用 JQuery 或者是从 Google 的 CDN 中加载 jQuery。当 jQuery 被加载，它会执行` bookmarklet()`函数，该函数包含我们的bookmarklet代码。我们还在这个文件顶部设置几个变量：

 - `jquery_version`: 加载的 jQuery 版本 
 - `site_url`和`static_url`：我们网站的主URL 和各自静态文件的主URL
 - `min_width`和`min_height `：我们的书签在网站中将要寻找的图像支持的最小宽度和最小高度，

现在让我们来实现 `bookmarklet`函数，编辑`  bookmarklet()`，让它看起来像这样：

```JavaScript
  function bookmarklet(msg) {
    // load CSS
    var css = jQuery('<link>');
    css.attr({
      rel: 'stylesheet',
      type: 'text/css',
      href: static_url + 'css/bookmarklet.css?r=' + Math.floor(Math.random()*99999999999999999999)
    });
    jQuery('head').append(css);

    // load HTML
    box_html = '<div id="bookmarklet"><a href="#" id="close">&times;</a><h1>Select an image to bookmark:</h1><div class="images"></div></div>';
    jQuery('body').append(box_html);

	  // close event
	  jQuery('#bookmarklet #close').click(function(){
      jQuery('#bookmarklet').remove();
	  });
      };
```

这段代码运行如下：
 1. 我们加载了`bookmarklet.css`样式表，使用一个随机的数字作为参数来避免浏览器的缓存
 2. 我们添加了定制的 HTML 到当前网站的`<body>`元素中。这个HTML由包含在当前网站寻找到的图片的`<div>`元素构成的。
 3. 我们添加了一个事件，当用户点击我们的 HTML 块中的关闭链接时，我们将移除我们添加进去的 HTML。我们使用 `#bookmarklet``#close`选择器来找到带有一个 ID 为`close`的 HTML 元素，这个 HTML 元素的父ID是 `bookmarklet`。jQuery 选择器允许你寻找 HTML 元素。jQuery 选择器返回所有给定的 CSS 选择器找到的元素，你可以在这个链接中找到一组 jQuery 选择器：http://api.jquery.com/category/selectors/

在加载了 CSS 样式表和 HTML 后，我们需要在网站中找到图片。在`bookmarklet()`函数的底部添加如下代码：

```JavaScript
    // find images and display them
    jQuery.each(jQuery('img[src$="jpg"]'), function(index, image) {
      if (jQuery(image).width() >= min_width && jQuery(image).height() >= min_height)
      {
        image_url = jQuery(image).attr('src');
        jQuery('#bookmarklet .images').append('<a href="#"><img src="'+ image_url +'" /></a>');
      }
    });
```
这段代码使用了`img[src$="jpg"]`选择器来找到所有的`<img>` HTML 元素，并且这些元素的`src`属性以`jpg`结尾。这意味着我们会找到当前网页中所有的 JPG 图片。我们通过`each()`方法来遍历所有的结果。我们添加了`<div class="images">` HTML 容器用以放置图片，容器的的尺寸刚好比`min_width`和` min_width`大一点。

这个 HTML 容器现在包含了可以被打上标签的图片，我们想要用户点击他们需要的图片然后给他们打上标签。在`bookmarklet()`函数中添加以下代码：

```JavaScript
    // when an image is selected open URL with it
    jQuery('#bookmarklet .images a').click(function(e){
      selected_image = jQuery(this).children('img').attr('src');
      // hide bookmarklet
      jQuery('#bookmarklet').hide();
      // open new window to submit the image
      window.open(site_url +'images/create/?url='
                  + encodeURIComponent(selected_image)
                  + '&title=' + encodeURIComponent(jQuery('title').text()),
                  '_blank');
    });
```
这段代码按照如下流程运行：
 1. 我们把一个`clck()`事件绑定到了图片的链接元素上
 2. 当一个用户点击一个图片时我们新建了一个变量`selected_image`，这个变量包含了被选择的图片的 URL。
 3. 我们隐藏了书签然后在浏览器中打开一个新的窗口，这个窗口访问了我们的网站中为一个新的图片打标签的 URL 。我们传递了网站的`title`元素和被选中图片的 URL 作为 GET 参数。

在你的浏览器中随便选择一个网址打开，然后点击你的书签。你将会看到一个白色的新窗口出现在当前网页上，它展示了所有尺寸大于 100*100px 的 JPG 图片，它看起来就像下面的例子一样：
![django-5-4][4]
因为我们已经开启了 Django 的开发服务器，使用 HTTP 来提供页面， 由于安全限制，书签将不能在 HTTPS 上工作。

如果你点击一幅图片，你将会被重定向到创建图片的页面，请求地址传递了网站的标题和被选中图片的 URL 作为 GET 参数。

![Django-5-5][5]
恭喜！这是你的第一个 JavaScript 书签！现在它已经和你的 Django 项目成为一体！

##**为你的图片创建一个详情视图**
我们将创建一个简单的详情视图，用于展示一张已经保存在我们的网站中的图片。打开 images 应用的`views.py`，将以下代码添加进去：

```python
from django.shortcuts import get_object_or_404
from .models import Image
def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    return render(request, 'images/image/detail.html', {'section':                                                                 'images','image': image})
```
这是一个用于展示图片的简单视图。编辑 iamges 应用的 `urls.py`，添加以下 URL 模式：

```python
url(r'^detail/(?P<id>\d+)/(?P<slug>[-\w]+)/$',
              views.image_detail, name='detail'),
```
编辑 images 应用的`models.py`,并且将`get_absolute_url()`方法添加进 `Image` 模型:

```python
from django.core.urlresolvers import reverse
class Image(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse('images:detail',args=(self.id,self.slug))
```
记住，为对象提供精确 URL 的通用模式是在模型中定义`get_absolute_url()`方法。

最后，在 images 应用的 模版路径`/images/image/`中新建一个模板，命名为`detail.html`，添加以下代码：

```html
{% extends "base.html" %}

{% block title %}{{ image.title }}{% endblock %}

{% block content %}
    <h1>{{ image.title }}</h1>
    <img src="{{ image.image.url }}" class="image-detail">
    {% with total_likes=image.users_like.count %}
        <div class="image-info">
                <div>
                    <span class="count">
                        {{ total_likes }}like{{ total_likes|pluralize }}
                    </span>
                 </div>
                 {{ image.description|linebreaks }}
        <div class="image-likes">
            {% for user in image.users_like.all %}
                <div>
                    <img src="{{ user.profile.photo.url }}">
                    <p>{{ user.first_name }}</p>
                </div>
            {% empty %}
                Nobody likes this image yet.
            {% endfor %}
        </div>
    {% endwith %}
{% endblock %}
```
这个模版用来展示一张被打标签图片。我们使用`{% with %}`标签来保存所有统计`user likes`查询集（QuerySet）的结果，并将这个结果保存在一个新的变量`total_likes`中。这样我们就可以避免计算两次查询集（QuerySet）的结果。我们也引入了图片的描述，迭代了`image.users_like.all`来展示所有喜欢这张图片的用户。

>使用`{% with %}`模版标签来防止 Django 做多次查询是很有用的

现在使用书签来为一张图片打上标签。在你提交图片之后你将会被重定向图片详情页面。这张图片将会包含一条提交成功的消息，效果如下：
![Django-5-6][6]
##**使用 sorl-thumbnail 创建缩略图**
我们在详情页展示原图片，但是不同的图片的尺寸是不同的。一些图片源文件或许会非常大，加载他们会耗费很长时间。展示规范图片的最好方法是生成缩略图。我们将使用一个 Django 应用，叫做`sorl-thumbnail`。

打开终端，用下面的命令来安装`sorl-thumbnail`：

```python
pip install sorl-thumbnail==12.3
```
编辑 bookmarklet 项目文件的`settings.py`,将`sorl-thumbnail`添加进`INSTALLED_APPS`.

运行下面的命令来同步你的数据库：

```python
python manage.py migrate
```
你看到的输出中应该包含下面这一行：

```
Creating table thumbnail_kvstore
```
`sorl-thumbnail`应用提供了不同的方法来定义一张图片的缩略图。它提供了`{% thumbnail %}`模版标签来在模版中生成缩略图，同时还有一个定制的`ImageField`字段，如果你想要在你的模型中定制缩略图的话。我们将要使用这个模版标签。编辑 `images/image/detail.html`模版，删除这一行：

```html
<img src="{{ image.image.url }}" class="image-detail">
```
替换成：

```html
{% load thumbnail %}
{% thumbnail image.image "300" as im %}
<a href="{{ image.image.url }}">
<img src="{{ im.url }}" class="image-detail">
</a>
{% endthumbnail %}
```
这里，我们定义了一个固定宽度为 300px 的缩略图。当用户第一次加载这页面时，缩略图将会被创建。生成的缩略图将会在接下来的请求中被使用。运行`python manage.py runserver`开启开发服务器，连接到一张已有图片的详情页。缩略图将会生成并展示在网站中。

`sorl-thumbnail`应用提供了几个选择来定制你的缩略图，包括裁减算法和能被应用的不同效果。如果你有任何生成缩略图的疑难点，你可以在你的设置中添加`THUMBNAIL_DEBUG = TRUE`来获得 debug 信息。你可以阅读`sorl-thumbnail`的完整文档：http://sorl-thumbnail.readthedocs.org/
##**用 jQuery 添加 AJAX 动作**
现在我们将在你的应用中添加 AJAX 动作。AJAX 源于 Asynchronous JavaScript and XML（异步 JavaScript 和 XML）。这个术语包含一组可以制造异步 HTTP 请求的技术，它包含从服务器异步发送和接收数据，不需要重载整个页面，虽然它的名字里有 XML， 但是 XML 不是必需的。你可以以其他的格式发送或者接收数据，如 JSON， HTML，或者是纯文本。

我们将在图片详情页添加一个供用户点击的链接，表示他们喜欢这张图片。我们将会用 AJAX 来避免重载整个页面。首先，在 `views.py` 中创建一个可供用户点击“喜欢”或“不喜欢”的视图。编辑 images 应用的`views.py`，将以下代码添加进去：

```python
@login_required
@require_POST
def image_like(request):
    image_id = request.POST.get('id')
    action = request.POST.get('action')
    if image_id and action:
        try:
            image = Image.objects.get(id=image_id)
            if action == 'like':
                image.users_like.add(request.user)
            else:
                image.users_like.remove(request.user)
            return JsonResponse({'status':'ok'})
        except:
            pass
    return JsonResponse({'status':'ko'})
```
我们在这个视图中使用了两个装饰器。 `login_required` 装饰器阻止未登录的用户连接到这个视图。`require_GET` 装饰器返回一个`HttpResponseNotAlloed`对象（状态吗：405）如果 HTTP 请求不是 GET 。这样就可以只允许 GET 请求来访问这个视图。 Django 同样也提供了`require_POST`装饰器来只允许 POST 请求，以及一个可让你传递一组请求方法作为参数的 `require_http_methods`装饰器。

在这个视图中我们使用了两个 GET 参数：
 1. `image_id`:用户操作的 image 对象的 ID 
 2. `action`: 用户想要执行的动作。我们把它的值设定为`like`或者'`dislike`
 
我们在`Image`模型的多对多字段`users_like`上使用 Django 提供的管理器来添加或者删除对象关系通过调用`add()`或者`remove()`方法来执行这些动作。调用`add()`时传递一个存在于关联模型中的对象集不会重复添加这个对象，同样，调用`remove()`时传递一个不存在于关联模型中的对象集什操作也不会执行。另一个有用的多对多管理器是`clear()`，它将删除所有的关联对象集。

最后，我们使用 Django 提供的`JsonResponse`类来将给你定的对象转换为一个 JSON 输出，这个类返回一个带有`application/json`内容类型的 HTTP 响应。

编辑 `images` 应用中的 `urls.py`,添加以下 URL 模式：

```
url(r'^like/$', views.image_like, name='like'),
```
##**加载 jQuery**
我们需要在我们的图片详情页中添加 AJAX 功能。我们首先将在 `base.html`模版中引入 AJAX。编辑 account 应用的 `base.html`模版，然后将以下代码在`</body>`标签前添加以下代码：

```html
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/
jquery.min.js"></script>
<script>
  $(document).ready(function(){
    {% block domready %}
    {% endblock %}
    });
</script>
```
我们从 Google 加载 jQuery 框架，Google提供了一个在高速内容分发网络中的流行 JavaScript 框架。你也可以自己下载 jQuery, 地址：http://jquery.com/ 。然后将下载的文件添加进应用的`static`路径下。

我们添加`<script>`标签来引入 JavaScript 代码。`$(dovument).ready()`是一个 jQuery 函数，这个函数会在 DOM 层加载完毕后执行。 DON 源于 Document Object Model。当一个页面被载入时，DOM 会由浏览器创建， DOM 被创建为一个树对象。通过在这个函数中包含我们的代码来确保我们可以与DOM中加载的所有HTML元素都能进行交互操作。我们的代码仅仅会在 DOM 对象被加载完毕之后执行。

在文档预处理函数中，我们会在模板中引入一个 Django 模板块叫做 **domready**, 在扩展了基础模版之后将会引入特定的 JavaScript 。

不要将 JavaScript 代码和 Django 模板标签搞混了。 Django 模板语言是在服务端被渲染并输出最终的 HTML 文档，JavaScript 是在客户端被执行的。在某些情况下，使用 Django 动态生成 JavaScript 很有用。

>在这一章中，我们在 Django 模板中引入（include）了 JavaSript 代码。更好的引入方法是加载（load） JavaSript. `js`文件是作为静态文件被提供的，特别在有大量脚本时尤其如此。

##**AJAX 请求中的跨站请求攻击（CSRF）**
你已经在第二章了解到了跨站请求攻击，在CSRF保护激活的情况下， Django 会检查所有 POST 请求中的 CSRF token。当你提交表但时，你可以使用`{% csrf_token %}`模板标签来发送带有 token 的表单。无论如何，像 POST 请求一样对 AJAX 请求传递CDRF token 有一点点不方便。因此，Django 允许你在你的 AJAX 请求中设置一个定制的 `X-CSRFToken` token 头（header）。这允许你安装一个 jQuery 或者任意 JavaScript 库来自动设置`X-CSRFToken`头在每一次请求中。

为了在所有的请求中加入 token ，你需要：
 1. 从 `csrftoken` cookie 中检索 CSRF token，它在CSRF保护激活的情况下会被设置
 2. 使用 `X-CSRFToken`头发送 token 到 AJAX 中

你可以找到更多关于 CSRF 保护 和 AJAX 的信息：http://docs.djangoproject.com/en/1.8/ref/csrf/#ajax

在你的`base.html`模板中添加最后一段代码：

```html
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/
jquery.min.js"></script>
<script src=" http://cdn.jsdelivr.net/jquery.cookie/1.4.1/jquery.
cookie.min.js "></script>
<script>
  var csrftoken = $.cookie('csrftoken');
  function csrfSafeMethod(method) {
    // these HTTP methods do not require CSRF protection
    return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
}
$.ajaxSetup({
  beforeSend: function(xhr, settings) {
    if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
      xhr.setRequestHeader("X-CSRFToken", csrftoken);
  }
}
});
$(document).ready(function(){
    {% block domready %}
    {% endblock %}
  });
</script>
```
上面这段代码解释如下：
 1. 我们从一个公共的 CDN 中载入了一个 jQuery Cookie 插件，这样我们就可以和 cookies 交互。
 2. 读取 `csrftoken` cookie
 3. 我们将定义`csrfSafeMethod`函数来检查一个 HTTP 方法是否安全。安全方法不要求 CSRF 保护，他们分别是 GET, HEAD, OPTIONS, TRACE。
 4. 我们用`$.ajaxSetup()`设置了 jQuery AJAX 请求,在每个 AJAX 请求执行前，我们会检查请求方法是否安全和当前请求是否跨域名。如果请求是不安全的，我们将用从 cookie 中获得的值来设置 `X-CSRFToken`头。这个设置将会应用到所有由 jQuery 执行的 AJAX 请求中

CSRF token将会在所有的不安全 HTTP 方法的 AJAX 请求中引入，比如 POST, PUT

##**用 JQuery 执行 AJAX请求**
编辑 images 应用中的 `images/image/detailmhtml`模板，删除这一行：

```
{% with total_likes=image.users_like.count %}
```
替换为：

```
{% with total_likes=image.users_like.count users_like=image.users_like.all %}
```
用`image-info`类属性修改`<div`元素：

```html
        <div class="image-info">
                <div>
                    <span class="count">
                        <span class="total">{{ total_likes }}</span>
                        like{{ total_likes|pluralize }}
                    </span>
                    <a href="#" data-id="{{ image.id }}" data-action="{% if request.user in users_like %}un{% endif %}like" class="like button">
                        {% if request.user not in users_like %}
                            Like
                        {% else %}
                            Unlike
                        {% endif %}
                    </a>
                </div>
            {{ image.description|linebreaks }}
        </div>
```
首先，我们添加了一个变量到`{% with %}`模板标签中来保存`image.uers_like.all`查询接的结果来避免执行两次查询。展示喜欢这张图片用户的总数，包含一个“like/unlike”链接。我们检查用户是否在关联对象``user_likes`中，基于当前的用户和图片的关系展示 like 或者 unlike。。我们将以下属性添加进了`<a>` HTML 元素中：

 - `data-id`:被展示图片的 ID
 - `data-action`：当用户点击这个链接时执行这个动作。这个动作可以是 like 或者是 unlike
我们将会在向 `iamge_like`视图的 AJAX 请求中添加这两个属性值。当一个用户点击`like/unlike`链接时，我们需要在客户端执行以下几个动作：
 1. 调用 AJAX 视图，并把图片的 ID 和动作参数传递进去
 2. 如果 AJAX 请求成功， 更新`<a>` HTML元素的`data-action`属性（like / unlike），根据此来修改它展示的文本
 3. 更新展示的 likes 的总数

在`images/image/detail.html`模板中添加`domready`块，使用如下 JavaScript 代码：

```html
{% block domready %}
    $('a.like').click(function(e){
        e.preventDefault();
        $.post('{% url "images:like" %}',
            {
                id: $(this).data('id'),
                action: $(this).data('action')
            },
            function(data){
                if (data['status'] == 'ok')
                {
                    var previous_action = $('a.like').data('action');

                    // toggle data-action
                    $('a.like').data('action', previous_action == 'like' ? 'unlike' : 'like');
                    // toggle link text
                    $('a.like').text(previous_action == 'like' ? 'Unlike' : 'Like');

                    // update total likes
                    var previous_likes = parseInt($('span.count .total').text());
                    $('span.count .total').text(previous_action == 'like' ? previous_likes + 1 : previous_likes - 1);
                }
        });

    });
{% endblock %}
```
这段代码工作流程如下：
 1. 我们使用`$.('a.like')` jQuery 选择器来找到所有的 class 属性是 like 的`<a>`标签
 2. 我们为点击事件定义了一个控制器函数。这个函数会在用户每次点击`like/unlike`时执行
 3.在控制器函数中，我们使用`e.preventDefault()`来避免`<a>`标签的默认行为。这会阻止链接把我们带到其他地方。
 4. 我们使用`$.post()`向服务器执行一个异步 POST 请求。 jQuery 也会提供一个`$.get()`方法来执行 GET 请求和一个低级别的 `$.ajax()`方法。
 5. 我们使用 Django 的`{% url %}`模板标签来构建为 AJAX 请求需要的URL
 6. 我们在请求中建立要发送的 POST 参数字典。他们是 Django 视图中期望的 `ID`和`action`参数。我们从`<a>`元素的`data-id`和`data-action`中获取两个参数的值。
 7. 我们定义了一个当 HTTP 应答被接收时的回调函数。它接收一个含有应答内容的数据属性。
 8. 我们获取接收数据的`status`属性然后检查它的值是否是`ok`。如果返回的`data`是期望中的那样，我们将切换`data-action`属性的链接和它的文本内容。这可以让用户取消这个动作。
 9. 我们基于执行的动作来增加或者减少 likes 的总数

在你的浏览器中打开一张你上传的图片的详情页，你可以看到初始的 like 统计和一个 `LIKE` 按钮：
![Django-5-7][7]
点击**`LIKE`**按钮，你将会看见 likes 的总数上升了，按钮的文本也变成了**`UNLIKE`**：
![Django-5-8][8]
当你点击**`UNLIKE`**按钮时动作被执行，按钮的文本也会变成**`LIKE`**，统计的总数也会据此下降。

在编写 JavaScript 时，特别是在写 AJAX 请求时， 我们建议应该使用一个类似于 Firebug 的工具来调试你的 JavaScript 脚本以及监视 CSS 和 HTML 的变化，你可以下载 Firebug ： http://getfirebug.com/。一些浏览器比如*Chrome*或者*Safari*也包含一些调试 JavaScript 的开发者工具。在那些浏览器中，你可以在网页的任何地方右键然后点击**Inspect element**来使用网页开发者工具。

##**为你的视图创建定制化的装饰器**
我们将会限制我们的 AJAX 视图只接收由 AJAX 发起的请求。Django Request 对象提供了一个 `is_ajax()`方法， 这个方法会检查请求是否带有`XMLHttpRequest`,也就是说，会检查这个请求是否是一个 AJAX 请求。这个值被设置在`HTTP_X_REQUESTED_WITH HTTP`头中， 这个头被大多数的由JavaScript库发起的 AJAX 请求包含。

我们将在我们的视图中创建一个装饰器，来检测`HTTP_X_RQUESTED_WITH`头。装饰器是一个可以接收一个函数为参数的函数，并且它可以在不改变作为参数的函数的情况下，拓展此函数的功能。如果装饰器的概念对你来说还很陌生，在继续阅读之前你或许可以看看这个：https:www.python.org/dev/peps/pep-0318/。

由于我们的装饰器将会是通用的，它将被应用到任何视图中，所以我们在我们的项目中将创建一个 `common`Python 包，在 bookmarklet 项目中创建如下路径：

  - common/
  - \__init__.py
  - decorators.py

编辑`decorators.py`,添加如下代码：

```python
from django.http import HttpResponseBadRequest

def ajax_required(f):
    def wrap(request, *args, **kwargs):
            if not request.is_ajax():
                return HttpResponseBadRequest()
            return f(request, *args, **kwargs)
    wrap.__doc__=f.__doc__
    wrap.__name__=f.__name__
    return wrap
```
这是我们定制的`ajax_required`装饰器。它定义一个当请求不是 AJAX 时返回`HttpResponseBadRequest`（HTTP 400）对象的wrap 函数，否则它将返回一个被装饰了的对象。

现在你可以编辑 images 应用的`views.py`，为你的 `image_like` AJAX 视图添加这个装饰器：

```python
from common.decorators import ajax_required

@ajax_required
@login_required
@require_POST
def image_like(request):
    # ...
```
如果你直接在你的浏览器中访问`http://127.0.0.1:8000/images/like/`,你将会得到一个 HTTP 400 的错误。

>如果你发现你正在视图中执行重复的检查，请为你的视图创建装饰器

##**在你的列表视图中添加 AJAX 分页**
我们需要在你的网站中列出所有的被标签的图片。我们将要使用 AJAX 分页来建立一个不受限的滚屏功能。不受限的滚屏是在用户滚动到底部时，自动加载下一页的结果来实现的。

我们将实现一个图片列表视图，这个视图既可以支持标准的浏览器请求，也支持包含分页的 AJAX 请求。当用户首次加载列表页时，我们展示第一页的图片。当用户滚动到底部时，我们用 AJAX 加载下一页的内容，然后将内容加入到页面的底部。

编辑 images 应用的`views.py`，添加以下代码：

```python
from django.http import HttpResponse
from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger

@login_required
def image_list(request):
    images = Image.objects.all()
    paginator = Paginator(images, 8)
    page = request.GET.get('page')
    try:
        images = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer deliver the first page
        images = paginator.page(1)
    except EmptyPage:
        if request.is_ajax():
            # If the request is AJAX and the page is out of range return an empty page
            return HttpResponse('')
        # If page is out of range deliver last page of results
        images = paginator.page(paginator.num_pages)
    if request.is_ajax():
        return render(request,
                      'images/image/list_ajax.html',
                      {'section': 'images', 'images': images})
    return render(request,
                  'images/image/list.html',
                   {'section': 'images', 'images': images})
```
在这个视图中，我们创建一个查询集（QuerySet）来从数据库中获得所有的图片。然后我们创建了一个`Paginator`对象来分页查询结果，每页有八张图片。如果请求的页面超出范围了，我们将会得到一个`EmptyPage`异常，在这种情况下并且请求又是由 AJAX 发起的话，我们将会返回一个空的`HttpResponse`，这将帮助我们在客户端停止 AJAX 分页，我们将会把结果渲染给两个不同的模板：

 1. 对于 AJAX 请求，我们渲染`list_ajax.html`模板。这个模板将只会包含我们请求页面的图片
 2. 对于标准请求： 我们渲染`list.html`模板。这个模板将会继承`base.html`来展示整个页面，并且`list_ajax.html`页面也会被引入在其中。

编辑 images 应用的`urls.py`,添加以下 URL 模式：

```
url(r'^$', views.image_list, name='list'),
```
最后，我们需要创建我们在上面提到的模板。在 images/image/ 下创建一个名为'list_ajax.html'的模板，添加以下代码：

```html
{% load thumbnail %}

{% for image in images %}
    <div class="image">
        <a href="{{ image.get_absolute_url }}">
            {% thumbnail image.image "300x300" crop="100%" as im %}
                <a href="{{ image.get_absolute_url }}">
                    <img src="{{ im.url }}">
                </a>
            {% endthumbnail %}
        </a>
        <div class="info">
            <a href="{{ image.get_absolute_url }}" class="title">{{ image.title }}</a>
        </div>
    </div>
{% endfor %}
```
这个模板将用于展示一组图片，它会作为结果返回给 AJAX 请求。之后，在相同路径下创建'list.html'模板，添加以下代码：

```html
{% extends "base.html" %}

{% block title %}Images bookmarked{% endblock %}

{% block content %}
    <h1>Images bookmarked</h1>
    <div id="image-list">
        {% include "images/image/list_ajax.html" %}
    </div>
{% endblock %}

```
这个列表模板继承了'`base.html`模板。为了避免重复编码，我们引入了`list_ajax.html`模板。这个`listmhtml`模板将会有一段 JavaScript 代码来加载当滚动到底部时的额外页面。
将以下代码添加进`list.html`模板中：

```html
{% block domready %}
    var page = 1;
    var empty_page = false;
    var block_request = false;

    $(window).scroll(function() {
        var margin = $(document).height() - $(window).height() - 200;
        if  ($(window).scrollTop() > margin && empty_page == false && block_request == false) {
		    block_request = true;
		    page += 1;
		    $.get('?page=' + page, function(data) {
		        if(data == '')
		        {
		            empty_page = true;
		        }
		        else {
                    block_request = false;
                    $('#image-list').append(data);
    	        }
            });
    	}
    });
{% endblock %}

```
这段代码实现了不受限的滚屏功能。我们在 `base.html`中定义的`domready`块中引入了 JavaScript 代码，这段代码的工作流程如下：

1. 我们定义了如下几个变量：

   - `page`：保存当前的页码
   - `empt_page`:让我们知道用户是否到了最后一页，然后接收一个空页面。只要接收到了一个空页面，我们就会停止发送额外的 AJAX 请求，因为我们确定此时已经没有结果了。
   - `block_requests`：当有进程中有 AJAX 请求时，阻止额外的请求。
2. 我们使用`$(window).scroll()`来捕获滚动事件，然后我们为此定义了一个控制器函数。
3. 我们计算边框变量来得到文档高度和窗口高度的差值，因为这个差值是用户将要滚动的内容的高度。我们从结果当中减去 200，这样我们就可以在用户接近底部 200pixels 时加载下一页的内容。
4. 我们只在以下两种条件满足时发送 AJAX 请求：没有其他 AJAX 请求被正在被执行时（译者注：就是同时只有一个 AJAX 请求）（`block_request`必须是`false`）,用户也没有到达页面底部（`empty_page`也必须是`false`）。
5. 我们将`block_request`设为`True`来避免滚动时间触发额外的 AJAX 请求，然后我们会在请求下一页时增加一次`page`计数。
6. 我们使用`$.get()`来执行一次 AJAX GET 请求，然后我们在一个叫做`data`的变量中接收 HTML 响应。这里有两种情况。
   - 响应没有内容：我们已经到了结果的末尾，所以这里没有更多的页面来供我们加载。我们把`empty_page`设为`True`来阻止加载更多的 AJAX 请求。
   - 响应含有数据：我们将数据添加到id为 `image-list`的 HTML 元素中，当用户滚动到底部时页面将直接扩展添加的结果。

在浏览器中访问`http://127.0.0.1:8000/images/`,你会看到你之前添加的一组图片，看起来像这样：
![Django-5-9][9]
  [1]: http://ohqrvqrlb.bkt.clouddn.com/django-5-1.png
  [2]: http://ohqrvqrlb.bkt.clouddn.com/django-5-2.png
  [3]: http://ohqrvqrlb.bkt.clouddn.com/django-5-3.png
  [4]: http://ohqrvqrlb.bkt.clouddn.com/django-5-4.png
  [5]: http://ohqrvqrlb.bkt.clouddn.com/django-5-5.png
  [6]: http://ohqrvqrlb.bkt.clouddn.com/django-5-6.png
  [7]: http://ohqrvqrlb.bkt.clouddn.com/django-5-7.png
  [8]: http://ohqrvqrlb.bkt.clouddn.com/django-5-8.png
  [9]: http://ohqrvqrlb.bkt.clouddn.com/django-5-9.png

滚动到底部将会加载下一页。确定你已经使用书签添加了多于 8 张图片，因为我们每一页展示的是 8 张图片。记得使用 Firebug 或者类似的工具来跟踪 AJAX 请求和调试你的 JavaScript 代码。

最后，编辑 `account` 应用中的`base.html`模板，为主菜单添加图片项：

```html
<li {% if section == "images" %}class="selected"{% endif %}><a href="{% url "images:list" %}">Images</a></li>
```
现在你可以从主菜单连接到图片列表了。

##**总结**
在这一章中，我们创建了一个 JavaScript 书签来从其他网站分享图片到我们的网站。你已经用 jQuery 实现了 AJAX 视图，还添加了 AJAX 分页。

在下一章中，将会教你如何创建一个粉丝系统和一个活动流。你将和通用关系、信号、与反规范化打交道。你也将学习如何在 Django  中使用 Redis。

（译者注：这是我翻译的《Django by Example》的第一篇文章，在之后的日子里我将会与 @夜夜月一同翻译这本书。翻译中有不对的地方，还请读者斧正。我是一名在校大学生，目前大二，出于对编程的喜爱而喜欢上 coding，希望能和各位多多交流。）


















