书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**（译者@ucag 注：哈哈哈，第九章终于来啦。这是在线商店的最后一章，下一章将会开始一个新的项目。所以这一章的内容会偏难，最难的是推荐引擎的编写，其中的算法可能还需要各位好好的推敲，如果看了中文的看不懂，大家可以去看看英文原版的书以及相关的文档。最近我也在研究机器学习，有兴趣的大家可以一起交流哈~）**

**（审校@夜夜月：大家好，我是来打酱油的~，粗校，主要更正了一些错字和格式，精校正进行到四章样子。）**

#**第九章**
##**拓展你的商店**
在上一章中，你学习了如何把支付网关整合进你的商店。你处理了支付通知，学会了如何生成 CSV 和 PDF 文件。在这一章中，你会把优惠券添加进你的商店中。你将学到国际化（internationalization）和本地化（localization）是如何工作的，你还会创建一个推荐引擎。
在这一章中将会包含一下知识点：

* 创建一个优惠券系统来应用折扣
* 把国际化添加进你的项目中
* 使用 Rosetta 来管理翻译
* 使用 django-parler 来翻译模型（model）
* 建立一个产品推荐引擎

##**创建一个优惠券系统**
很多的在线商店会送出很多优惠券，这些优惠券可以在顾客的采购中兑换为相应的折扣。在线优惠券通常是由一串给顾客的代码构成，这串代码在一个特定的时间段内是有效的。这串代码可以被兑换一次或者多次。

我们将会为我们的商店创建一个优惠券系统。优惠券将会在顾客在某一个特定的时间段内输入时生效。优惠券没有任何兑换数的限制，他们也可用于购物车的总金额中。对于这个功能，我们将会创建一个模型（model）来储存优惠券代码，优惠券有效的时间段，以及折扣的力度。

在 `myshop` 项目内使用如下命令创建一个新的应用：

```
python manage.py startapp coupons
```

编辑 `myshop` 的 `settings.py` 文件，像下面这样把把应用添加到 `INSTALLED_APPS` 中：

```
INSTALLED_APPS = (
	  # ...
	  'coupons',
)
```
新的应用已经在我们的 Django 项目中激活了。

##**创建优惠券模型（model）**
让我们开始创建 `Coupon` 模型（model）。编辑 `coupons` 应用中的 `models.py` 文件，添加以下代码：

```python
from django.db import models
from django.core.validators import MinValueValidator,\
									MaxValueValidator
class Coupon(models.Model):
	code = models.CharField(max_length=50,
							unique=True)
	valid_from = models.DateTimeField()
	valid_to = models.DateTimeField()
	discount = models.IntegerField(
				validators=[MinValueValidator(0),
							MaxValueValidator(100)])
	active = models.BooleanField()
	
	def __str__(self):
		return self.code
```

我们将会用这个模型（model）来储存优惠券。 `Coupon` 模型（model）包含以下几个字段：

* `code`：用户必须要输入的代码来将优惠券应用到他们购买的商品中
* `valid_from`：表示优惠券会在何时生效的时间和日期值
* `valid_to`：表示优惠券会在何时过期
* `discount`：折扣率（这是一个百分比，所以它的值的范围是 0 到 1000）。我们使用验证器来限制接收的最小值和最大值
* `active`：表示优惠券是否激活的布尔值

执行下面的命令来为 `coupons` 生成首次迁移：

```
python manage.py makemigrations
```

输出应该包含以下这几行：

```
Migrations for 'coupons':
	0001_initial.py:
		- Create model Coupon
```

之后我们执行下面的命令来应用迁移：

```
python manage.py migrate
```

你可以看见包含下面这一行的输出：

```
Applying coupons.0001_initial... OK
```

迁移现在已经被应用到了数据库中。让我们把 `Coupon` 模型（model）添加到管理站点。编辑 `coupons` 应用的 `admin.py` 文件，添加以下代码：

```python
from django.contrib import admin
from .models improt Coupon

class CouponAdmin(admin.ModelAdmin):
	list_display = ['code', 'valid_from', 'valid_to',
					'discount', 'active']
	list_filter = ['active', 'valid_from', 'valid_to']
	search_fields = ['code']
admin.site.register(Coupon, CouponAdmin)
```

`Coupon` 模型（model）现在已经注册进了管理站点中。确保你已经用命令 `python manage.py runserver` 打开了开发服务器。访问 http://127.0.0.1:8000/admin/coupons/add 。你可以看见下面的表单：

![django-9-1](http://ohqrvqrlb.bkt.clouddn.com/django-9-1.png)

填写表单创建一个在当前日期有效的新优惠券，确保你点击了 **Active** 复选框，然后点击 **Save**按钮。

##**把应用优惠券到购物车中**
我们可以保存新的优惠券以及检索目前的优惠券。现在我们需要一个方法来让顾客可以应用他们的优惠券到他们购买的产品中。花点时间来想想这个功能该如何实现。应用一张优惠券的流程如下：

	1. 用户将产品添加进购物车
	2. 用户在购物车详情页的表单中输入优惠代码
	3. 当用户输入优惠代码然后提交表单时，我们查找一张和所给优惠代码相符的有效优惠券。我们必须检查用户输入的优惠券代码， `active` 属性为 `True` ，当前时间位于 `valid_from 和 `valid_to` 之间。
	4. 如果查找到了相应的优惠券，我们把它保存在用户会话中，然后展示包含折扣了的购物车以及更新总价。
	5. 当用户下单时，我们把优惠券保存到所给的订单中。

在 `coupons` 应用路径下创建一个新的文件，命名为 `forms.py` 文件，添加以下代码：

```python
from django import forms

class CouponApplyForm(forms.Form):
	code = forms.CharField()
```
	 
我们将会用这个表格来让用户输入优惠券代码。编辑 `coupons` 应用中的 `views.py` 文件，添加以下代码：

```python
from django.shortcuts import render, redirect
from django.utils import timezone
from django.views.decorators.http import require_POST
from .models import Coupon
from .forms import CouponApplyForm

@require_POST
def coupon_apply(request):
	now = timezone.now()
	form = CouponApplyForm(request.POST)
	if form.is_valid():
		code = form.cleaned_data['code']
		try:
			coupon = Coupon.objects.get(code__iexact=code,
									valid_from__lte=now,
									valid_to__gte=now,
									active=True)
			request.session['coupon_id'] = coupon.id
		except Coupon.DoesNotExist:
			request.session['coupon_id'] = None
	return redirect('cart:cart_detail')
```	 
	 
`coupon_apply` 视图（view）验证优惠券然后把它保存在用户会话（session）中。我们使用 `require_POST` 装饰器来限制这个视图（view）仅接受 POST 请求。在视图（view）中，我们执行了以下几个任务：

	1.我们用上传的数据实例化了 `CouponApplyForm` 然后检查表单是否合法。
	2. 如果表单是合法的，我们就从表单的 `cleaned_data` 字典中获取 `code` 。我们尝试用所给的代码检索 `Coupon` 对象。我们使用 `iexact` 字段来对照查找大小写不敏感的精确匹配项。优惠券在当前必须是激活的（`active=True`）以及必须在当前日期内是有效的。我们使用 Django 的 `timezone.now()` 函数来获得当前的时区识别时间和日期（time-zone-aware） 然后我们把它和 `valid_from` 和 `valid_to` 字段做比较，对这两个日期分别执行 `lte` （小于等于）运算和 `gte` （大于等于）运算来进行字段查找。
	3. 我们在用户的会话中保存优惠券的 `id``。
	4. 我们把用户重定向到 `cart_detail` URL 来展示应用了优惠券的购物车。

我们需要一个 `coupon_apply` 视图（view）的 URL 模式。在 `coupon` 应用路径下创建一个新的文件，命名为 `urls.py` ，添加以下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
	url(r'^apply/$', views.coupon_apply, name='apply'),
]
```

然后，编辑 `myshop` 项目的主 `urls.py` 文件，引入 `coupons` 的 URL 模式：

```python
url(r'^coupons/', include('coupons.urls', namespace='coupons')),
``` 

记得把这个放在 `shop.urls` 模式之前。

现在编辑 `cart` 应用的 `cart.py`，包含以下导入：

```python
from coupons.models import Coupon
```

把下面这行代码添加进 `Cart` 类的 `__init__()` 方法中来从会话中初始化优惠券：

```python
# store current applied coupon
self.coupon_id = self.session.get('coupon_id')
```

在这行代码中，我们尝试从当前会话中得到 `coupon_id` 会话键，然后把它保存在 `Cart` 对象中。把以下方法添加进 `Cart` 对象中：

```python
@property
def coupon(self):
	if self.coupon_id:
		return Coupon.objects.get(id=self.coupon_id)
	return None

def get_discount(self):
	if self.coupon:
		return (self.coupon.discount / Decimal('100')) \
				* self.get_total_price()
	return Decimal('0')

def get_total_price_after_discount(self):
	return self.get_total_price() - self.get_discount()
```

下面是这几个方法：

* `coupon()`：我们定义这个方法作为 `property` 。如果购物车包含 `coupon_id` 函数，就会返回一个带有给定 `id` 的`Coupon` 对象
* `get_discount()`：如果购物车包含 `coupon` ，我们就检索它的折扣比率，然后返回购物车中被扣除折扣的总和。
* `get_total_price_after_discount()`：返回被减去折扣之后的总价。

`Cart` 类现在已经准备好处理应用于当前会话的优惠券了，然后将它应用于相应折扣中。

让我们在购物车详情视图（view）中引入优惠券系统。编辑 `cart` 应用的 `views.py` ，然后把下面这一行添加到顶部：

```python
from coupons.forms import CouponApplyForm
```

之后，编辑 `cart_detail` 视图（view），然后添加新的表单：

```python
def cart_detail(request):
	cart = Cart(request)
	for item in cart:
		item['update_quantity_form'] = CartAddProductForm(
					initial={'quantity': item['quantity'],
					'update': True})
	coupon_apply_form = CouponApplyForm()
	return render(request,
			'cart/detail.html',
			{'cart': cart,
			'coupon_apply_form': coupon_apply_form})
```

编辑 `cart` 应用的 `acrt/detail.html` 文件，找到下面这几行：

```html
<tr class="total">
	<td>Total</td>
	<td colspan="4"></td>
	<td class="num">${{ cart.get_total_price }}</td>
