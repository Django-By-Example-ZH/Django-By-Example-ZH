书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

（译者@ucag注：咳咳，第七章终于来了。其实在一月份就翻译完了😂😂但是后来我回老家了，就没发出来。各位久等了～）

（译者@夜夜月注：真羡慕有寒假和暑假的人- -愿你们的寒暑假作业越多越好，粗略的校对了下，精校版本请大家继续等待）

#**第七章**

##**建立一个在线商店**

在上一章，你创建了一个用户跟踪系统和建立了一个用户活跃流。你也学习了 Django 信号是如何工作的，并且把 Redis 融合进了项目中来为图像视图计数。在这一章中，你将学会如何建立一个最基本的在线商店。你将会为你的产品创建目录和使用 Django sessions 实现一个购物车。你也将学习怎样定制上下文处理器（ context processors ）以及用 Celery 来激活动态任务。

在这一章中，你将学会：

 - 创建一个产品目录
 - 使用 Django sessions 建立购物车
 - 管理顾客的订单
 - 用 Celery 发送异步通知

##**创建一个在线商店项目（project）**

我们将从新建一个在线商店项目开始。我们的用户可以浏览产品目录并且可以向购物车中添加商品。最后，他们将清点购物车然后下单。这一章涵盖了在线商店的以下几个功能：

 - 创建产品目录模型（模型），将它们添加到管理站点，创建基本的视图（view）来展示目录
 - 使用 Django sessions 建立一个购物车系统，使用户可以在浏览网站的过程中保存他们选中的商品
 - 创建下单表单和功能
 - 发送一封异步的确认邮件在用户下单的时候

首先，用以下命令来为你的新项目创建一个虚拟环境，然后激活它：

```shell
mkdir env
virtualenv env/myshop
source env/myshop/bin/activate
```

用以下命令在你的虚拟环境中安装 Django :

```shell
pip install django==1.0.6
```

创建一个叫做 `myshop` 的新项目，再创建一个叫做 `shop` 的应用，命令如下：

```shell
django-admin startproject myshop
cd myshop
django-admin startapp shop
```

编辑你项目中的 `settings.py` 文件，像下面这样将你的应用添加到 `INSTALLED_APPS` 中：

```python
INSTALLED_APPS = [
	# ...
	'shop',
]
```

现在你的应用已经在项目中激活。接下来让我们为产品目录定义模型（models）。

##**创建产品目录模型（models）**

我们商店中的目录将会由不同分类的产品组成。每一个产品会有一个名字，一段可选的描述，一张可选的图片，价格，以及库存。 编辑位于`shop`应用中的`models.py`文件，添加以下代码：

```python
from django.db import models

class Category(models.Model):
	 name = models.CharField(max_length=200,
							      db_index=True)
	 slug = models.SlugField(max_length=200,
	                        db_index=True,
							       unique=True)
	 class Meta:
		  ordering = ('name',)
		  verbose_name = 'category'
		  verbose_name_plural = 'categories'
 
    def __str__(self):
        return self.name
        
class Product(models.Model):
    category = models.ForeignKey(Category, 
                                 related_name='products')
    name = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(max_length=200, db_index=True)
    image = models.ImageField(upload_to='products/%Y/%m/%d',
                              blank=True)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField()
    available = models.BooleanField(default=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ('name',)
        index_together = (('id', 'slug'),)

    def __str__(self):
        return self.name
```

这是我们的 `Category` 和 `Product` 模型（models）。`Category` 模型（models）由一个 `name` 字段和一个唯一的 `slug` 字段构成。`Product` 模型（model）：

 - `category`: 这是一个链接向 `Category` 的 `ForeignKey` 。这是个多对一（many-to-one）关系。一个产品可以属于一个分类，一个分类也可包含多个产品。
 - `name`: 这是产品的名字
 - `slug`: 用来为这个产品建立 URL 的 slug
 - `image`: 可选的产品图片
 - `description`: 可选的产品描述
 - `price`: 这是个 `DecimalField`**（译者@ucag注：十进制字段）**。这个字段使用 Python 的 `decimal.Decimal` 元类来保存一个固定精度的十进制数。`max_digits` 属性可用于设定数字的最大值， `decimal_places` 属性用于设置小数位数。
 - `stock`: 这是个 `PositiveIntegerField`**（译者@ucag注：正整数字段）** 来保存这个产品的库存。
 - `available`: 这个布尔值用于展示产品是否可供购买。这使得我们可在目录中使产品废弃或生效。
 - `created`: 当对象被创建时这个字段被保存。
 - `update`: 当对象最后一次被更新时这个字段被保存。

对于 `price` 字段，我们使用 `DecimalField` 而不是 `FloatField` 来避免精度问题。

> 我们总是使用 `DecimalField` 来保存货币值。 `FloatField` 在内部使用 Python 的 `float` 类型。反之， `DecimalField` 使用的是 Python 中的 `Decimal` 类型，使用 `Decimal` 类型可以避免精度问题。

在 `Product` 模型（model）中的 `Meta` 类中，我们使用 `index_together` 元选项来指定 `id` 和 `slug` 字段的共同索引。我们定义这个索引，因为我们准备使用这两个字段来查询产品，两个字段被索引在一起来提高使用双字段查询的效率。

由于我们会在模型（models）中和图片打交道，打开 shell ，用下面的命令安装 Pillow ：

```shell
pip isntall Pillow==2.9.0
```

现在，运行下面的命令来为你的项目创建初始迁移：

```shell
python manage.py makemigrations
```

你将会看到以下输出：

```shell
Migrations for 'shop':
	0001_initial.py:
	  - Create model Category
	  - Create model Product
	  - Alter index_together for product (1 constraint(s))
```

用下面的命令来同步你的数据库：

```shell
python mange.py migrate
```

你将会看到包含下面这一行的输出：

```shell
 Applying shop.0001_initial... OK
```

现在数据库已经和你的模型（models）同步了。

##**注册目录模型（models）到管理站点**

让我们把模型（models）注册到管理站点，这样我们就可以轻松管理产品和产品分类了。编辑 `shop` 应用的 `admin.py` 文件，添加如下代码：

```python
from django.contrib import admin
from .models import Category, Product

class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug']
    prepopulated_fields = {'slug': ('name',)}
admin.site.register(Category, CategoryAdmin)

class ProductAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'price', 'stock', 
                    'available', 'created', 'updated']
    list_filter = ['available', 'created', 'updated']
    list_editable = ['price', 'stock', 'available']
    prepopulated_fields = {'slug': ('name',)}
admin.site.register(Product, ProductAdmin)
```

记住，我们使用 `prepopulated_fields` 属性来指定那些要使用其他字段来自动赋值的字段。正如你以前看到的那样，这样做可以很方便的生成 slugs 。我们在 `ProductAdmin` 类中使用 `list_editable` 属性来设置可被编辑的字段，并且这些字段都在管理站点的列表页被列出。这样可以让你一次编辑多行。任何在 `list_editable` 的字段也必须在 `list_display` 中，因为只有这样被展示的字段才可以被编辑。