</tr>
```

把它们换成以下几行：

```html
{% if cart.coupon %}
<tr class="subtotal">
	<td>Subtotal</td>
	<td colspan="4"></td>
	<td class="num">${{ cart.get_total_price }}</td>
</tr>
<tr>
<td>
	"{{ cart.coupon.code }}" coupon
	({{ cart.coupon.discount }}% off)
	</td>
	<td colspan="4"></td>
	<td class="num neg">
	- ${{ cart.get_discount|floatformat:"2" }}
	</td>
</tr>
{% endif %}
<tr class="total">
	<td>Total</td>
	<td colspan="4"></td>
	<td class="num">
	${{ cart.get_total_price_after_discount|floatformat:"2" }}
	</td>
</tr>
```

这段代码用于展示可选优惠券以及折扣率。如果购物车中有优惠券，我们就在第一行展示购物车的总价作为 **小计**。然后我们在第二行展示当前可应用于购物车的优惠券。最后，我们调用 `cart` 对象的 `get_total_price_after_discount()` 方法来展示折扣了的总价格。

在同一个文件中，在 `</table>` 标签之后引入以下代码：

```html
<p>Apply a coupon:</p>
<form action="{% url "coupons:apply" %}" method="post">
	{{ coupon_apply_form }}
	<input type="submit" value="Apply">
	{% csrf_token %}
</form>
```

我们将会展示一个表单来让用户输入优惠券代码，然后将它应用于当前的购物车当中。

访问 http://127.0.0.1:8000 ，向购物车当中添加一个商品，然后在表单中输入你创建的优惠代码来应用你的优惠券。你可以看到购物车像下面这样展示优惠券折扣：

![django-9-2](http://ohqrvqrlb.bkt.clouddn.com/django-9-2.png)

让我们把优惠券添加到购物流程中的下一步。编辑 `orders` 应用的 `orders/order/create.html` 模板（template），找到下面这几行：

```html
<ul>
{% for item in cart %}
<li>
	{{ item.quantity }}x {{ item.product.name }}
	<span>${{ item.total_price }}</span>
</li>
{% endfor %}
</ul>
```

把它替换为下面这几行：

```html
<ul>
{% for item in cart %}
	<li>
	{{ item.quantity }}x {{ item.product.name }}
	<span>${{ item.total_price }}</span>
	</li>
{% endfor %}
{% if cart.coupon %}
<li>
"{{ cart.coupon.code }}" ({{ cart.coupon.discount }}% off)
<span>- ${{ cart.get_discount|floatformat:"2" }}</span>
</li>
{% endif %}
</ul>
```

订单汇总现在已经包含使用了的优惠券，如果有优惠券的话。现在找到下面这一行：

```html
<p>Total: ${{ cart.get_total_price }}</p>
```

把他们换成以下一行：

```html
<p>Total: ${{ cart.get_total_price_after_discount|floatformat:"2" }}</p>
```

这样做之后，总价也将会在减去优惠券折扣被计算出来。
访问 http://127.0.0.1:8000/orders/create/ 。你会看到包含使用了优惠券的订单小计：

![django-9-3](http://ohqrvqrlb.bkt.clouddn.com/django-9-3.png)

用户现在可以在购物车当中使用优惠券了。尽管，当用户结账时，我们依然需要在订单中储存优惠券信息。

##**在订单中使用优惠券**
我们会储存每张订单中使用的优惠券。首先，我们需要修改 `Order` 模型（model）来储存相关联的 `Coupon` 对象，如果有这个对象的话。

编辑 `orders` 应用的 `models.py` 文件，添加以下代码：

```python
from decimal import Decimal
from django.core.validators import MinValueValidator, \
									MaxValueValidator
from coupons.models import Coupon
```

然后，把下列字段添加进 `Order` 模型（model）中：

```python
coupon = models.ForeignKey(Coupon,
							related_name='orders',
							null=True,
							blank=True)
discount = models.IntegerField(default=0,
						validators=[MinValueValidator(0),
								MaxValueValidator(100)])
```

这些字段让用户可以在订单中储存可选的优惠券信息，以及优惠券的相应折扣。折扣被存在关联的 `Coupon` 对象中，但是我们在 `Order` 模型（model）中引入它以便我们在优惠券被更改或者删除时好保存它。

因为 `Order` 模型（model）已经被改变了，我们需要创建迁移。执行下面的命令：

```
python manage.py makemigrations
```

你可以看到如下输出：

```
Migrations for 'orders':
	0002_auto_20150606_1735.py:
		- Add field coupon to order
		- Add field discount to order
```

用下面的命令来执行迁移：

```
python manage.py migrate orders
```

你必须要确保新的迁移已经被应用了。 `Order` 模型（model）的字段变更现在已经同步到了数据库中。

回到 `models.py` 文件中，按照如下更改 `Order` 模型（model）的 `get_total_cost()` 方法：

```python
def get_total_cost(self):
	total_cost = sum(item.get_cost() for item in self.items.all())
	return total_cost - total_cost * \
		(self.discount / Decimal('100'))
```

`Order` 模型（model）的 `get_total_cost()` 现在已经把使用了的折扣包含在内了，如果有折扣的话。

编辑 `orders` 应用的 `views.py` 文件，更改 `order_create` 视图（view）以便在创建新的订单时保存相关联的优惠券和折扣。找到下面这一行：

```python
order = form.save()
```

把它替换为下面这几行：

```python
order = form.save(commit=False)
if cart.coupon:
	order.coupon = cart.coupon
	order.discount = cart.coupon.discount
order.save()
```

在新的代码中，我们使用 `OrderCrateForm` 表单的 `save()` 方法创建了一个 `Order` 对象。我们使用 `commit=False` 来避免将它保存在数据库中。如果购物车当中有优惠券，我们就会保存相关联的优惠券和折扣。然后才把 `order` 对象保存到数据库中。

确保用 `python manage.py runserver` 运行了开发服务器。使用 `./ngrok http:8000` 命令来启动 Ngrok 。

在你的浏览器中打开 Ngrok 提供的 URL , 然后用用你创建的优惠券完成一次购物。当你完成一次成功的支付后，有可以访问 http://127.0.0.1:8000/admin/orders/order/ ，检查 `order` 对象是否包含了优惠券和折扣，如下：

![django-9-4](http://ohqrvqrlb.bkt.clouddn.com/django-9-4.png)

你也可以更改管理界面的订单详情模板（template）和订单的 PDF 账单，以便用展示购物车的方式来展示使用了的优惠券。

下面。我们将会为我们的项目添加国际化（internationalization）。

#**添加国际化（internationalization）和本地化（localization）**
Django 提供了完整的国际化和本地化的支持。这使得你可以把你的项目翻译为多种语言以及处理地区特性的 日期，时间，数字，和时区 的格式。让我们一起来搞清楚国际化和本地化的区别。国际化（通常简写为：**i18n**）是为让软件适应潜在的不同语言和多种语言的使用做的处理，这样它就不是以某种特定语言或某几种语言为硬编码的软件了。本地化（简写为：**l10n**）是对软件的翻译以及使软件适应某种特定语言的处理。Django 自身已经使用自带的国际化框架被翻译为了50多种语言。

##**使用 Django 国际化**##
国际化框架让你可以容易的在 Python 代码和模板（template）中标记需要翻译的字符串。它依赖于 GNU 文本获取集来生成和管理消息文件。**消息文件**是一个表示一种语言的纯文本文件。它包含某一语言的一部分或者所有的翻译字符串。消息文件有 `.po` 扩展名。

一旦翻译完成，信息文件就会被编译一遍快速的连接到被翻译的字符串。编译后的翻译文件的扩展名为 `.mo` 。

##**国际化和本地化设置**
Django 提供了几种国际化的设置。下面是几种最有关联的设置：

* `USE_I18N`：布尔值。用于设定 Django 翻译系统是否启动。默认值为 `True`。
	-`USE_L10N`：布尔值。表示本地格式化是否启动。当被激活式，本地格式化被用于展示日期和数字。默认为 `False` 。
* `USE_TZ`：布尔值。用于指定日期和时间是否是时区别（timezone-aware）。
* `LANGUAGE_CODE`：项目的默认语言。在标准的语言 ID 格式中，比如，`en-us` 是美式英语，`en-gb` 是英式英语。这个设置要 `USE_I18N` 为 `True` 才能生效。你可以在这个网站找到一个合法的语言 ID 表：`http://www.i18nguy.com/unicode/language-identifiers.html` 。
* `LANGUAGES`：包含项目可用语言的元组。它们由包含 **语言代码** 和 **语言名字** 的双元组构成的。你可以在 `django.conf.global_settions` 里看到可用语言的列表。当你选择你的网站将会使用哪一种语言时，你就把 `LANGUAGES` 设置为那个列表的子列表。
* `LOCAL_PATHS`：Django 寻找包含翻译的信息文件的路径列表。
* `TIME_ZONE`：代表项目时区的字符串。当你使用 `startproject` 命令创建新项目时它被设置为 `UTC` 。你也可以把它设置为其他的时区，比如 `Europe/Madrid` 。

这是一些可用的国际化和本地化的设置。你可以在这个网站找到全部的（设置）列表： https://docs.djangoproject.com/en/1.8/ref/settings/#globalization-i18n-l10n 。

##**国际化管理命令**
Django 使用 `manage.py` 或者 `django-admin.py` 工具包管理翻译，包含以下命令：

* `makemessages`：运行于源代码树中，寻找所有被用于翻译的字符串，然后在 `locale` 路径中创建或更新 `.po` 信息文件。每一种语言创建一个单一的 `.po` 文件。
* `compilemessages`： 把扩展名为 `.po` 的信息文件编译为用于检索翻译的 `.mo` 文件。

你需要文本获取工具集来创建，更新，以及编译信息文件。大部分的 Linux 发行版都包含文本获取工具集。如果你在使用 Mac OS X，用 Honebrew (`http://brew.sh`)是最简单的安装它的方法，使用以下命令 ：`brew install gettext`。你或许也需要将它和命令行强制连接 `brew link gettext --force` 。对于 Windows ，安装步骤如下 ： https://docs.djangoproject.com/en/1.8/topics/i18n/translation/#gettext-on-windows 。

##**怎么把翻译添加到 Django 项目中**
让我们看看国际化项目的过程。我们需要像下面这样做：

	1. 我们在 Python 代码和模板（template）中中标记需要翻译的字符串。
	2. 我们运行 `makemessages` 命令来创建或者更新包含所有翻译字符串的信息文件。
	3. 我们翻译包含在信息文件中的字符串，然后使用  `compilemessages` 编译他们。

##**Django 如何决定当前语言**
Django 配备有一个基于请求数据的中间件，这个中间件用于决定当前语言是什么。这个中间件是位于 `django.middleware.locale.localMiddleware` 的 `LocaleMiddleware` ,它执行下面的任务：

	1. 如果使用 `i18_patterns` —— 这是你使用的被翻译的 URL 模式，它在被请求的 URL 中寻找一个语言前缀来决定当前语言。
	2. 如果没有找到语言前缀，就会在当前用户会话中寻找当前的 `LANGUAGE_SESSION_KEY` 。
	3. 如果在会话中没有设置语言，就会寻找带有当前语言的 cookie 。这个 cookie 的定制名可在 `LANGUAGE_COOKIE_NAME` 中设置。默认地，这个 cookie 的名字是 `django_language` 。
	4. 如果没有找到 cookie，就会在请求 HTTP 头中寻找 `Accept-Language` 。
	5. 如果 `Accept-Language` 头中没有指定语言，Django 就使用在 `LANGUAGE_CODE` 设置中定义的语言。

默认的，Django 会使用 `LANGUAGE_OCDE` 中设置的语言，除非你正在使用 `LocaleMiddleware` 。上述操作进会在使用这个中间件时生效。

##**为我们的项目准备国际化**
让我们的项目准备好使用不同的语言吧。我们将要为我们的商店创建英文版和西班牙语版。编辑项目中的 `settings.py` 文件，添加下列 `LANGUAGES` 设置。把它放在 `LANGUAGE_OCDE` 旁边：

```python
LANGUAGES = (
		('en', 'English'),
		('es', 'Spanish'),
)
```

`LANGUAGES ` 设置包含两个由语言代码和语言名的元组构成，比如 `en-us` 或 `en-gb` ，或者一般的设置为 `en` 。这样设置之后，我们指定我们的应用只会对英语和西班牙语可用。如果我们不指定 `LANGUAGES` 设置，网站将会对所有 Django 的翻译语言有效。

确保你的 `LANGUAGE_OCDE` 像如下设置：

```
LANGUAGE_OCDE = 'en'
```

把 `django.middleware.locale.LocaleMiddleware` 添加到 `MIDDLEWARE_CLASSES` 设置中。确保这个设置在 `SessionsMiddleware` 之后，因为 `LocaleMiddleware` 需要使用会话数据。它同样必须放在 `CommonMiddleware` 之前，因为后者需要一种激活了的语言来解析请求 URL 。`MIDDLEWARE_CLASSES` 设置看起来应该如下：

```python
MIDDLEWARE_CLASSES = (
	'django.contrib.sessions.middleware.SessionMiddleware',
	'django.middleware.locale.LocaleMiddleware',
	'django.middleware.common.CommonMiddleware',
	# ...
)
```

>中间件的顺序非常重要，因为每个中间件所依赖的数据可能是由其他中间件处理之后的才获得的。中间件按照 `MIDDLEWARE_CLASSES` 的顺序应用在每个请求上，以及反序应用于响应中。

在主项目路径下，在 `manage.py` 文件同级，创建以下文件结构：

```
locale/
	en/
	es/
```

`locale` 路径是放置应用信息文件的地方。再次编辑 `settings.py` ，然后添加以下设置：

```
LOCALE_PATHS = (
	os.path.join(BASE_DIR, 'locale/'),
)
```

`LOCALE_PATHS` 设置指定了 Django 寻找翻译文件的路径。第一个路径有最高优先权。

当你在你的项目路径下使用 `makemessages` 命令时，信息文件将会在 `locale` 路径下生成。尽管，对于包含 `locale` 路径的应用，信息文件就会保存在这个应用的 `locale` 路径中。

##**翻译 Python 代码**
我们翻译在 Python 代码中的字母，你可以使用 `django.utils.translation` 中的 `gettext()` 函数来标记需要翻译的字符串。这个函数翻译信息然后返回一个字符串。约定俗成的用法是导入这个函数后把它命名为 `_` （下划线）。

你可以在这个网站找到所有关于翻译的文档 ： https://docs.djangoproject.com/en/1.8/topics/i18n/translation/ 。

##**标准翻译**
下面的代码展示了如何标记一个翻译字符串：

```python
from django.utils.translation import gettext as _
output = _('Text to be translated.')
```

##**惰性翻译（Lazy translation）**
Django 对所有的翻译函数引入了惰性（lazy）版，这些惰性函数都带有后缀 `_lazy()` 。当使用惰性函数时，字符串在值被连接时就会被翻译，而不是在被调用时翻译（这也是它为什么被惰性翻译的原因）。惰性函数迟早会派上用场，特别是当标记字符串在模型（model）加载的执行路径中时。

>使用 `gettext_lazy()` 而不是 `gettext()` ，字符串就会在连接到值的时候被翻译而不会在函数调用的时候被翻译。Django 为所有的翻译都提供了惰性版本。

##**翻译引入的变量**
标记的翻译字符串可以包含占位符来引入翻译中的变量。下面就是一个带有占位符的翻译字字符串的例子：

```python
from django.utils.translation import gettext as _
month = _('April')
day = '14'
output = _('Today is %(month)s %(day)s') % {'month': month,
                                            'day': day}
```

通过使用占位符，你可以重新排序文字变量。举个例子，以前的英文版本是 'Today is April 14' ，但是西班牙的版本是这样的 'Hoy es 14 de Abril' 。当你的翻译字符串有多于一个参数时，我们总是使用字符串插值来代替位置插值

##**翻译中的复数形式**
对于复数形式，你可以使用 `gettext()` 和 `gettext_lazy()` 。这些函数基于一个可以表示对象数量的参数来翻译单数和复数形式。下面这个例子展示了如何使用它们：

```python
output = ngettext('there is %(count)d product',
				'there are %(count)d products',
				count) % {'count': count}
```

现在你已经基本知道了如何在你的 Python 代码中翻译字符了，现在是把翻译应用到项目中的时候了。

##**翻译你的代码**
编辑项目中的 `settings.py` ，导入 `gettext_lazy()` 函数，然后像下面这样修改 `LANGUAGES` 的设置：

```python
from django.utils.translation import gettext_lazy as _

LANGUAGES = (
	('en', _('English')),
	('es', _('Spanish')),
)
```

我们使用 `gettext_lazy()` 而不是 `gettext()` 来避免循环导入，这样就可以在语言名被连接时就翻译它们。

在你的项目路径下执行下面的命令：

```python
django-admin makemessages --all
```

你可以看到如下输出：

```
processing locale es
processing locale en
```

看下 `locale/` 路径。你可以看到如下文件结构：

```
en/
	LC_MESSAGES/
		django.po
es/
	LC_MESSAGES/
		django.po
```

`.po` 文件已经为每一个语言创建好了。用文本编辑器打开 `es/LC_MESSAGES/django.po` 。在文件的末尾，你可以看到如下几行：

```
#: settings.py:104
msgid "English"
msgstr ""

#: settings.py:105
msgid "Spanish"
msgstr ""
```

每一个翻译字符串前都有一个显示该文件详情的注释以及它在哪一行被找到。每个翻译都包含两个字符串：

* `msgid`：在源代码中的翻译字符串
* `msgstr`：对应语言的翻译，默认为空。这是你输入所给字符串翻译的地方。

按照如下，根据所给的 `msgid` 在 `msgtsr` 中填写翻译：

```
#: settings.py:104
msgid "English"
msgstr "Inglés"

#: settings.py:105
msgid "Spanish"
msgstr "Español"
```

保存修改后的信息文件，打开 shell ，运行下面的命令：

```
django-admin compilemessages
```

如果一切顺利，你可以看到像如下的输出：

```
processing file django.po in myshop/locale/en/LC_MESSAGES
processing file django.po in myshop/locale/es/LC_MESSAGES
```

输出给出了被编译的信息文件的相关信息。再看一眼 `locale` 路径的 `myshop` 。你可以看到下面的文件：

```
en/
	LC_MESSAGES/
		django.mo
		django.po
es/
	LC_MESSAGES/
		django.mo
		django.po
```

你可以看到每个语言的 `.mo` 的编译文件已经生成了。

我们已经翻译了语言本身的名字。现在让我们翻译展示在网站中模型（model）字段的名字。编辑 `orders` 应用的 `models.py` ，为 `Order` 模型（model）添加翻译的被标记名：

```python
from django.utils.translation import gettext_lazy as _

class Order(models.Model):
	first_name = models.CharField(_('first name'),
								max_length=50)
	last_name = models.CharField(_('last name'),
								max_length=50)
	email = models.EmailField(_('e-mail'),)
	address = models.CharField(_('address'),
								max_length=250)
	postal_code = models.CharField(_('postal code'),
								max_length=20)
	city = models.CharField(_('city'),
						max_length=100)
#...
```

我们添加为当用户下一个新订单时展示的字段添加了名字，它们分别是 `first_name` ,`last_name`, `email`, `address`, `postal_code` ,`city`。记住，你也可以使用每个字段的 `verbose_name` 属性。

在 `orders` 应用路径内创建以下路径：

```
locale/
	en/
	es/
```

通过创建 `locale` 路径，这个应用的翻译字符串就会储存在这个路径下的信息文件里，而不是在主信息文件里。这样做之后，你就可以为每个应用生成独自的翻译文件。

在项目路径下打开 shell ，运行下面的命令：

```
django-admin makemessages --all
```

你可以看到如下输出：

```
processing locale es
processing locale en
```

用文本编辑器打开 `es/LC_MESSAGES/django.po` 文件。你将会看到每一个模型（model）的翻译字符串。为每一个所给的 `msgid` 字符串填写 `msgstr` 的翻译：

```
#: orders/models.py:10
msgid "first name"
msgstr "nombre"

#: orders/models.py:12
msgid "last name"
msgstr "apellidos"

#: orders/models.py:14
msgid "e-mail"
msgstr "e-mail"

#: orders/models.py:15
msgid "address"
msgstr "dirección"

#: orders/models.py:17
msgid "postal code"
msgstr "código postal"

#: orders/models.py:19
msgid "city"
msgstr "ciudad"
```

在你添加完翻译之后，保存文件。

在文本编辑器内，你可以使用 Poedit 来编辑翻译。 Poedit 是一个用来编辑翻译的软件，它使用  gettext 。它有 Linux ，Windows，Mac OS X 版，你可以在这个网站下载 Poedit : http://poedit.net/ 。