现在，使用如下命令为你的站点创建一个超级用户：

```shell
python manage.py createsuperuser
```

使用命令 `python manage.py runserver` 启动开发服务器。 访问 http://127.0.0.1:8000/admin/shop/product/add ,登录你刚才创建的超级用户。在管理站点的交互界面添加一个新的品种和产品。 product 的更改页面如下所示：

![django-7-1](http://ohqrvqrlb.bkt.clouddn.com/django-7-1.png)

##**创建目录视图（views）**

为了展示产品目录， 我们需要创建一个视图（view）来列出所有产品或者是给出的筛选后的产品。编辑 `shop` 应用中的 `views.py` 文件，添加如下代码：

```python
from django.shortcuts import render, get_object_or_404
from .models import Category, Product

def product_list(request, category_slug=None):
    category = None
    categories = Category.objects.all()
    products = Product.objects.filter(available=True)
    if category_slug:
        category = get_object_or_404(Category, slug=category_slug)
        products = products.filter(category=category)
    return render(request, 
                  'shop/product/list.html', 
                  {'category': category,
                  'categories': categories,
                  'products': products})
```

我们只筛选 `available=True` 的查询集来检索可用的产品。我们使用一个可选参数 `category_slug` 通过所给产品类别来有选择性的筛选产品。

我们也需要一个视图来检索和展示单一的产品。把下面的代码添加进去：

```python
def product_detail(request, id, slug):
    product = get_object_or_404(Product, 
                                id=id, 
                                slug=slug, 
                                available=True)
    return render(request, 
                  'shop/product/detail.html', 
                  {'product': product})
```

`product_detail` 视图（view）接收 `id` 和 `slug` 参数来检索 `Product` 实例。我们可以只用 ID 就可以得到这个实例，因为它是一个独一无二的属性。尽管，我们在 URL 中引入了 slug 来建立搜索引擎友好（SEO-friendly）的 URL。

在创建了产品列表和明细视图（views）之后，我们该为它们定义 URL 模式了。在 `shop` 应用的路径下创建一个新的文件，命名为 `urls.py` ，然后添加如下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^$', views.product_list, name='product_list'),
    url(r'^(?P<category_slug>[-\w]+)/$', 
        views.product_list, 
        name='product_list_by_category'),
    url(r'^(?P<id>\d+)/(?P<slug>[-\w]+)/$', 
        views.product_detail, 
        name='product_detail'),
```

这些是我们产品目录的URL模式。 我们为 `product_list` 视图（view）定义了两个不同的 URL 模式。 命名为`product_list` 的模式不带参数调用 `product_list` 视图（view）；命名为 `product_list_bu_category` 的模式向视图（view）函数传递一个 `category_slug` 参数，以便通过给定的产品种类来筛选产品。我们为 `product_detail` 视图（view）添加的模式传递了 `id` 和 `slug` 参数来检索特定的产品。

像这样编辑 `myshop` 项目中的 `urls.py` 文件：

```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^', include('shop.urls', namespace='shop')),
    ]
```

在项目的主要 URL 模式中，我们引入了 `shop` 应用的 URL 模式，并指定了一个命名空间，叫做 `shop`。


现在，编辑 `shop` 应用中的 `models.py` 文件，导入 `reverse()` 函数，然后给 `Category` 模型和 `Product` 模型添加 `get_absolute_url()` 方法：

```python
from django.core.urlresolvers import reverse
# ...
class Category(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse('shop:product_list_by_category',
                        args=[self.slug])
                        
class Product(models.Model):
# ...
    def get_absolute_url(self):
        return reverse('shop:product_detail',
            args=[self.id, self.slug])
```

正如你已经知道的那样， `get_absolute_url()` 是检索一个对象的 URL 约定俗成的方法。这里，我们将使用我们刚刚在 `urls.py` 文件中定义的 URL 模式。

#**创建目录模板（templates）**
现在，我们需要为产品列表和明细视图创建模板（templates）。在 `shop` 应用的路径下创建如下路径和文件：

```shell
templates/
    shop/
        base.html
        product/
            list.html
            detail.html
```

我们需要定义一个基础模板（template），然后在产品列表和明细模板（templates）中继承它。 编辑 `shop/base.html` 模板（template），添加如下代码：

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>{% block title %}My shop{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <a href="/" class="logo">My shop</a>
    </div>
    <div id="subheader">
        <div class="cart">
            Your cart is empty.
        </div>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

这就是我们将为我们的商店应用使用的基础模板（template）。为了引入模板使用的 CSS 和图像，你需要复制这一章示例代码中的静态文件，位于 `shop` 应用中的 `static/` 路径下。把它们复制到你的项目中相同的地方。

编辑 `shop/product/list.html` 模板（template），然后添加如下代码：

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    {% if category %}{{ category.name }}{% else %}Products{% endif %}
{% endblock %}

{% block content %}
    <div id="sidebar">
        <h3>Categories</h3>
        <ul>
            <li {% if not category %}class="selected"{% endif %}>
                <a href="{% url "shop:product_list" %}">All</a>
            </li>
        {% for c in categories %}
            <li {% if category.slug == c.slug %}class="selected"{% endif %}>
                <a href="{{ c.get_absolute_url }}">{{ c.name }}</a>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div id="main" class="product-list">
        <h1>{% if category %}{{ category.name }}{% else %}Products{% endif %}</h1>
        {% for product in products %}
            <div class="item">
                <a href="{{ product.get_absolute_url }}">
                    <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
                </a>
                <a href="{{ product.get_absolute_url }}">{{ product.name }}</a><br>
                ${{ product.price }}
            </div>
        {% endfor %}
    </div>
{% endblock %} 
```

这是产品列表模板（template）。它继承了 `shop/base.html` 并且使用了 `categories` 上下文变量来展示所有在侧边栏里的产品种类，以及 `products` 上下文变量来展示当前页面的产品。相同的模板用于展示所有的可用的产品以及经目录分类筛选后的产品。由于`Product` 模型的 `image` 字段可以为空，我们需要为没有图片的产品提供一个默认图像。这个图片位于我们的静态文件路径下，相对路径为 `img/no_image.png`。

因为我们在使用 `ImageField` 来保存产品图片，我们需要开发服务器来服务上传图片文件。编辑 `myshop` 项目的 `settings.py` 文件，添加以下设置：

```python
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media/')
```

`MEDIA_URL` 是基础 URL，它为用户上传的媒体文件提供服务。`MEDIA_ROOT` 是一个本地路径，媒体文件就在这个路径下，并且是由我们动态的将 `BASE_DIR` 添加到它的前面而得到的。

为了让 Django 给通过开发服务器上传的媒体文件提供服务，编辑`myshop` 中的 `urls.py` 文件，添加如下代码：

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ...
]
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL,
                          document_root=settings.MEDIA_ROOT)