让我们来翻译项目中的表单吧。 `orders` 应用的 `OrderCrateForm` 还没有被翻译，因为它是一个 `ModelForm` ,使用 `Order` 模型（model）的 `verbose_name` 属性作为每个字段的标签。我们将会翻译 `cart` 和 `coupons` 应用的表单。

编辑 `cart` 应用路径下的 `forms.py` 文件，给 `CartAddProductForm` 的 `quantity` 字段添加一个 `label` 属性，然后按照如下标记这个需要翻译的字段：

```python
from django import forms
from django.utils.translation import gettext_lazy as _

PRODUCT_QUANTITY_CHOICES = [(i, str(i)) for i in range(1, 21)]

class CartAddProductForm(forms.Form):
	quantity = forms.TypedChoiceField(
						choices=PRODUCT_QUANTITY_CHOICES,
						coerce=int,
						label=_('Quantity'))
	update = forms.BooleanField(required=False,
						initial=False,
						widget=forms.HiddenInput)
```

编辑 `coupons` 应用的 `forms.py` ，按照如下翻译 `CouponApplyForm` ：

```python
from django import forms
from django.utils.translation import gettext_lazy as _

class CouponApplyForm(forms.Form):
	code = forms.CharField(label=_('Coupon'))
```

##**翻译模板（templates）**
Django 提供了 `{% trans %}` 和 `{% blocktrans %}` 模板（template）标签来翻译模板（template）中的字符串。为了使用翻译模板（template）标签，你需要在你的模板（template）顶部添加 `{%  load i18n %}` 来载入她们。

##**{% trans %}模板（template）标签**
`{% trans %}`模板（template）标签让你可以标记需要翻译的字符串，常量，或者是参数。在内部，Django 对所给文本执行 `gettext()` 。这是如何标记模板（template）中的翻译字符串：

```
{% trans "Text to be translated" %}
```

你可使用 `as` 来储存你在模板（template）内使用的全局变量里的被翻译内容。下面这个例子保存了一个变量中叫做 `greeting` 的被翻译文本：

```html
{% trans "Hello!" as greeting %}
<h1>{{ greeting }}</h1>
```

` {% trans %}` 标签对于简单的翻译字符串是很有用的，但是它不能包含变量的翻译内容。

##**{% blocktrans %}模板（template）标签**
`{% blocktrans %}` 模板（template）标签让你可以标记含有占位符的变量和字符的内容。线面这个例子展示了如何使用 `{% blocktrans %}` 标签来标记包含一个 `name` 变量的翻译内容：

```html
{% blocktrans %}Hello {{ name }}!{% endblocktrans %}
```

你可以用 `with` 来引入模板（template）描述，比如连接对象的属性或者应用模板（template）的变量过滤器。你必须为他们用占位符。你不能在 `blocktrans` 内连接任何描述或者对象属性。下面的例子展示了如何使用 `with` 来引入一个应用了 `capfirst` 过滤器的对象属性：

```html
{% blocktrans with name=user.name|capfirst %}
Hello {{ name }}!
{% endblocktrans %}
```

>当你的字符串中含有变量时使用 `{% blocktrans %}` 代替 `{% trans %}` 。

##**翻译商店模板（template）**
编辑 `shop` 应用的 `shop/base.html` 。确保你已经在顶部载入了 `i18n` 标签，然后按照如下标记翻译字符串：

```html
{% load i18n %}
{% load static %}
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<title>
{% block title %}{% trans "My shop" %}{% endblock %}
</title>
<link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
<div id="header">
<a href="/" class="logo">{% trans "My shop" %}</a>
</div>
<div id="subheader">
<div class="cart">
{% with total_items=cart|length %}
{% if cart|length > 0 %}
{% trans "Your cart" %}:
<a href="{% url "cart:cart_detail" %}">
{% blocktrans with total_items_plural=total_
items|pluralize
total_price=cart.get_total_price %}
{{ total_items }} item{{ total_items_plural }},
${{ total_price }}
{% endblocktrans %}
</a>
{% else %}
{% trans "Your cart is empty." %}
{% endif %}
{% endwith %}
</div>
</div>
<div id="content">
{% block content %}
{% endblock %}
</div>
</body>
</html>
```

注意展示在购物车小计的 `{% blocktrans %}` 标签。购物车小计在之前是这样的：

```html
{{ total_items }} item{{ total_items|pluralize }},
${{ cart.get_total_price }}
```

我们使用 ` {% blocktrans with ... %}` 来使用 `total_
items|pluralize` （模板（template）标签生效的地方）和 `cart_total_price` （连接对象方法的地方）的占位符：

```html
{% blocktrans with total_items_plural=total_items|pluralize
total_price=cart.get_total_price %}
{{ total_items }} item{{ total_items_plural }},
${{ total_price }}
{% endblocktrans %}
```

下面，编辑 `shop` 应用的 `shop/product/detail.html` 模板（template）然后在顶部载入 `i18n` 标签，但是它必须位于 `{% extends %}` 标签的下面：

```html
{% load i18n %}
```

找到下面这一行：

```html
<input type="submit" value="Add to cart">
```

把它替换为下面这一行：

```
<input type="submit" value="{% trans "Add to cart" %}">
```

现在翻译 `orders` 应用模板（template）。编辑 `orders` 应用的 `orders/order/create.html` 模板（template），然后标记翻译文本：

```html
{% extends "shop/base.html" %}
{% load i18n %}
{% block title %}
{% trans "Checkout" %}
{% endblock %}
{% block content %}
<h1>{% trans "Checkout" %}</h1>
<div class="order-info">
<h3>{% trans "Your order" %}</h3>
<ul>
{% for item in cart %}
<li>
{{ item.quantity }}x {{ item.product.name }}
<span>${{ item.total_price }}</span>
</li>
{% endfor %}
{% if cart.coupon %}
<li>
{% blocktrans with code=cart.coupon.code
discount=cart.coupon.discount %}
"{{ code }}" ({{ discount }}% off)
{% endblocktrans %}
<span>- ${{ cart.get_discount|floatformat:"2" }}</span>
</li>
{% endif %}
</ul>
<p>{% trans "Total" %}: ${{
cart.get_total_price_after_discount|floatformat:"2" }}</p>
</div>
<form action="." method="post" class="order-form">
{{ form.as_p }}
<p><input type="submit" value="{% trans "Place order" %}"></p>
{% csrf_token %}
</form>
{% endblock %}
```

看看本章中下列文件中的代码，看看字符串是如何被标记的：

* `shop` 应用：`shop/product/list.html`
* `orders` 应用：`orders/order/created.html`
* `cart` 应用：`cart/detail.html`

让位我们更新信息文件来引入新的翻译字符串。打开 shell ，运行下面的命令：

```
django-admin makemessages --all
```

`.po` 文件已经在 `myshop` 项目的 `locale` 路径下，你将看到 `orders` 应用现在已经包含我们标记的所有需要翻译的字符串。

编辑项目和`orderes` 应用中的 `.po` 翻译文件，然后引入西班牙语翻译。你可以参考本章中翻译了的 `.po` 文件：

在项目路径下打开 shell ，然后运行下面的命令：

```
cd orders/
django-admin compilemessages
cd ../
```

我们已经编译了 `orders` 应用的翻译。

运行下面的命令，这样应用中不包含 `locale` 路径的翻译就被包含进了项目的信息文件中：

```
django-admin compilemessages
```

##**使用 Rosetta 翻译交互界面**
Rosetta 是一个让你可以编辑翻译的第三方应用，它有类似 Django 管理站点的交互界面。Rosetta 让编辑 `.po` 文件和更新它变得很简单，让我们把它添加进项目中：

```
pip install django-rosetta==0.7.6
```

然后，把 `rosetts` 添加进项目中的`setting.py`文件中的`INSTALLED_APPS` 设置中。

你需要把 Rosetta 的 URL 添加进主 URL 配置中。编辑项目中的主 `urls.py` 文件，把下列 URL 模式添加进去：

```python
url(r'^rosetta/', include('rosetta.urls')),
```

确保你把它放在了 `shop.urls` 模式之前以避免错误的匹配。

访问 http://127.0.0.1:8000/admin/ ,然后使用超级管理员账户登录。然后导航到 http://127.0.0.1:8000/rosetta/ 。你可以看到当前语言列表：