```

记住，我们仅仅在开发中像这样提供静态文件服务。在生产环境下，你不应该用 Django 来服务静态文件。

使用管理站点为你的商店添加几个产品，然后访问 http://127.0.0.1:8000/ 。你可以看到如下的产品列表页：

![django-7-2](http://ohqrvqrlb.bkt.clouddn.com/django-7-2.png)

如果你用管理站点创建了几个产品，并且没有上传任何图片的话，就会显示默认的 `no_img.png` 。

![django-7-3](http://ohqrvqrlb.bkt.clouddn.com/django-7-3.png)

让我们编辑产品明细模板（template）。 编辑 `shop/product/detail.html` 模板（template），添加以下代码：

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    {% if category %}{{ category.title }}{% else %}Products{% endif %}
{% endblock %}

{% block content %}
    <div class="product-detail">
        <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
        <h1>{{ product.name }}</h1>
        <h2><a href="{{ product.category.get_absolute_url }}">{{ product.category }}</a></h2>
        <p class="price">${{ product.price }}</p>
            {{ product.description|linebreaks }}
    </div>
{% endblock %}
```

我们可以调用相关联的产品类别的 `get_absolute_url()` 方法来展示有效的属于同一目录的产品。现在，访问 http://127.0.0.1:8000 ，点击任意产品，查看产品明细页面。看起来像这样：

![django-7-4](http://ohqrvqrlb.bkt.clouddn.com/django-7-4.png)

##**创建购物车**

在创建了产品目录之后，下一步我们要创建一个购物车系统，这个购物车系统可以让用户选中他们想买的商品。购物车允许用户在最终下单之前选中他们想要的物品并且可以在用户浏览网站时暂时保存它们。购物车存在于会话中，所以购物车中的物品会在用户访问期间被保存。

我们将会使用 Django 的会话框架（seesion framework）来保存购物车。在购物车最终被完成或用户下单之前，购物车将会保存在会话中。我们需要为购物车和购物车里的商品创建额外的 Django 模型（models）。

##**使用 Django 会话**

Django 提供了一个会话框架，这个框架支持匿名会话和用户会话。会话框架允许你为任意访问对象保存任何数据。会话数据保存在服务端，并且如果你使用基于 cookies 的会话引擎的话， cookies 会包含 session ID 。会话中间件控制发送和接收 cookies 。默认的会话引擎把会话保存在数据库中，但是正如你一会儿会看到的那样，你也可以选择不同的会话引擎。为了使用会话，你必须确认你项目的 `MIDDLEWARE_CLASSES` 设置中包含了 `django.contrib.sessions.middleware.SessionMiddleware` 。这个中间件负责控制会话，并且是在你使用命令`startproject
`创建项目时被默认添加的。

会话中间件使当前会话在 `request` 对象中可用。你可以用 `request.seesion` 连接当前会话，它的使用方式和 Python 的字典相似。会话字典接收任何默认的可被序列化为 JSON 的 Python 对象。你可以在会话中像这样设置变量：

```python
request.session['foo'] = 'bar'
```

检索会话中的键:

```python
request.session.get('foo')
```

删除会话中已有键：

```python
del request.session['foo']
```

正如你所见，我们像使用 Python 字典一样使用 `request.session` 。

> 当用户登录时，他们的匿名会话将会丢失，然后新的会话将会为认证后的用户创建。如果你在匿名会话中储存了在登录后依然需要被持有的数据，你需要从旧的会话中复制数据到新的会话。

##**会话设置**

你可以使用几种设置来为你的项目配置会话系统。最重要的部分是 `SESSION_ENGINE` .这个设置让你可以配置会话将会在哪里被储存。默认地， Django 用 `django.contrib.sessions` 的 `Sessions` 模型把会话保存在数据库中。

Django 提供了以下几个选择来保存会话数据：

- Database sessions（数据库会话）:会话数据将会被保存在数据库中。这是默认的会话引擎。
- File-based sessions（基于文件的会话）：会话数据保存在文件系统中。
- Cached sessions（缓存会话）:会话数据保存在缓存后端中。你可以使用 `CACHES` 设置来指定一个缓存后端。在缓存系统中保存会话拥有最好的性能。
- Cached sessions（缓存会话）：会话数据储存于缓存后端。你可以使用 `CACHES` 设置来制定一个缓存后端。在缓存系统中储存会话数据会有更好的性能表现。
- Cached database sessions（缓存于数据库中的会话）：会话数据保存于可高速写入的缓存和数据库中。只会在缓存中没有数据时才会从数据库中读取数据。
- Cookie-based sessions（基于 cookie 的会话）：会话数据储存于发送向浏览器的 cookie 中。

>为了得到更好的性能，使用基于缓存的会话引擎（  cach-based session engine）吧。 Django 支持 Mercached ，以及 Redis 的第三方缓存后端和其他的缓存系统。

你可以用其他的设置来定制你的会话。这里有一些和会话有关的重要设置：

- `SESSION_COOKIE_AGE`：cookie 会话保持的时间。以秒为单位。默认值为 1209600 （2 周）。
- `SESSION_COOKIE_DOMAIN`：这是为会话 cookie 使用的域名。把它的值设置为 `.mydomain.com` 来使跨域名 cookie 生效。
- `SESSION_COOKIE_SECURE`：这是一个布尔值。它表示只有在连接为 HTTPS 时 cookie 才会被发送。
- `SESSION_EXPIRE_AT_BROWSER_CLOSE`：这是一个布尔值。它表示会话会在浏览器关闭时就过期。
- `SESSION_SAVE_EVERY_REQUEST`：这是一个布尔值。如果为 `True` ，每一次请求的 session 都将会被储存进数据库中。 session 的过期时间也会每次刷新。

在这个网站你可以看到所有的 session 设置：https://docs.djangoproject.com/en/1.8/ref/settings/#sessions

##**会话过期**

你可以通过 `SESSION_EXPIRE_AT_BROWSER_CLOSE` 选择使用 browser-length 会话或者持久会话。默认的设置是 `False` ，强制把会话的有效期设置为 `  SESSION_COOKIE_AGE` 的值。如果你把 ` SESSION_EXPIRE_AT_BROWSER_CLOSE` 的值设为 `True`，会话将会在用户关闭浏览器时过期，且 `SESSION_COOKIE_AGE` 将不会对此有任何影响。

你可以使用 `request.session` 的 `set_expiry()` 方法来覆写当前会话的有效期。

##**在会话中保存购物车**

我们需要创建一个能序列化为 JSON 的简单结构，这样就可以把购物车中的东西储存在会话中。购物车必须包含以下数据，每个物品的数据都要包含在其中：
 
  - `Product` 实例的 `id`
  - 选择的产品数量
  - 产品的总价格

因为产品的价格可能会变化，我们采取当产品被添加进购物车时同时保存产品价格和产品本身的办法。这样做，我们就可以保持用户在把商品添加进购物车时他们看到的商品价格不变了，即使产品的价格在之后有了变更。

现在，你需要把购物车和会话关联起来。购物车像下面这样工作：

 - 当需要一个购物车时，我们检查顾客是否已经设置了一个会话键（ session key）。如果会话中没有购物车，我们就创建一个新的购物车，然后把它保存在购物车的会话键中。
 - 对于连续的请求，我们在会话键中执行相同的检查和获取购物车内物品的操作。我们在会话中检索购物车的物品和他们在数据库中相关联的 `Product` 对象。

编辑你的项目中 `settings.py`，把以下设置添加进去：

```python
CART_SESSION_ID = 'cart'
```

添加的这个键将会用于我们的会话中来储存购物车。因为 Django 的会话对于每个访问者是独立的**（译者@ucag注：原文为 per-visitor ，没能想出一个和它对应的中文词，根据上下文，我就把这个词翻译为了一个短语）**，我们可以在所有的会话中使用相同的会话键。

让我们创建一个应用来管理我们的购物车。打开终端，然后创建一个新的应用，在项目路径下运行以下命令：

```shell
python manage.py startapp cart
```

然后编辑你添加的项目中的 `settings.py` ，在 `INSTALLED_APPS` 中添加 `cart`：

```python
INSTALLED_APPS = (
    # ...
    'shop',
    'cart',
)
```

在 `cart` 应用路径内创建一个新的文件，命名为 `cart.py` ，把以下代码添加进去：

```python
from decimal import Decimal
from django.conf import settings
from shop.models import Product

class Cart(object):
    def __init__(self, request):
        """
        Initialize the cart.
        """
        self.session = request.session
        cart = self.session.get(settings.CART_SESSION_ID)
        if not cart:
            # save an empty cart in the session
            cart = self.session[settings.CART_SESSION_ID] = {}
        self.cart = cart
```

这个 `Cart` 类可以让我们管理购物车。我们需要把购物车与一个 `request` 对象一同初始化。我们使用 `self.session = request.session` 保存当前会话以便使其对 `Cart`类的其他方法可用。首先，我们使用  `self.session.get(settings.CART_SESSION_ID)` 尝试从当前会话中获取购物车。如果当前会话中没有购物车，我们就在会话中设置一个空字典，这样就可以在会话中设置一个空的购物车。我们希望我们的购物车字典使用产品 ID 作为键，以数量和价格为键值对的字典为值。这样做，我们就能保证一个产品在购物车当中不被重复添加；我们也能简化获取任意购物车物品数据的步骤。

让我们写一个方法来向购物车当中添加产品或者更新产品的数量。把 `save()` 和 `add()` 方法添加进 `Cart` 类当中：

```python
def add(self, product, quantity=1, update_quantity=False):
    """
    Add a product to the cart or update its quantity.
    """
    product_id = str(product.id)
    if product_id not in self.cart:
        self.cart[product_id] = {'quantity': 0,
                                 'price': str(product.price)}
    if update_quantity:
        self.cart[product_id]['quantity'] = quantity
    else:
        self.cart[product_id]['quantity'] += quantity
    self.save()
    
def save(self):
    # update the session cart
    self.session[settings.CART_SESSION_ID] = self.cart
    # mark the session as "modified" to make sure it is saved
    self.session.modified = True
```

`add()` 函数接受以下参数：

- `product`：需要在购物车中更新或者向购物车添加的 `Product` 对象
- `quantity`：一个产品数量的可选参数。默认为 1 
- `update_quantity`：这是一个布尔值，它表示数量是否需要按照给定的数量参数更新（`True`），不然新的数量必须要被加进已存在的数量中（`False`）

我们在购物车字典中把产品 `id` 作为键。我们把产品 `id` 转换为字符串，因为 Django 使用 JSON 来序列化会话数据，而 JSON 又只接受支字符串的键名。产品 `id` 为键，一个有 `quantity` 和 `price` 的字典作为值。产品的价格从十进制数转换为了字符串，这样才能将它序列化。最后，我们调用 `save()` 方法把购物车保存到会话中。

`save()` 方法会把购物车中所有的改动都保存到会话中，然后用 `session.modified = True` 标记改动了的会话。这是为了告诉 Django 会话已经被改动，需要将它保存起来。

我们也需要一个方法来从购物车当中删除购物车。把下面的方法添加进 `Cart` 类当中：

```python
def remove(self, product):
    """
    Remove a product from the cart.
    """
    product_id = str(product.id)
    if product_id in self.cart:
        del self.cart[product_id]
        self.save()
```

`remove` 方法从购物车字典中删除给定的产品，然后调用 `save()` 方法来更新会话中的购物车。

我们将迭代购物车当中的物品，然后获取相应的 `Product` 实例。为恶劣达到我们的目的，你需要定义 `__iter__()` 方法。把下列代码添加进 `Cart` 类中：

```python
def __iter__(self):
    """
    Iterate over the items in the cart and get the products
    from the database.
    """
    product_ids = self.cart.keys()
    # get the product objects and add them to the cart
    products = Product.objects.filter(id__in=product_ids)
    for product in products:
        self.cart[str(product.id)]['product'] = product
        
    for item in self.cart.values():
        item['price'] = Decimal(item['price'])
        item['total_price'] = item['price'] * item['quantity']
        yield item
```

在 `__iter__()` 方法中，我们检索购物车中的 `Product` 实例来把他们添加进购物车的物品中。之后，我们迭代所有的购物车物品，把他们的 `price` 转换回十进制数，然后为每个添加一个 `total_price` 属性。现在我们就可以很容易的在购物车当中迭代物品了。

我们还需要一个方法来返回购物车中物品的总数量。当 `len()` 方法在一个对象上执行时，Python 会调用对象的 `__len__()` 方法来检索它的长度。我们将会定义一个定制的 `__len__()` 方法来返回保存在购物车中保存的所有物品数量。把下面这个 `__len__()` 方法添加进 `Cart` 类中：

```python
def __len__(self):
    """
    Count all items in the cart.
    """
    return sum(item['quantity'] for item in self.cart.values())
```

我们返回所有购物车物品的数量。

添加下列方法来计算购物车中物品的总价：

```python
def get_total_price(self):
    return sum(Decimal(item['price']) * item['quantity'] for item in self.cart.values())
```

最后，添加一个方法来清空购物车会话：

```python
def clear(self):
    # remove cart from session
    del self.session[settings.CART_SESSION_ID]
        self.session.modified = True
```

我们的 `Cart` 类现在已经准备好管理购物车了。

#**创建购物车视图**

既然我们已经创建了 `Cart` 类来管理购物车，我们就需要创建添加，更新，或者删除物品的视图了。我们需要创建以下视图：

- 用于添加或者更新物品的视图，且能够控制当前的和更新的数量
- 从购物车中删除物品的视图
- 展示购物车物品和总数的视图

##**添加物品**

为了把物品添加进购物车，我们需要一个允许用户选择数量的表单。在 `cart` 应用路径下创建一个 `forms.py` 文件，然后添加以下代码：

```python
from django import forms

PRODUCT_QUANTITY_CHOICES = [(i, str(i)) for i in range(1, 21)]

class CartAddProductForm(forms.Form):
    quantity = forms.TypedChoiceField(
                choices=PRODUCT_QUANTITY_CHOICES,
                coerce=int)
    update = forms.BooleanField(required=False,
                initial=False,
                widget=forms.HiddenInput)
```

我们将要使用这个表单来向购物车添加产品。我们的 `CartAddProductForm` 类包含以下两个字段：

- `quantity`：让用户可以在 1~20 之间选择产品的数量。我们使用了带有 `coerce=int` 的 `TypeChoiceField` 字段来把输入转换为整数
- `update`：让你展示数量是否要被加进已当前的产品数量上（`False`），否则如果当前数量必须被用给定的数量给更新（`True`）。我们为这个字段使用了`HiddenInput` 控件，因为我们不想把它展示给用户。

让我们一个新的视图来想购物车中添加物品。编辑 `cart` 应用的 `views.py` ，添加以下代码：

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.views.decorators.http import require_POST
from shop.models import Product
from .cart import Cart
from .forms import CartAddProductForm

@require_POST
def cart_add(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    form = CartAddProductForm(request.POST)
    if form.is_valid():
        cd = form.cleaned_data
        cart.add(product=product,
                quantity=cd['quantity'],
                update_quantity=cd['update'])
    return redirect('cart:cart_detail')
```

这个视图是为了想购物车添加新的产品或者更新当前产品的数量。我们使用 `require_POST` 装饰器来只响应 POST 请求，因为这个视图将会变更数据。这个视图接收产品 ID 作为参数。我们用给定的 ID 来检索 `Product` 实例，然后验证 `CartAddProductForm`。如果表单是合法的，我们将在购物车中添加或者更新产品。我们将创建 `cart_detail` 视图。

我们还需要一个视图来删除购物车中的物品。将以下代码添加进 `cart` 应用的 `views.py` 中：

```python
def cart_remove(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    cart.remove(product)
    return redirect('cart:cart_detail')
```

`cart_detail` 视图接收产品 ID 作为参数。我们根据给定的产品 ID 检索相应的 `Product` 实例，然后将它从购物车中删除。然后，我们将用户重定向到 `cart_detail` URL。

最后，我们需要一个视图来展示购物车和其中的物品。讲一下代码添加进 `veiws.py` 中：

```python
def cart_detail(request):
    cart = Cart(request)
    return render(request, 'cart/detail.html', {'cart': cart})
```

`cart_detail` 视图获取当前购物车并展示它。

我们已经创建了视图来向购物车中添加物品，或从购物车中更新数量，删除物品，还有展示他们。然我们为这些视图添加 URL 模式。在 `cart` 应用中创建一个新的文件，命名为 `urls.py`。把下面这些 URL 模式添加进去：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^$', views.cart_detail, name='cart_detail'),
    url(r'^add/(?P<product_id>\d+)/$',
            views.cart_add,
            name='cart_add'),
    url(r'^remove/(?P<product_id>\d+)/$',
            views.cart_remove,
            name='cart_remove'),
]
```

编辑 `myshop` 应用的主 `urls.py` 文件，添加以下 URL 模式来引用 `cart` URLs：

```python
urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^cart/', include('cart.urls', namespace='cart')),
    url(r'^', include('shop.urls', namespace='shop')),
]
```

确保你在 `shop.urls` 之前引用它，因为它比前者更加有限制性。

##**创建展示购物车的模板**

`cart_add` 和 `cart_remove` 视图没有渲染任何模板，但是我们需要为 `cart_detail` 创建模板。

在 `cart` 应用路径下创建以下文件结构：

```
templates/
    cart/
        detail.html
```

编辑 `cart/detail.html` 模板，然后添加以下代码：

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    Your shopping cart
{% endblock %}

{% block content %}
    <h1>Your shopping cart</h1>
    <table class="cart">
        <thead>
            <tr>
                <th>Image</th>
                <th>Product</th>
                <th>Quantity</th>
                <th>Remove</th>
                <th>Unit price</th>                
                <th>Price</th>
            </tr>
        </thead>
        <tbody>
        {% for item in cart %}
            {% with product=item.product %}
            <tr>
                <td>
                    <a href="{{ product.get_absolute_url }}">
                        <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
                    </a>
                </td>
                <td>{{ product.name }}</td>
                <td>{{ item.quantity }}</td>
                <td><a href="{% url "cart:cart_remove" product.id %}">Remove</a></td>
                <td class="num">${{ item.price }}</td>
                <td class="num">${{ item.total_price }}</td>
            </tr>
            {% endwith %}
        {% endfor %}
        <tr class="total">
            <td>Total</td>
            <td colspan="4"></td>
            <td class="num">${{ cart.get_total_price }}</td>
        </tr>
        </tbody>
    </table>
    <p class="text-right">
        <a href="{% url "shop:product_list" %}" class="button light">Continue shopping</a>
        <a href="#" class="button">Checkout</a>
    </p>
{% endblock %}
```

这个模板被用于展示购物车的内容。它包含了一个保存于当前购物车物品的表格。我们允许用用户使用发送到 `cart_add` 表单来改变选中的产品数量。我们通过提供一个 *Remove* 链接来允许用户从购物车中删除物品。

##**向购物车中添加物品**

现在，我们需要在产品详情页添加一个 **Add to cart** 按钮。编辑 `shop` 应用中的 `views.py`，然后把 `CartAddProductForm` 添加进 `product_detail` 视图中：

```python
from cart.forms import CartAddProductForm

def product_detail(request, id, slug):
    product = get_object_or_404(Product, id=id,
                                slug=slug,
                                available=True)
    cart_product_form = CartAddProductForm()
    return render(request,
            'shop/product/detail.html',
            {'product': product,
            'cart_product_form': cart_product_form})
```

编辑 `shop` 应用的 `shop/product/detail.html` 模板，然后将如下表格按照这样添加产品价格：

```html
<p class="price">${{ product.price }}</p>
<form action="{% url "cart:cart_add" product.id %}" method="post">
{{ cart_product_form }}
{% csrf_token %}
<input type="submit" value="Add to cart">
</form>
```

确保用 `python manage.py runserver` 运行开发服务器。现在，打开 http://127.0.0.1:8000/，导航到产品详情页。现在它包含了一个表单来选择数量在将产品添加进购物车之前。这个页面看起来像这样：

![django-7-5](http://ohqrvqrlb.bkt.clouddn.com/django-7-5.png)

选择一个数量，然后点击 **Add to cart** 按钮。表单将会通过 POST 方法提交到 `cart_add` 视图。视图会把产品添加进当前会话的购物车当中，包括当前产品的价格和选定的数量。然后，用户将会被重定向到购物车详情页，它长得像这个样子：

![django-7-6](http://ohqrvqrlb.bkt.clouddn.com/django-7-6.png)

##**在购物车中更新产品数量**

当用户看到购物车时，他们可能想要在下单之前改变产品数量。我们将会允许用户在详情页改变产品数量。

编辑 `cart` 应用的 `views.py`，然后把 `cart_detail` 改成这个样子：

```python
def cart_detail(request):
    cart = Cart(request)
    for item in cart:
        item['update_quantity_form'] = CartAddProductForm(
                                    initial={'quantity': item['quantity'],
                                    'update': True})
    return render(request, 'cart/detail.html', {'cart': cart})
```

我们为每一个购物车中的物品创建了 `CartAddProductForm` 实例来允许用户改变产品的数量。我们把表单和当前物品数量一同初始化，然后把 `update` 字段设为 `True` ，这样当我们提交表单到 `cart_add` 视图时，当前的数量就被新的数量替换了。

现在，编辑 `cart` 应用的 `cart/detail.html` 模板，然后找到这一行：

```html
<td> {{ item.quantity }} </td>
```

把它替换为下面这样的代码：

```html
<td>
<form action="{% url "cart:cart_add" product.id %}" method="post">
{{ item.update_quantity_form.quantity }}
{{ item.update_quantity_form.update }}
<input type="submit" value="Update">
{% csrf_token %}
</form>
</td>
```

在你的浏览器中打开 http://127.0.0.1:8000/cart/ 。你将会看到一个表单来编辑每个物品的数量，长得像下面这样：

![django-7-7](http://ohqrvqrlb.bkt.clouddn.com/django-7-7.png)

改变物品的数量，然后点击 **Update** 按钮来测试新的功能。

#**为当前购物车创建上下文处理器**

你可能已经注意到我们在网站的头部展示了 **Your cart is empty** 的信息。当我们开始向购物车添加物品时，我们将看到它已经替换为了购物车中物品的总数和总花费。由于这是个展示在整个页面的东西，我们将创建一个上下文处理器来引用当前请求中的购物车，尽管我们的视图函数已经处理了它。

##**上下文处理器**

上下文处理器是一个接收 `request` 对象为参数并返回一个已经添加了请求上下文字典的 Python 函数。他们在你需要让什么东西在所有模板都可用时迟早会派上用场。

一般的，当你用 `startproject` 命令创建一个新的项目时，你的项目将会包含下面的模板上下文处理器，他们位于 `TEMPLATES` 设置中的 `context_processors` 内：

- `django.template.context_processors.debug`：在上下文中设置 `debug` 布尔值和 `sql_queries` 变量，来表示在 request 中执行的 SQL 查询语句表
- `django.template.context_processors.request`：在上下文中设置 request 变量
- `django.contrib.auth.context_processors.auth`：在请求中设置用户变量
- `django.contrib.messages.context_processors.messages`：在包含所有使用消息框架发送的信息的上下文中设置一个 `messages` 变量

Django 也使用 `django.template.context_processors.csrf` 来避免跨站请求攻击。这个上下文处理器不在设置中，但是它总是可用的并且由安全原因不可被关闭。

你可以在这个网站看到所有的内建上下文处理器：https://docs.djangoproject.com/en/1.8/ref/templates/api/#built-in-template-context-processors

##**把购物车添加进请求上下文中**

让我们创建一个上下文处理器来将当前购物车添加进模板请求上下文中。这样我们就可以在任意模板中获取任意购物车了。

在 `cart` 应用路径里添加一个新文件，并命名为 `context_processors.py` 。上下文处理器可以位于你代码中的任何地方，但是在这里创建他们将会使你的代码变得组织有序。将以下代码添加进去：

```python
from .cart import Cart

def cart(request):
    return {'cart': Cart(request)}
```

如你所见，一个上下文处理器是一个函数，这个函数接收一个 `request` 对象作为参数，然后返回一个对象字典，这些对象可用于所有使用 `RequestContext` 渲染的模板。在我们的上下文处理器中，我们使用 `request` 对象实例化了购物车，然后让它作为一个名为 `cart` 的参数对模板可用。

编辑项目中的 `settings.py` ，然后把 `cart.context_processors.cart` 添加进 `TEMPLATE` 内的 `context_processors` 选项中。改变后的设置如下：

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'cart.context_processors.cart',
            ],
        },
    },
]
```

你的上下文处理器将会在使用 `RequestContext` 渲染 模板时执行。 `cart` 变量将会被设置在模板上下文中。

> 上下文处理器会在所有的使用 `RequestContext` 的请求中执行。你可能想要创建一个定制的模板标签来代替一个上下文处理器，如果你想要链接到数据库的话。

现在，编辑 `shop`应用的 `shop/base.html` 模板，然后找到这一行：

```html
<div class="cart">
Your cart is empty.
</div>
```

把它替换为下面的代码：

```html
<div class="cart">
    {% with total_items=cart|length %}
        {% if cart|length > 0 %}
            Your cart:
            <a href="{% url "cart:cart_detail" %}">
                {{ total_items }} item{{ total_items|pluralize }},
                ${{ cart.get_total_price }}
            </a>
        {% else %}
            Your cart is empty.
        {% endif %}
    {% endwith %}
</div>
```

使用 `python manage.py runserver` 重载你的服务器。打开 http://127.0.0.1:8000/ ,添加一些产品到购物车里。在网站头里，你可以看到当前物品总数和总花费，就象这样：

![django-7-8](http://ohqrvqrlb.bkt.clouddn.com/django-7-8.png)

#**保存用户订单**

当购物车已经结账完毕时，你需要把订单保存进数据库中。订单将要保存客户信息和他们购买的产品信息。

使用下面的命令创建一个新的应用来管理用户订单：

```shell
python manage.py startapp orders
```

编辑项目中的 `settings.py` ，然后把 `orders` 添加进 `INSTALLED_APPS` 中：

```python
INSTALLED_APPS = (
    # ...
    'orders',
)
```

现在你已经激活了你的新应用。

#**创建订单模型**

你需要一个模型来保存订单的详细信息，第二个模型用来保存购买的物品，包括物品的价格和数量。编辑 `orders` 应用的 `models.py` ，然后添加以下代码：

```python
from django.db import models
from shop.models import Product

class Order(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    email = models.EmailField()
    address = models.CharField(max_length=250)
    postal_code = models.CharField(max_length=20)
    city = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    paid = models.BooleanField(default=False)
    
    class Meta:
        ordering = ('-created',)
        
    def __str__(self):
        return 'Order {}'.format(self.id)
        
    def get_total_cost(self):
        return sum(item.get_cost() for item in self.items.all())
        
        
class OrderItem(models.Model):
    order = models.ForeignKey(Order, related_name='items')
    product = models.ForeignKey(Product,
                    related_name='order_items')
    price = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.PositiveIntegerField(default=1)
    
    def __str__(self):
        return '{}'.format(self.id)
        
    def get_cost(self):
        return self.price * self.quantity
```

`Order` 模型包含几个用户信息的字段和一个 `paid` 布尔值字段，这个字段默认值为 `False` 。待会儿，我们将使用这个字段来区分支付和未支付订单。我们也定义了一个 `get_total_cost()` 方法来得到订单中购买物品的总花费。

`OrderItem` 模型让我们可以保存物品，数量和每个物品的支付价格。我们引用 `get_cost()` 来返回物品的花费。

给 `orders` 应用下运行首次迁移：

```shell
python manage.py makemigrations
```

你将看到如下输出：

```shell
Migrations for 'orders':
    0001_initial.py:
        - Create model Order
        - Create model OrderItem
```

运行以下命令来应用新的迁移：

```shell
python manage.py migrate
```

你的订单模型已经同步到了数据库中

##**在管理站点引用订单模型**

让我们把订单模型添加到管理站点。编辑 `orders` 应用的 `admin.py`：

```python
from django.contrib import admin
from .models import Order, OrderItem

class OrderItemInline(admin.TabularInline):
    model = OrderItem
    raw_id_fields = ['product']
    
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'first_name', 'last_name', 'email',
                    'address', 'postal_code', 'city', 'paid',
                    'created', 'updated']
    list_filter = ['paid', 'created', 'updated']
    inlines = [OrderItemInline]
    
admin.site.register(Order, OrderAdmin)
```

我们在  `OrderItem` 使用 `ModelInline` 来把它引用为  `OrderAdmin` 类的内联元素。一个内联元素允许你在同一编辑页引用模型，并且将这个模型作为父模型。

用 `python manage.py runserver` 命令打开开发服务器，访问 http://127.0.1:8000/admin/orders/order/add/ 。你将会看到如下页面：

![django-7-9](http://ohqrvqrlb.bkt.clouddn.com/django-7-9.png)

##**创建顾客订单**

我们需要使用订单模型来保存在用户最终下单时在购物车中的物品，创建新的订单的工作流程如下：

* 1. 向用户展示一个订单表来让他们填写数据
* 2. 我们用用户输入的数据创建一个新的 `Order` 实例，然后我们创建每个物品相关联的 `OrderItem` 实例。
* 3. 我们清空购物车，然后把用户重定向到成功页面

首先，我们需要一个表单来输入订单详情。在`orders` 应用路径内创建一个新的文件，命名为 `forms.py` 。添加以下代码：

```python
from django import forms
from .models import Order

class OrderCreateForm(forms.ModelForm):
    class Meta:
        model = Order
        fields = ['first_name', 'last_name', 'email', 'address',
                'postal_code', 'city']
```
这是我们将要用于创建新的 `Order` 对象的表单。现在，我们需要一个视图来管理表格以及创建一个新的订单。编辑`orders`应用的 `views.py` ，添加以下代码：

```python
from django.shortcuts import render
from .models import OrderItem
from .forms import OrderCreateForm
from cart.cart import Cart

def order_create(request):
    cart = Cart(request)
    if request.method == 'POST':
        form = OrderCreateForm(request.POST)
        if form.is_valid():
            order = form.save()
            for item in cart:
                OrderItem.objects.create(order=order,
                    product=item['product'],
                    price=item['price'],
                    quantity=item['quantity'])
            # clear the cart
            cart.clear()
            return render(request,
                'orders/order/created.html',
                {'order': order})
    else:
        form = OrderCreateForm()
    return render(request,
            'orders/order/create.html',
            {'cart': cart, 'form': form})
```

在 `order_create` 视图中，我们将用 `cart = Cart(request)` 获取到当前会话中的购物车。基于请求方法，我们将执行以下几个任务：

- **GET 请求**：实例化 `OrderCreateForm` 表单然后渲染模板 `orders/order/create.html`
- **POST 请求**：验证提交的数据。如果数据是合法的，我们将使用 `order = form.save()` 来创建一个新的 `Order` 实例。然后我们将会把它保存进数据库中，之后再把它保存进 `order` 变量里。在创建 `order` 之后，我们将迭代无购车的物品然后为每个物品创建 `OrderItem`。最后，我们清空购物车。

现在，在 `orders` 应用路径下创建一个新的文件，把它命名为 `urls.py`。添加以下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
        url(r'^create/$',
            views.order_create,
            name='order_create'),
]
```

这个是 `order_create` 视图的 URL 模式。编辑 `myshop` 的 `urls.py` ，把下面的模式引用进去。记得要把它放在 `shop.urls` 模式之前：

```
url(r'^orders/', include('orders.urls', namespace='orders')),
```

编辑 `cart` 应用的 `cart/detail.html` 模板，找到下面这一行：

```html
<a href="#" class="button">Checkout</a>
```

替换为：

```html
<a href="{% url "orders:order_create" %}" class="button">
Checkout
</a>
```

用户现在可以从购物车详情页导航到订单表了。我们依然需要定义一个下单模板。在 `orders` 应用路径下创建如下文件结构：

```
templates/
    orders/
        order/
            create.html
            created.html
```

编辑 `ordrs/order/create.html` 模板，添加以下代码：

```html
{% extends "shop/base.html" %}

{% block title %}
Checkout
{% endblock %}

{% block content %}
    <h1>Checkout</h1>
    
    <div class="order-info">
        <h3>Your order</h3>
        <ul>
            {% for item in cart %}
            <li>
                {{ item.quantity }}x {{ item.product.name }}
                <span>${{ item.total_price }}</span>
            </li>
            {% endfor %}
        </ul>
    <p>Total: ${{ cart.get_total_price }}</p>
</div>

<form action="." method="post" class="order-form">
    {{ form.as_p }}
    <p><input type="submit" value="Place order"></p>
    {% csrf_token %}
</form>
{% endblock %}
```

模板展示的购物车物品包括物品总量和下单表。

编辑 ` orders/order/created.html` 模板，然后添加以下代码：

```html
{% extends "shop/base.html" %}

{% block title %}
Thank you
{% endblock %}

{% block content %}
    <h1>Thank you</h1>
    <p>Your order has been successfully completed. Your order number is
<strong>{{ order.id }}</strong>.</p>
{% endblock %}
```

这是当订单成功创建时我们渲染的模板。打开开发服务器，访问 http://127.0.0.1:8000/ ，在购物车当中添加几个产品进去，然后结账。
你就会看到下面这个页面：

![django-7-10](http://ohqrvqrlb.bkt.clouddn.com/django-7-10.png)

用合法的数据填写表单，然后点击 **Place order** 按钮。订单就会被创建，然后你将会看到成功页面：

![django-7-11](http://ohqrvqrlb.bkt.clouddn.com/django-7-11.png)

#**使用 Celery 执行异步操作**

你在视图执行的每个操作都会影响响应的时间。在很多场景下你可能想要尽快的给用户返回响应，并且让服务器异步地执行一些操作。这特别和耗时进程或从属于失败的进程时需要重新操作时有着密不可分的关系。比如，一个视频分享平台允许用户上传视频但是需要相当长的时间来转码上传的视频。这个网站可能会返回一个响应给用户，告诉他们转码即将开始，然后开始异步转码。另一个例子是给用户发送邮件。如果你的网站发送了通知邮件，SMTP 连接可能会失败或者减慢响应的速度。执行异步操作来避免阻塞执行就变得必要起来。

Celery 是一个分发队列，它可以处理大量的信息。它既可以执行实时操作也支持任务调度。使用 Celery 不仅可以让你很轻松的创建异步任务还可以让这些任务尽快执行，但是你需要在一个指定的时间调度他们执行。

你可以在这个网站找到 Celery 的官方文档：http://celery.readthedocs.org/en/latest/

##**安装 Celery**
让我们安装 Celery 然后把它整合进你的项目中。用下面的命令安装 Celery：

```shell
pip install celery==3.1.18
```

Celery 需要一个消息代理（message broker）来管理请求。这个代理负责向 Celery 的 worker 发送消息，当接收到消息时 worker 就会执行任务。让我们安装一个消息代理。

##**安装 RabbitMQ**

有几个 Celery 的消息代理可供选择，包括键值对储存，比如 Redis 或者是实时消息系统，比如 RabbitMQ。我们会用 RabbitMQ 配置 Celery ，因为它是 Celery 推荐的 message worker。

如果你用的是 Linux，你可以用下面这个命令安装 RabbitMQ ：

```shell
apt-get install rabbitmg
```
**（译者@夜夜月注：这是debian系linux的安装方式）**

如果你需要在 Mac OSX 或者 Windows 上安装 RabbitMQ，你可以在这个网站找到独立的支持版本：
https://www.rabbitmq.com/download.html

在安装它之后，使用下面的命令执行 RabbitMQ：

```shell
rabbitmg-server
```

你将会在最后一行看到这样的输出：

```shell
Starting broker... completed with 10 plugins
```

RabbitMQ 正在运行了，准备接收消息。

##**把 Celery 添加进你的项目**

你必须为 Celery 实例提供配置。在 `myshop` 的 `settings.py` 文件的旁边创建一个新的文件，命名为 `celery.py` 。这个文件会包含你项目的 Celery 配置。添加以下代码：

```python
import os
from celery import Celery
from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myshop.settings')

app = Celery('myshop')

app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)
```

在这段代码中，我们为 Celery 命令行程序设置了 `DJANGO_SETTINGS_MODULE` 变量。然后我们用  `app=Celery('myshop')` 创建了一个实例。我们用 `config_from_object()` 方法来加载项目设置中任意的定制化配置。最后，我们告诉 Celery 自动查找我们列举在  `INSTALLED_APPS` 设置中的异步应用任务。Celery 将在每个应用路径下查找 `task.py` 来加载定义在其中的异步任务。

你需要在你项目中的 `__init__.py` 文件中导入 `celery`来确保在 Django 开始的时候就会被加载。编辑 `myshop/__init__.py` 然后添加以下代码：

```python
# import celery
from .celery import app as celery_app
```

现在，你可以为你的项目开始编写异步任务了。

>`CELERY_ALWAYS_EAGER` 设置允许你在本地用异步的方式执行任务而不是把他们发送向队列中。这在不运行 Celery 的情况下， 运行单元测试或者是运行在本地环境中的项目是很有用的。

##**向你的应用中添加异步任务**

我们将创建一个异步任务来发送消息邮件来让用户知道他们下单了。

约定俗成的一般用法是，在你的应用路径下的 `tasks` 模型里引入你应用的异步任务。在 `orders` 应用内创建一个新的文件，并命名为 `task.py` 。这是 Celery 寻找异步任务的地方。添加以下代码：

```python
from celery import task
from django.core.mail import send_mail
from .models import Order

@task
def order_created(order_id):
    """
    Task to send an e-mail notification when an order is
    successfully created.
    """
    order = Order.objects.get(id=order_id)
    subject = 'Order nr. {}'.format(order.id)
    message = 'Dear {},\n\nYou have successfully placed an order.\
                Your order id is {}.'.format(order.first_name,
                                            order.id)
    mail_sent = send_mail(subject,
                        message,
                        'admin@myshop.com',
                        [order.email])
    return mail_sent
```

我们通过使用 `task` 装饰器来定义我们的 `order_created` 任务。如你所见，一个 Celery 任务 只是一个用 `task` 装饰的 Python 函数。我们的 `task` 函数接收一个 `order_id` 参数。通常推荐的做法是只传递 ID 给任务函数然后在任务被执行的时候需找相关的对象，我们使用 Django 提供的 `send_mail()` 函数来发送一封提示邮件给用户告诉他们下单了。如果你不想安装邮件设置，你可以通过一下 `settings`.py` 中的设置告诉 Django 把邮件传给控制台：

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

> 异步任务不仅仅适用于耗时进程，也适用于失败进程组中的进程，这些进程或许不会消耗太多时间，但是他们或许会链接失败或者需要再次尝试连接策略。

现在我们要把任务添加到 `order_create` 视图中。打开 `orders` 应用的 `views.py` 文件，按照如下导入任务：

```python
from .tasks import order_created
```

然后在清除购物车之后调用 `order_created` 异步任务：

```python
# clear the cart
cart.clear()
# launch asynchronous task
order_created.delay(order.id)
```

我们调用任务的 `delay()` 方法并异步地执行它。之后任务将会被添加进队列中，将会尽快被一个 worker 执行。

打开另外一个 shell ，使用以下命令开启 celery worker ：

```shell
celery -A myshop worker -1 info
```

Celery worker 现在已经运行，准备好执行任务了。确保 Django 的开发服务器也在运行当中。访问 http://127.0.0.1:8000/ ,在购物车中添加一些商品，然后完成一个订单。在 shell 中，你已经打开过了 Celery worker 所以你可以看到以下的相似输出：

```shell
[2015-09-14 19:43:47,526: INFO/MainProcess] Received task: orders.
tasks.order_created[933e383c-095e-4cbd-b909-70c07e6a2ddf]
[2015-09-14 19:43:50,851: INFO/MainProcess] Task orders.tasks.
order_created[933e383c-095e-4cbd-b909-70c07e6a2ddf] succeeded in
3.318835098994896s: 1
```

任务已经被执行了，你会接收到一封订单通知邮件。

##**监控 Celery**

你或许想要监控执行了的异步任务。下面就是一个基于 web 的监控 Celery 的工具。你可以用下面的命令安装 Flower:

```shell
pip install flower
```

安装之后，你可以在你的项目路径下用以下命令启动 Flower ：

```shell
celery -A myshop flower
```

在你的浏览器中访问 http://localhost:555/dashboard ，你可以看到激活了的 Celery worker 和正在执行的异步任务统计：

![django-7-12](http://ohqrvqrlb.bkt.clouddn.com/django-7-12.png)

你可以在这个网站找到 Flower 的文档：http://flower.readthedocs.org/en/latest/

#**总结**
在这一章中，你创建了一个最基本的商店应用。你创建了产品目录以及使用会话的购物车。你实现了定制化的上下文处理器来使购物车在你的模板中可用，实现了一个下单表格。你也学到了如何用 Celery 执行异步任务。

在下一章中，你将会学习在你的商店中整合一个支付网关，添加管理站点的定制化动作，以 CSV 的形式导出数据，以及动态的生成 PDF 文件。

（译者@夜夜月注：接下来就是第八章了，请大家等待，正在进行中= =，不确定啥时候放出来= =，我是懒人）

