![django-9-5](http://ohqrvqrlb.bkt.clouddn.com/django-9-5.png)

在 **Filter** 模块，点击 **All** 来展示所有可用的信息文件，包括属于 `orders` 应用的信息文件。点击**Spanish**模块下的 **Myshop** 链接来编辑西班牙语翻译。你可以看到翻译字符串的列表：

![django-9-6](http://ohqrvqrlb.bkt.clouddn.com/django-9-6.png)

你可以在 **Spanish** 行下输入翻译。**Occurences** 行展示了代码中翻译字符串被找到的文件和行数、

包含占位符的翻译看起来像这样：

![django-9-7](http://ohqrvqrlb.bkt.clouddn.com/django-9-7.png)

Rosetta 使用了不同的背景色来展示占位符。当你翻译内容时，确保你没有翻译占位符。比如，用下面这个字符串举例：

```
%(total_items)s item%(total_items_plural)s, $%(total_price)s
```

它翻译为西班牙语之后长得像这样：

```
%(total_items)s producto%(total_items_plural)s, $%(total_price)s
```

你可以看看本章项目中使用相同西班牙语翻译的文件源代码。

当你完成编辑翻译后，点击**Save and translate next block** 按钮来保存翻译到 `.po` 文件中。Rosetta 在你保存翻译时编辑信息文件，所以并不需要你运行 `compilemessages` 命令。尽管，Rosetta 需要 `locale`  路径的写入权限来写入信息文件。确保路径有有效权限。

如果你想让其他用户编辑翻译，访问：http://127.0.0.1:8000/admin/auth/group/add/,然后创建一个名为 `translations` 的新组。当编辑一个用户时，在**Permissions**模块下，把 `translations` 组添加进**ChosenGroups**中。Rosetta 只对超级用户和 `translations` 中的用户是可用的。

你可以在这个网站阅读 Rosetta 的文档：http://django-rosetta.readthedocs.org/en/latest/ 。

>当你在生产环境中添加新的翻译时，如果你的 Django 运行在一个真实服务器上，你必须在运行 `compilemessages` 或保存翻译之后重启你的服务器来让 Rosetta 的更改生效。

##**惰性翻译**
你或许已经注意到了 Rosetta 有 **Fuzzy** 这一行。这不是 Rosetta 的特性，它是由 gettext 提供的。如果翻译的 flag 是激活的，那么它就不会被包含进编译后的信息文件中。flag 用于需要翻译器修订的翻译字符串。当 `.po` 文件在更新新的翻译字符串时，一些翻译字符串可能被自动标记为 fuzzy. 当 gettext 找到一些变动不大的 `msgid` 时就会发生这样的情况，gettext 就会把它认为的旧的翻译和匹配在一起然后会在回把它标记为 fuzzy 以用于回查。翻译器之后会回查模糊翻译，会移除 fuzzy 标签然后再次编译信息文件。

##**国际化的 URL 模式**
Django 提供了用于国际化的 URLs。它包含两种主要用于国际化的 URLs：

* **URL 模式中的语言前缀**：把语言的前缀添加到 URLs 当中，以便在不同的基本 URL 下提供每一种语言的版本。
* **翻译后的 URL 模式**：标记要翻译的 URL 模式，这样同样的 URL 就可以服务于不同的语言。

翻译 URLs 的原因是这样就可以优化你的站点，方便搜索引擎搜索。通过添加语言前缀，你就可以为每一种语言提供索引，而不是所有语言用一种索引。并且, 把 URLs 为不同语言，你就可以提供给搜索引擎更好的搜索序列。

##**把语言前缀添加到 URLs 模式中**
Django 允许你可以把语言前缀添加到 URLs 模式中。举个例子，网站的英文版可以服务于 `/en/` 起始路径之下，西班牙语版服务于 `/es/` 起始路径之下。

为了在你的 URLs 模式中使用不同语言，你必须确保 `settings.py` 中的 `MIDDLEWARE_CLASSES` 设置中有 `django.middleware.localMiddlewar` 。Django 将会使用它来辨别当前请求中的语言。

让我们把语言前缀添加到 URLs 模式中。编辑 `myshop` 项目的 `urls.py` ，添加以下库：

```python
from django.conf.urls.i18n import i18n_patterns
```

然后添加 `i18n_patterns()` :

```python
urlpatterns = i18n_patterns(
	url(r'^admin/', include(admin.site.urls)),
	url(r'^cart/', include('cart.urls', namespace='cart')),
	url(r'^orders/', include('orders.urls', namespace='orders')),
	url(r'^payment/', include('payment.urls',
						namespace='payment')),
	url(r'^paypal/', include('paypal.standard.ipn.urls')),
	url(r'^coupons/', include('coupons.urls',
					namespace='coupons')),
	url(r'^rosetta/', include('rosetta.urls')),
	url(r'^', include('shop.urls', namespace='shop')),
	)
```

你可以把 `i18n_patterns()` 和 `patterns()` URLs 模式结合起来，这样一些模式就会包含语言前缀另一些就不会包含。尽管，最好还是使用翻译后的 URLs 来避免 URL 匹配一个未翻译的 URL 模式的可能。

打开开发服务器，访问 http://127.0.0.1:8000/ ，因为你在 Django 中使用 `LocaleMiddleware` 来执行 `How Django determines the current language` 中描述的步骤来决定当前语言，然后它就会把你重定向到包含相同语言前缀的 URL。看看浏览器中 URL ，它看起来像这样：http://127.0.0.1:8000/en/.当前语言将会被设置在浏览器的 `Accept-Language` 头中，设为英语或者西班牙语或者是 `LANGUAGE_OCDE`（English） 中的默认设置。

##**翻译 URL 模式**
Django 支持 URL 模式中有翻译了的字符串。你可以为每一种语言使用不同的 URL 模式。你可以使用 `ugettet_lazy()` 函数标记 URL 模式中需要翻译的字符串。

编辑 `myshop` 项目中的 `urls.py` 文件，把翻译字符串添加进 `cart`,`orders`,`payment`,`coupons` 的 URLs 模式的正则表达式中：

```python
from django.utils.translation import gettext_lazy as _

urlpatterns = i18n_patterns(
	url(r'^admin/', include(admin.site.urls)),
	url(_(r'^cart/'), include('cart.urls', namespace='cart')),
	url(_(r'^orders/'), include('orders.urls',
					namespace='orders')),
	url(_(r'^payment/'), include('payment.urls',
					namespace='payment')),
	url(r'^paypal/', include('paypal.standard.ipn.urls')),
	url(_(r'^coupons/'), include('coupons.urls',
					namespace='coupons')),
	url(r'^rosetta/', include('rosetta.urls')),
	url(r'^', include('shop.urls', namespace='shop')),
)
```

编辑 `orders` 应用的 `urls.py` 文件，标记需要翻译的 URLs 模式：

```python
from django.conf.urls import url
from .import views
from django.utils.translation import gettext_lazy as _

urlpatterns = [
	url(_(r'^create/$'), views.order_create, name='order_create'),
	# ...
]
```

编辑 `payment` 应用的 `urls.py` 文件，把代码改成这样：

```python
from django.conf.urls import url
from . import views
from django.utils.translation import gettext_lazy as _

urlpatterns = [
	url(_(r'^process/$'), views.payment_process, name='process'),
	url(_(r'^done/$'), views.payment_done, name='done'),
	url(_(r'^canceled/$'),
		views.payment_canceled,
		name='canceled'),
]
```

我们不需要翻译 `shop` 应用的 URL 模式，因为它们是用变量创建的，而且也没有包含其他的任何字符。

打开 shell ，运行下面的命令来把新的翻译更新到信息文件：

```
django-admin makemessages --all
```

确保开发服务器正在运行中，访问：http://127.0.0.1:8000/en/rosetta/ ,点击**Spanish**下的**Myshop** 链接。你可以使用显示过滤器（display filter）来查看没有被翻译的字符串。确保你的 URL 翻译有正则表达式中的特殊字符。翻译 URLs 是一个精细活儿；如果你替换了正则表达式，你可能会破坏 URL。

##**允许用户切换语言**
因为我们正在提供多语种服务，我们应当让用户可以选择站点的语言。我们将会为我们的站点添加语言选择器。语言选择器由可用的语言列表组成，我们使用链接来展示这些语言选项：

编辑 `shop/base.html` 模板（template），找到下面这一行：

```html
<div id="header">
<a href="/" class="logo">{% trans "My shop" %}</a>
</div>
```

把他们换成以下几行：：

```html
<div id="header">
<a href="/" class="logo">{% trans "My shop" %}</a>
{% get_current_language as LANGUAGE_CODE %}
{% get_available_languages as LANGUAGES %}
{% get_language_info_list for LANGUAGES as languages %}
<div class="languages">
<p>{% trans "Language" %}:</p>
<ul class="languages">
{% for language in languages %}
<li>
<a href="/{{ language.code }}/" {% if language.code ==
LANGUAGE_CODE %} class="selected"{% endif %}>
{{ language.name_local }}
</a>
</li>
{% endfor %}
</ul>
</div>
</div>
```

我们是这样创建我们的语言选择器的：

	1. 首先使用 `{% load i18n %}` 加载国际化
	2. 使用 `{% get_current_language %}` 标签来检索当前语言
	3. 使用 `{% get_available_languages %}` 模板（template）标签来过去 `LANGUAGES` 中定义语言
	4. 使用 `{% get_language_info_list %}` 来提供简易的语言属性连接入口
	5. 我们创建了一个 HTML 列表来展示所有的可用语言列表然后我们为当前激活语言添加了 `selected` 属性

我们使用基于项目设置中语言变量的 i18n` 提供的模板（template）标签。现在访问：`http://127.0.0.1:8000/` ，你应该能在站点顶部的右边看到语言选择器：

![django-9-8](http://ohqrvqrlb.bkt.clouddn.com/django-9-8.png)
	
用户现在可以轻易的切换他们的语言了。

##**使用 django-parler 翻译模型（models）**
Django 没有提供开箱即用的模型（models）翻译的解决办法。你必须要自己实现你自己的解决办法来管理不同语言中的内容或者使用第三方模块来管理模型（model）翻译。有几种第三方应用允许你翻译模型（model）字段。每一种手采用了不同的方法保存和连接翻译。其中一种应用是 django-parler 。这个模块提供了非常有效的办法来翻译模型（models）并且它和 Django 的管理站点整合的非常好。

django-parler 为每一个模型（model）生成包含翻译的分离数据库表。这个表包含了所有的翻译的字段和源对象的外键翻译。它也包含了一个语言字段，一位内每一行都会保存一个语言的内容。

##**安装 django-parler **
使用 pip 安装 django-parler :

```
pip install django-parler==1.5.1
```

编辑项目中 `settings.py` ，把 `parler` 添加到 `INSTALLED_APPS` 中，同样也把下面的配置添加到配置文件中：

```
PARLER_LANGUAGES = {
	None: (
		{'code': 'en',},
		{'code': 'es',},
	),
	'default': {
		'fallback': 'en',
		'hide_untranslated': False,
	}
}
```

这个设置为 django-parler 定义了可用语言 `en` 和 `es` 。我们指定默认语言为 `en` ，然后我们指定 django-parler 应该隐藏未翻译的内容。

##**翻译模型（model）字段**
让我们为我们的产品目录添加翻译。 django-parler 提供了 `TranslatedModel` 模型（model）类和 `TranslatedFields` 闭包（wrapper）来翻译模型（model）字段。编辑 `shop` 应用路径下的 `models.py` 文件，添加以下导入：

```python
from parler.models import TranslatableModel, TranslatedFields
```

然后更改 `Category` 模型（model）来使 `name` 和 `slug` 字段可以被翻译。
我们依然保留了每行未翻译的字段：

```python
class Category(TranslatableModel):
	name = models.CharField(max_length=200, db_index=True)
	slug = models.SlugField(max_length=200,
							db_index=True,
							unique=True)
	translations = TranslatedFields(
		name = models.CharField(max_length=200,
								db_index=True),
		slug = models.SlugField(max_length=200,
								db_index=True,
								unique=True)
)
```

`Category` 模型（model）继承了 `TranslatableModel` 而不是 `models.Model`。并且 `name` 和 `slug` 字段都被引入到了 `TranslatedFields` 闭包（wrapper）中。

编辑 `Product` 模型（model），为 `name`,`slug` ,`description` 添加翻译，同样我们也保留了每行未翻译的字段：

```python
class Product(TranslatableModel):
	name = models.CharField(max_length=200, db_index=True)
	slug = models.SlugField(max_length=200, db_index=True)
	description = models.TextField(blank=True)
	translations = TranslatedFields(
		name = models.CharField(max_length=200, db_index=True),
		slug = models.SlugField(max_length=200, db_index=True),
		description = models.TextField(blank=True)
	)
	category = models.ForeignKey(Category,
						related_name='products')
	image = models.ImageField(upload_to='products/%Y/%m/%d',
						blank=True)
	price = models.DecimalField(max_digits=10, decimal_places=2)
	stock = models.PositiveIntegerField()
	available = models.BooleanField(default=True)
	created = models.DateTimeField(auto_now_add=True)
	updated = models.DateTimeField(auto_now=True)
```

django-parler 为每个可翻译模型（model）生成了一个模型（model）。在下面的图片中，你可以看到 `Product` 字段和生成的 `ProductTranslation` 模型（model）：

![django-9-9](http://ohqrvqrlb.bkt.clouddn.com/django-9-9.png)

django-parler 生成的 `ProductTranslation` 模型（model）有 `name`,`slug`,`description` 可翻译字段，一个 `language_code` 字段，和主要的 `Product` 对象的 `ForeignKey` 字段。`Product` 和 `ProductTranslation` 之间有一个一对多关系。一个 `ProductTranslation` 对象会为每个可用语言生成 `Product` 对象。

因为 Django 为翻译都使用了相互独立的表，所以有一些 Django 的特性我们是用不了。使用翻译后的字段来默认排序是不可能的。你可以在查询集（QuerySets）中使用翻译字段来过滤，但是你不可以在 `ordering Meta` 选项中引入可翻译字段。编辑 `shop` 应用的 `models.py` ，然后注释掉 `Category Meta` 类的 `ordering` 属性：

```python
class Meta:
	# ordering = ('name',)
	verbose_name = 'category'
	verbose_name_plural = 'categories'
```

我们也必须注释掉 `Product Meta` 类的 `index_together` 属性，因为当前的 django-parler 的版本不支持对它的验证。编辑 `Product Meta` ：

```python
class Meta:
	ordering = ('-created',)
	# index_together = (('id', 'slug'),)
```

你可以在这个网站阅读更多 django-parler 兼容的有关内容：
http://django-parler.readthedocs.org/en/latest/compatibility.html 。

##**创建一次定制的迁移**
当你创建新的翻译模型（models）时，你需要执行 `makemessages` 来生成模型（models）的迁移，然后把变更同步到数据库中。尽管当使已存在字段可翻译化时，你或许有想要在数据库中保留的数据。我们将迁移当前数据到新的翻译模型（models）中。因此，我们添加了翻译字段但是有意保存了原始字段。

翻译添加到当前字段的步骤如下：

	1. 在保留原始字段的情况下，创建新翻译模型（model）的迁移
	2. 创建定制化迁移，从当前已有的字段中拷贝一份数据到翻译模型（models）中
	3. 从源模型（models）中删除字段

运行下面的命令来为我们添加到 `Category` 和 `Product` 模型（model）中的翻译字段创建迁移：

```
python manage.py makemigrations shop --name "add_translation_model"
```

你可以看到如下输出：

```
Migrations for 'shop':
	0002_add_translation_model.py:
		- Create model CategoryTranslation
		- Create model ProductTranslation
		- Change Meta options on category
		- Alter index_together for product (0 constraint(s))
		- Add field master to producttranslation
		- Add field master to categorytranslation
		- Alter unique_together for producttranslation (1 constraint(s))
		- Alter unique_together for categorytranslation (1 constraint(s))
```

##**迁移已有数据**
现在我们需要创建定制迁移来把已有数据拷贝到新的翻译模型（model）中。使用这个命令创建一个空的迁移：

```
python manage.py makemigrations --empty shop --name "migrate_translatable_fields"
```

你将会看到如下输出：

```
Migrations for 'shop':
	0003_migrate_translatable_fields.py
```

编辑 `shop/migrations/0003_migrate_translatable_fields.py` ，然后添加下面的代码：

```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals
from django.db import models, migrations
from django.apps import apps
from django.conf import settings
from django.core.exceptions import ObjectDoesNotExist

translatable_models = {
	'Category': ['name', 'slug'],
	'Product': ['name', 'slug', 'description'],
}

def forwards_func(apps, schema_editor):
	for model, fields in translatable_models.items():
		Model = apps.get_model('shop', model)
		ModelTranslation = apps.get_model('shop',
								'{}Translation'.format(model))

	for obj in Model.objects.all():
		translation_fields = {field: getattr(obj, field) for field in fields}
		translation = ModelTranslation.objects.create(
						master_id=obj.pk,
						language_code=settings.LANGUAGE_CODE,
						**translation_fields)

def backwards_func(apps, schema_editor):
	for model, fields in translatable_models.items():
		Model = apps.get_model('shop', model)
		ModelTranslation = apps.get_model('shop',
								'{}Translation'.format(model))
	for obj in Model.objects.all():
		translation = _get_translation(obj, ModelTranslation)
		for field in fields:
			setattr(obj, field, getattr(translation, field))
		obj.save()

def _get_translation(obj, MyModelTranslation):
	translations = MyModelTranslation.objects.filter(master_id=obj.pk)
	try:
		# Try default translation
		return translations.get(language_code=settings.LANGUAGE_CODE)
	except ObjectDoesNotExist:
		# Hope there is a single translation
		return translations.get()

class Migration(migrations.Migration):
	dependencies = [
	('shop', '0002_add_translation_model'),
	]
	operations = [
	migrations.RunPython(forwards_func, backwards_func),
	]
```

这段迁移包括了用于执行应用和反转迁移的 `forwards_func()` 和 `backwards_func()` 。

迁移工作流程如下：

	1. 我们在 `translatable_models` 字典中定义了模型（model）和它们的可翻译字段
	2. 为了应用迁移，我们使用 `app.get_model()` 来迭代包含翻译的模型（model）来得到这个模型（model）和其可翻译的模型（model）
	3. 我们在数据库中迭代所有的当前对象，然后为定义在项目设置中的 `LANGUAGE_CODE` 创建一个翻译对象。我们引入了 `ForeignKey` 到源对像和一份从源字段中可翻译字段的拷贝。

backwards 函数执行的是反转操作，它检索默认的翻译对象，然后把可翻译字段的值拷贝到新的模型（model）中。

最后，我们需要删除我们不再需要的源字段。编辑 `shop` 应用的 `models.py` ，然后删除 `Category` 模型（model）的 `name` 和 `slug` 字段：

```python
class Category(TranslatableModel):
	translations = TranslatedFields(
		name = models.CharField(max_length=200, db_index=True),
		slug = models.SlugField(max_length=200,
						db_index=True,
						unique=True)
	)
```

删除 `Product` 模型（model）的 `name` 和 `slug` 字段：

```python
class Product(TranslatableModel):
	translations = TranslatedFields(
		name = models.CharField(max_length=200, db_index=True),
		slug = models.SlugField(max_length=200, db_index=True),
		description = models.TextField(blank=True)
	)
	category = models.ForeignKey(Category,
						related_name='products')
	image = models.ImageField(upload_to='products/%Y/%m/%d',
						blank=True)
	price = models.DecimalField(max_digits=10, decimal_places=2)
	stock = models.PositiveIntegerField()
	available = models.BooleanField(default=True)
	created = models.DateTimeField(auto_now_add=True)
	updated = models.DateTimeField(auto_now=True)
```

现在我们必须要创建最后一次迁移来应用其中的变更。如果我们尝试 `manage.py` 工具，我们将会得到一个错误，因为我们还没有让管理站点适应翻译模型（models）。让我们先来修整一下管理站点。

##**在管理站点中整合翻译**
Django-parler 很好和 Django 的管理站点相融合。它用 `TranslatableAdmin` 重写了 Django 的 `ModelAdmin` 类 来管理翻译。

编辑 shop 应用的 `admin.py` 文件，添加下面的库：

```
from parler.admin import TranslatableAdmin
```

把 `CategoryAdmin` 和 `ProductAdmin` 的继承类改为 `TranslatableAdmin` 取代原来的 `ModelAdmin`。 Django-parler 现在还不支持 `prepopulated_fields` 属性，但是它支持 `get_prepopulated_fields()` 方法来提供相同的功能。让我们据此做一些改变。 `admin.py` 文件看起来像这样：

```python
from django.contrib import admin
from .models import Category, Product
from parler.admin import TranslatableAdmin

class CategoryAdmin(TranslatableAdmin):
	list_display = ['name', 'slug']
	
	def get_prepopulated_fields(self, request, obj=None):
		return {'slug': ('name',)}
admin.site.register(Category, CategoryAdmin)

class ProductAdmin(TranslatableAdmin):
	list_display = ['name', 'slug', 'category', 'price', 'stock',
					'available', 'created', 'updated']
	list_filter = ['available', 'created', 'updated', 'category']
	list_editable = ['price', 'stock', 'available']

	def get_prepopulated_fields(self, request, obj=None):
		return {'slug': ('name',)}

admin.site.register(Product, ProductAdmin)
```

我们已经使管理站点能够和新的翻译模型（model）工作了。现在我们就能把变更同步到数据库中了。

##**应用翻译模型（model）迁移**
我们在变更管理站点之前已经从模型（model）中删除了旧的字段。现在我们需要为这次变更创建迁移。打开 shell ，运行下面的命令：

```
python manage.py makemigrations shop --name "remove_untranslated_fields"
```

你将会看到如下输出：

```
Migrations for 'shop':
	0004_remove_untranslated_fields.py:
		- Remove field name from category
		- Remove field slug from category
		- Remove field description from product
		- Remove field name from product
		- Remove field slug from product
```

迁移之后，我们就已经删除了源字段保留了可翻译字段。

总结一下，我们创建了以下迁移：

	1. 将可翻译字段添加到模型（models）中
	2. 将源字段中的数据迁移到可翻译字段中
	3. 从源模型（models）中删除源字段

运行下面的命令来应用我们创建的迁移：

```
python manage.py migrate shop
```

你将会看到如下输出：

```
Applying shop.0002_add_translation_model... OK
Applying shop.0003_migrate_translatable_fields... OK
Applying shop.0004_remove_untranslated_fields... OK
```

我们的模型（models）现在已经同步到了数据库中。让我们来翻译一个对象。

用命令  `pythong manage.py runserver` 运行开发服务器，在浏览器中访问：http://127.0.0.1:8000/en/admin/shop/category/add/ 。你就会看到包含两个标签的 **Add category** 页，一个标签是英文的，一个标签是西班牙语的。

![django-9-10](http://ohqrvqrlb.bkt.clouddn.com/django-9-10.png)

你现在可以添加一个翻译然后点击**Save**按钮。确保你在切换标签之前保存了他们，不然让门就会丢失。

##**使视图（views）适应翻译**
我们要使我们的 `shop` 视图（views）适应我们的翻译查询集（QuerySets）。在命令行运行 `python manage.py shell` ，看看你是如何检索和查询翻译字段的。为了从当前激活语言中得到一个字段的内容，你只需要用和连接一般字段相同的办法连接这个字段：

```
>>> from shop.models import Product
>>> product=Product.objects.first()
>>> product.name
'Black tea'
```

当你连接到被翻译了字段时，他们已经被用当前语言处理过了。你可以为一个对象设置不同的当前语言，这样你就可以获得指定的翻译了：

```
>>> product.set_current_language('es')
>>> product.name
'Té negro'
>>> product.get_current_language()
'es
```

当使用 `filter()` 执行一次查询集（QuerySets）时，你可以使用  `translations__` 语法筛选相关联对象:

```
>>> Product.objects.filter(translations__name='Black tea')
[<Product: Black tea>]
```

你也可以使用 `language()` 管理器来为每个检索的对象指定特定的语言：

```
>>> Product.objects.language('es').all()
[<Product: Té negro>, <Product: Té en polvo>, <Product: Té rojo>,
<Product: Té verde>]
```

如你所见，连接到翻译字段的方法还是很直接的。

让我们修改一下产品目录的视图（views）吧。编辑 `shop` 应用的 `views.py` ，在 `product_list` 视图（view）中找到下面这一行：

```python
category = get_object_or_404(Category, slug=category_slug)
```

把它替换为下面这几行：

```python
language = request.LANGUAGE_CODE
category = get_object_or_404(Category,
					translations__language_code=language,
					translations__slug=category_slug)
```

然后，编辑 `product_detail` 视图（view），找到下面这几行：

```python
product = get_object_or_404(Product,
							id=id,
							slug=slug,
							available=True)
```

把他们换成以下几行：：

```python
language = request.LANGUAGE_CODE
get_object_or_404(Product,
					id=id,
					translations__language_code=language,
					translations__slug=slug,
					available=True)
```

现在 `product_list` 和 `product_detail` 已经可以使用翻译字段来检索了。运行开发服务器，访问：http://127.0.0.1:8000/es/ ，你可以看到产品列表页，包含了所有被翻译为西班牙语的产品：

![django-9-11](http://ohqrvqrlb.bkt.clouddn.com/django-9-11.png)

现在每个产品的 URLs 已经使用 `slug` 字段被翻译为了的当前语言。比如，一个西班牙语产品的 URL 为：http://127.0.0.1:8000/es/2/te-rojo/ ,但是它的英文 URL 为：http://127.0.0.1:8000/en/2/red-tea/ 。如果你导航到产品详情页，你可以看到用被选语言翻译后的 URL 和内容，就像下面的例子：

![django-9-12](http://ohqrvqrlb.bkt.clouddn.com/django-9-12.png)

如果你想要更多了解 django-parler ，你可以在这个网站找到所有的文档：http//django-parler.readthedocs.org/en/latest/ 。

你已经学习了如何翻译 Python 代码，模板（template），URL 模式，模型（model）字段。为了完成国际化和本地化的工作，我们需要展示本地化格式的时间和日期、以及数字。

##**本地格式化**
基于用户的语言，你可能想要用不同的格式展示日期、时间和数字。本第格式化可通过修改 `settings.py ` 中的 `USE_L1ON` 设置为 `True` 来使本地格式化生效。

当 `USE_L1ON` 可用时，Django 将会在模板（template）任何输出一个值时尝试使用某一语言的特定格式。你可以看到你的网站中用一个点来做分隔符的十进制英文数字，尽管在西班牙语版中他们使用逗号来做分隔符。这和 Django 里为 `es` 指定的语言格式。你可以在这里查看西班牙语的格式配置：https://github.com/django/django/blob/stable/1.8.x/django/conf/locale/es/formats.py 。

通常，你把 `USE_L10N` 的值设为 `True` 然后 Django 就会为每一种语言应用本地化格式。虽然在某些情况下你有不想要被格式化的值。这和输出特别相关， JavaScript 或者 JSON 必须要提供机器可读的格式。

Django 提供了 `{% localize %}` 模板（template）表标签来让你可以开启或者关闭模板（template）的本地化。这使得你可以控制本地格式化。你需要载入 `l10n ` 标签来使用这个模板（template）标签。下面的例子是如何在模板（template）中开启和关闭它：

```html
{% load l10n %}

{% localize on %}
{{ value }}
{% endlocalize %}

{% localize off %}
{{ value }}
{% endlocalize %}
```

Django 同样也提供了 `localize` 和 `unlocalize` 模板（template）过滤器来强制或取消一个值的本地化。过滤器可以像下面这样使用：

```html
{{ value|localize }}
{{ value|unlocalize }}
```

你也可以创建定制化的格式文件来指定特定语言的格式。你可以在这个网站找到所有关于本第格式化的信息：https://docs.djangoproject.com/en/1.8/topics/i18n/formatting/ 。

##**使用 django-localflavor 来验证表单字段**
django-localflavor 是一个包含特殊工具的第三方模块，比如它的有些表单字段或者模型（model）字段对于每个国家是不同的。它对于验证本地地区，本地电话号码，身份证号，社保号等等都是非常有用的。这个包被集成进了一系列以 ISO 3166 国家代码命名的模块里。

使用下面的命令安装 django-localflavor :

```
pip install django-localflavor==1.1
```

编辑项目中的 `settings.py` ，把 `localflavor` 添加进 `INSTALLED_APPS` 设置中。

我们将会添加美国（U.S.）邮政编码字段，这样之后创建一个新的订单就需要一个美国邮政编码了。

编辑 `orders` 应用的 `forms.py` ，让它看起来像这样：

```python
from django import forms
from .models import Order
from localflavor.us.forms import USZipCodeField

class OrderCreateForm(forms.ModelForm):
	postal_code = USZipCodeField()
	class Meta:
	model = Order
	fields = ['first_name', 'last_name', 'email', 'address',
						'postal_code', 'city',]
```

我们从 `localflavor` 的 `us` 包中引入了 `USZipCodeField` 然后将它应用在 `OrderCreateForm` 的 `postal_code` 字段中。访问：http://127.0.0.1:8000/en/orders/create/ 然后尝试输入一个 3 位邮政编码。你将会得到 `USZipCodeField` 引发的验证错误：

```
Enter a zip code in the format XXXXX or XXXXX-XXXX.
```

这只是一个在你的项目中使用定制字段来达到验证目的的简单例子，localflavor 提供的本地组件对于让你的项目适应不同的国家是非常有用的。你可以阅读 django-localflavor 的文档，看看所有的可用地区组件：https://django-localflavor.readthedocs.org/en/latest/ 。

下面，我们将会为我们的店铺创建一个推荐引擎。

##**创建一个推荐引擎**
推荐引擎是一个可以预测用户对于某一物品偏好和比率的系统。这个系统会基于用户的行为和它对于用户的了解选出相关的商品。如今，推荐系统用于许多的在线服务。它们帮助用户从大量的相关数据中挑选他们可能感兴趣的产品。提供良好的推荐可以鼓励用户更多的消费。电商网站受益于日益增长的相关产品推荐销售中。

我们将会创建一个简单但强大的推荐引擎来推荐经常一起购买的商品。我们将基于历史销售来推荐商品，这样就可以辨别出哪些商品通常是一起购买的了。我们将会在两种不同的场景下推荐互补的产品:

* **产品详情页**：我们将会展示和所给商品经常一起购买的产品列表。比如像这样展示：**Users who bought this also bought X, Y, Z**（购买了这个产品的用户也购买了X, Y,Z）。我们需要一个数据结构来让我们能储被展示产品中和每个产品一起购买的次数。
* **购物车详情页**：基于用户添加到购物车当中的产品，我们将会推荐经常和他们一起购买的产品。在这样的情况下，我们计算的包含相关产品的评分一定要相加。

我们将会使用 Redis 来储存被一起购买的产品。记得你已经在**第六章 跟踪用户操作 **使用了 Redis 。如果你还没有安装 Redis ，你可以在那一章找到安装指导。

##**推荐基于历时购物的产品**
现在，让我们推荐给用户一些基于他们添加到购物车的产品。我们将会在 Redis 中为每个产品储存一个键（key）。产品的键将会和它的评分一同储存在 Redis 的有序集中。在一次新的订单被完成时，我们每次都会为一同购买的产品的评分加一。

当一份订单付款成功后，我们保存每个购买产品的键，包括同意订单中的有序产品集。这个有序集让我们可以为一起购买的产品打分。

编辑项目中的额 `settings.py` ,添加以下设置：

```
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 1
```

这里的设置是为了和 Redis 服务器建立连接。在 `shop` 应用内创建一个新的文件，命名为 `recommender.py` 。添加以下代码：

```python
import redis
from django.conf import settings
from .models import Product

# connect to redis
r = redis.StrictRedis(host=settings.REDIS_HOST,
					port=settings.REDIS_PORT,
					db=settings.REDIS_DB)

class Recommender(object):

	def get_product_key(self, id):
		return 'product:{}:purchased_with'.format(id)
	
	def products_bought(self, products):
		product_ids = [p.id for p in products]
		for product_id in product_ids:
			for with_id in product_ids:
				# get the other products bought with each product
				if product_id != with_id:
					# increment score for product purchased together
					r.zincrby(self.get_product_key(product_id),
									with_id,
									amount=1)				
```	

`Recommender` 类使我们可以储存购买的产品然后检索所给产品的产品推荐。`get_product_key()` 方法检索 `Product` 对象的 `id` ，然后为储存产品的有序集创建一个 Redis 键（key），看起来像这样：`product:[id]:purchased_with`

`products_bought()` 方法检索被一起购买的产品列表（它们都属于同一个订单）。在这个方法中，我们执行以下几个任务：

	1. 得到所给 `Product` 对象的产品 ID
	2. 迭代所有的产品 ID。对于每个 `id` ，我们迭代所有的产品 ID 并且跳过所有相同的产品，这样我们就可以得到和每个产品一起购买的产品。
	3. 我们使用 `get_product_id()` 方法来获取 Redis 产品键。对于一个 ID 为 33 的产品，这个方法返回键 `product:33:purchased_with` 。这个键用于包含和这个产品被一同购买的产品 ID 的有序集。
	4. 我们把有序集中的每个产品 `id` 的评分加一。评分表示另一个产品和所给产品一起购买的次数。

我们有一个方法来保存和对一同购买的产品评分。现在我们需要一个方法来检索被所给购物列表的一起购买的产品。把 `suggest_product_for()` 方法添加到 `Recommender` 类中：

```python
def suggest_products_for(self, products, max_results=6):
	product_ids = [p.id for p in products]
	if len(products) == 1:
		# only 1 product
		suggestions = r.zrange(
						self.get_product_key(product_ids[0]),
						0, -1, desc=True)[:max_results]
	else:
		# generate a temporary key
		flat_ids = ''.join([str(id) for id in product_ids])
		tmp_key = 'tmp_{}'.format(flat_ids)
		# multiple products, combine scores of all products
		# store the resulting sorted set in a temporary key
		keys = [self.get_product_key(id) for id in product_ids]
		r.zunionstore(tmp_key, keys)
		# remove ids for the products the recommendation is for
		r.zrem(tmp_key, *product_ids)
		# get the product ids by their score, descendant sort
		suggestions = r.zrange(tmp_key, 0, -1,
						desc=True)[:max_results]
		# remove the temporary key
		r.delete(tmp_key)
	suggested_products_ids = [int(id) for id in suggestions]

	# get suggested products and sort by order of appearance
	suggested_products = list(Product.objects.filter(id__in=suggested_
	products_ids))
	suggested_products.sort(key=lambda x: suggested_products_ids.
	index(x.id))
	return suggested_products
```

`suggest_product_for()` 方法接收下列参数：

* `products`：这是一个 `Product` 对象列表。它可以包含一个或者多个产品
* `max_results`：这是一个整数，用于展示返回的推荐的最大数量

在这个方法中，我们执行以下的动作：

	1. 得到所给 `Product` 对象的 ID
	2. 如果只有一个产品，我们就检索一同购买的产品的 ID，并按照他们的购买时间来排序。这样做，我们就可以使用 Redis 的 `ZRANGE` 命令。我们通过 `max_results` 属性来限制结果的数量。
	3. 如果有多于一个的产品被给予，我们就生成一个临时的和产品 ID 一同创建的 Redis 键。
	4. 我们把包含在每个所给产品的有序集中东西的评分组合并相加，我们使用 Redis 的 `ZUNIONSTORE` 命令来实现这个操作。`ZUNIONSTORE` 命令执行了对有序集的所给键的求和，然后在新的 Redis 键中保存每个元素的求和。你可以在这里阅读更多关于这个命令的信息：http://redisio/commands/ZUNIONSTORE 。我们在临时键中保存分数的求和。
	5. 因为我们已经求和了评分，我们或许会获取我们推荐的重复商品。我们就使用 `ZREM` 命令来把他们从生成的有序集中删除。
	6. 我们从临时键中检索产品 ID，使用 `ZREM` 命令来按照他们的评分排序。我们把结果的数量限制在 `max_results` 属性指定的值内。然后我们删除了临时键。
	7. 最后，我们用所给的 `id` 获取 `Product` 对象，并且按照同样的顺序来排序。

为了更加方便使用，让我们添加一个方法来清除所有的推荐。
把下列方法添加进 `Recommender` 类中：

```python
def clear_purchases(self):
	for id in Product.objects.values_list('id', flat=True):
		r.delete(self.get_product_key(id))
```

让我们试试我们的推荐引擎。确保你在数据库中引入了几个 `Product` 对象并且在 shell 中使用了下面的命令来初始化 Redis 服务器：

```
src/redis-server
```

打开另外一个 shell ，执行 `python manage.py shell` ，输入下面的代码来检索几个产品：

```
from shop.models import Product
black_tea = Product.objects.get(translations__name='Black tea')
red_tea = Product.objects.get(translations__name='Red tea')
green_tea = Product.objects.get(translations__name='Green tea')
tea_powder = Product.objects.get(translations__name='Tea powder')
```

然后，添加一些测试购物到推荐引擎中：

```
from shop.recommender import Recommender
r = Recommender()
r.products_bought([black_tea, red_tea])
r.products_bought([black_tea, green_tea])
r.products_bought([red_tea, black_tea, tea_powder])
r.products_bought([green_tea, tea_powder])
r.products_bought([black_tea, tea_powder])
r.products_bought([red_tea, green_tea])
```

我们已经保存了下面的评分：

```
black_tea: red_tea (2), tea_powder (2), green_tea (1)
red_tea: black_tea (2), tea_powder (1), green_tea (1)
green_tea: black_tea (1), tea_powder (1), red_tea(1)
tea_powder: black_tea (2), red_tea (1), green_tea (1)
```

让我们看看为单一产品的推荐吧：

```
>>> r.suggest_products_for([black_tea])
[<Product: Tea powder>, <Product: Red tea>, <Product: Green tea>]
>>> r.suggest_products_for([red_tea])
[<Product: Black tea>, <Product: Tea powder>, <Product: Green tea>]
>>> r.suggest_products_for([green_tea])
[<Product: Black tea>, <Product: Tea powder>, <Product: Red tea>]
>>> r.suggest_products_for([tea_powder])
[<Product: Black tea>, <Product: Red tea>, <Product: Green tea>]
```

正如你所看到的那样，推荐产品的顺序正式基于他们的评分。让我们来看看为多个产品的推荐吧：

```
>>> r.suggest_products_for([black_tea, red_tea])
[<Product: Tea powder>, <Product: Green tea>]
>>> r.suggest_products_for([green_tea, red_tea])
[<Product: Black tea>, <Product: Tea powder>]
>>> r.suggest_products_for([tea_powder, black_tea])
[<Product: Red tea>, <Product: Green tea>]
```

你可以看到推荐产品的顺序和评分总和的顺序是一样的。比如 `black_tea`, `red_tea`, `tea_powder(2+1)` 和 `green_tea(1=1)` 的产品推荐就是这样。

我们必须保证我们的推荐算法按照预期那样工作。让我们在我们的站点上展示我们的推荐吧。

编辑 `shop` 应用的 `views.py` ，添加以下库：

```python
from .recommender import Recommender
```

把下面的代码添加进 `product_detail` 视图（view）中的 `render()` 之前：

```python
r = Recommender()
recommended_products = r.suggest_products_for([product], 4)
```

我们得到了四个最多产品推荐。 `product_detail` 视图（view）现在看起来像这样：

```python
from .recommender import Recommender

def product_detail(request, id, slug):
	product = get_object_or_404(Product,
								id=id,
								slug=slug,
								available=True)
	cart_product_form = CartAddProductForm()
	r = Recommender()
	recommended_products = r.suggest_products_for([product], 4)
	return render(request,
					'shop/product/detail.html',
					{'product': product,
					'cart_product_form': cart_product_form,
					'recommended_products': recommended_products})
```

现在编辑 `shop` 应用的 `shop/product/detail.html` 模板（template），把以下代码添加在 ` {{ product.description|linebreaks }} ` 的后面：

```html
{% if recommended_products %}
<div class="recommendations">
<h3>{% trans "People who bought this also bought" %}</h3>
{% for p in recommended_products %}
<div class="item">
<a href="{{ p.get_absolute_url }}">
<img src="{% if p.image %}{{ p.image.url }}{% else %}{%
static "img/no_image.png" %}{% endif %}">
</a>
<p><a href="{{ p.get_absolute_url }}">{{ p.name }}</a></p>
</div>
{% endfor %}
</div>
{% endif %}
```

运行开发服务器，访问：http://127.0.0.1:8000/en/ 。点击一个产品来查看它的详情页。你应该看到展示在下方的推荐产品，就象这样：

![django-9-13](http://ohqrvqrlb.bkt.clouddn.com/django-9-13.png)

我们也将在购物车当中引入产品推荐。产品推荐将会基于用户购物车中添加的产品。编辑 `cart` 应用的 `views.py` ，添加以下库：

```python
from shop.recommender import Recommender
```

编辑 `cart_detail` 视图（view），让它看起来像这样：

```python
def cart_detail(request):
	cart = Cart(request)
	for item in cart:
		item['update_quantity_form'] = CartAddProductForm(
						initial={'quantity': item['quantity'],
								'update': True})
	coupon_apply_form = CouponApplyForm()
	
	r = Recommender()
	cart_products = [item['product'] for item in cart]
	recommended_products = r.suggest_products_for(cart_products,
	                            max_results=4)
	return render(request,
				'cart/detail.html',
				{'cart': cart,
				'coupon_apply_form': coupon_apply_form,
				'recommended_products': recommendeproducts})
```

编辑 `cart` 应用的 `cart/detail.html` ，把下列代码添加在 `<table>`HTML 标签后面：

```html
{% if recommended_products %}
<div class="recommendations cart">
<h3>{% trans "People who bought this also bought" %}</h3>
{% for p in recommended_products %}
<div class="item">
<a href="{{ p.get_absolute_url }}">
<img src="{% if p.image %}{{ p.image.url }}{% else %}{%
static "img/no_image.png" %}{% endif %}">
</a>
<p><a href="{{ p.get_absolute_url }}">{{ p.name }}</a></p>
</div>
{% endfor %}
</div>
{% endif %}
```

访问 http://127.0.0.1:8000/en/ ，在购物车当中添加一些商品。东航拟导航向 http://127.0.0.1:8000/en/cart/ 时，你可以在购物车下看到锐减的产品，就像下面这样：

![django-9-14](http://ohqrvqrlb.bkt.clouddn.com/django-9-14.png)

恭喜你！你已经使用 Django 和 Redis 构建了一个完整的推荐引擎。

##**总结**
在这一章中，你使用会话创建了一个优惠券系统。你学习了国际化和本地化是如何工作的。你也使用 Redis 构建了一个推荐引擎。

在下一章中，你将开始一个新的项目。你将会用 Django 创建一个在线学习平台，并且使用基于类的视图（view）。你将学会创建一个定制的内容管理系统（CMS）。

（译者@夜夜月：- -下一章又轮到我了。。。。。。放出时间不固定。。。。。。）

