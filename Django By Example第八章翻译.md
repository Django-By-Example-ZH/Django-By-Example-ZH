书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

（译者注：还有4章！还有4章全书就翻译完成了！）

#第八章

## 管理付款和订单

在上一章，你创建了一个基础的在线商店包含一个产品列表以及订单系统。你还学习了如何执行异步的任务通过使用Celery。在这一章中，你会学习到如何集成一个支付网关**（译者注：支付网关（Payment Gateway）是银行金融网络系统和Internet网络之间的接口，是由银行操作的将Internet上传输的数据转换为金融机构内部数据的一组服务器设备，或由指派的第三方处理商家支付信息和顾客的支付指令。以上是我百度的。）**到你的站点中。你还会扩展管理平台站点来管理订单和用不同的格式导出它们。

在这一章中，我们会覆盖以下几点：

* 集成一个支付网关到你的站点中
* 管理支付通知
* 导出订单为CSV格式
* 创建定制视图给管理页面
* 动态的生成PDF支票

##集成一个支付网关

一个支付网关允许你在线处理支付。通过使用一个支付网关，你可以管理顾客的订单以及委托一个可靠的，安全的第三方处理支付。这意味着你无需担心存储信用卡信息到你的系统中。

PayPal 提供了多种方法来集成它的网管到你的站点中。标准的集成由一个*Buy now*按钮组成，这个按钮你可以已经在别的网站见到过（译者注：国内还是支付宝和微信比较多）。这个按钮会重定向购买者到PayPal去处理支付。我们将要集成PayPal支付标准包含一个定制的*Buy now*按钮到我们的站点中。PayPal将会处理支付并且发送一个消息通知给我们的服务指明该笔支付的状态。

##创建一个PayPal账户

你需要有一个PayPal商业账户来集成支付网关到你的站点中。如果你还没有一个PayPal账户，去 https://www.paypal.com/signup/account 注册。确保你选择了一个*Bussiness Account*并且注册成为PayPal支付标准解决方案，如下图所示：

![django-8-0](http://ohqrvqrlb.bkt.clouddn.com/django-8-0.png)

填写你的详情在注册表单中并且完成注册流程。PayPal会发送给你一封e-mail来核对你的账户。

##安装django-paypal

Django-paypal是一个第三方django应用，它可以简化集成PayPal到Django项目中。我们将要使用它来集成PayPal支付标准解决方案到我们的商店中。你可以找到django-paypal的文档，访问 http://django-paypal.readthedocs.org/。

安装django-paypal在shell中通过以下命令：

    pip install django-paypal==0.2.5 
    
**(译者注：现在应该有最新版本，书上使用的是0.2.5版本)**

编辑你的项目中的*settings.py*文件，添加'paypal.standard.ipn'到*INSTALLED_APPS*设置中，如下所示：

```python
INSTALLED_APPS = (    # ...    'paypal.standard.ipn',)
```
这个应用提供自django-paypal来集成PayPal支付标准通过**Instant Payment Notification(IPN)**。我们之后会操作支付通知。

添加以下设置到*myshop*的*settings.py*文件来配置django-paypal：

```python
# django-paypal settings
PAYPAL_RECEIVER_EMAIL = 'mypaypalemail@myshop.com'PAYPAL_TEST = True
```

以上两个设置含义如下：

* PAYPAL_RECEIVER_EMAIL：你的PayPal账户e-mail。使用你创建的PayPal账户e-mail替换*mypaypalemail@myshop.com*。
* PAYPAL_TEST：一个布尔类型指示是否PayPal的沙箱环境，该环境可以用来处理支付。这个沙箱允许你测试你的PayPal集成在迁移到一个正式生产的环境之前。

打开shell运行如下命令来同步django-paypal的模型（models）到数据库中：

    python manage.py migrate
    
你会看到如下类似的输出：

```shell
Running migrations:    Rendering model states... DONE    Applying ipn.0001_initial... OK    Applying ipn.0002_paypalipn_mp_id... OK    Applying ipn.0003_auto_20141117_1647... OK
```

django-paypal的模型（models）如今已经同步到了数据库中。你还需要添加django-paypal的URL模式到你的项目中。编辑主的*urls.py*文件，该文件位于*myshop*目录，然后添加以下的URL模式。记住粘贴该URL模式要在*shop.urls*模式之前为了避免错误的模式匹配：

    url(r'^paypal/', include('paypal.standard.ipn.urls')),
    
让我们添加支付网关到结账流程中。

##添加支付网关

结账流程工作如下：

* 1.用户添加物品到他们的购物车中
* 2.用户结账他们的购物车
* 3.用户被重定向到PayPal进行支付
* 4.PayPal发送一个支付通知给我们的站点
* 5.PayPal重定向用户回到我们的网站

创建一个新的应用到你的项目中使用如下命令：

    python manage.py startapp payment
    
我们将要使用这个应用去管理结账过程和用户支付。

编辑你的项目的*settings.py*文件，添加'payment'到*INSTALLED_APPS*设置中，如下所示：

```python
INSTALLED_APPS = (    # ...    'paypal.standard.ipn',    'payment',)
```

*payment*应用现在已经在项目中激活。编辑*orders*应用的*views.py*文件并且确保包含以下导入：

```python
from django.shortcuts import render, redirectfrom django.core.urlresolvers import reverse
```

替换以下*order_create*视图（view）的内容：

```python# launch asynchronous taskorder_created.delay(order.id)return render(request, 'orders/order/created.html', locals())
```

新的内容为：

```python
# launch asynchronous taskorder_created.delay(order.id) # set the order in the session
request.session['order_id'] = order.id # redirect to the payment
return redirect(reverse('payment:process'))
```

在成功的创建一个新的订单之后，我们设置这个订单ID到当前的会话中使用*order_id*会话键（session key）。之后，我们重定向用户到`payment:process`URL，这个我们下一步就是创建。

编辑*payment*应用的*views.py*文件然后添加如下代码：

```python
from decimal import Decimalfrom django.conf import settingsfrom django.core.urlresolvers import reversefrom django.shortcuts import render, get_object_or_404from paypal.standard.forms import PayPalPaymentsFormfrom orders.models import Order
def payment_process(request):    order_id = request.session.get('order_id')    order = get_object_or_404(Order, id=order_id)    host = request.get_host()    paypal_dict = {        'business': settings.PAYPAL_RECEIVER_EMAIL,        'amount': '%.2f' % order.get_total_cost().quantize(                                                Decimal('.01')),        'item_name': 'Order {}'.format(order.id),        'invoice': str(order.id),        'currency_code': 'USD',        'notify_url': 'http://{}{}'.format(host,                                        reverse('paypal-ipn')),        'return_url': 'http://{}{}'.format(host,                                        reverse('payment:done')),        'cancel_return': 'http://{}{}'.format(host,
                                    reverse('payment:canceled')),       }       form = PayPalPaymentsForm(initial=paypal_dict)       return render(request,                     'payment/process.html',                     {'order': order, 'form':form})
```

在`payment_process`视图（view）中，我们生成了一个PayPal的**Buy now**按钮用来支付一个订单。首先，我们拿到当前的订单从`order_id`会话键中，这个键值被之前的`order_create`视图（view）设置。我们拿到这个`order`对象通过给予的ID并且构建一个新的`PayPalPaymentsForm`，该表单表单包含以下字段：

* business：PayPal商业账户用来处理支付。我们使用e-mail账户，该账户定义在`PAYPAL_RECEIVER_EMAIL`设置那里。
* amount：向顾客索要的总价。
* item_name：正在出售的商品名。我们使用订单ID，因为订单可能包含很多产品。
* currency_code：本次支付的货币。我们设置这里为USD使用U.S. Dollar**(译者注：传说中的美金)**。需要使用相同的货币，该货币被设置在你的PayPal账户中（例如：EUR 对应欧元）。
* notify_url：这个URL PayPal将会发送IPN请求过去。我们使用django-paypal提供的`paypal-ipn` URL。这个视图（view）与这个URL关联来操作支付通知以及存储它们到数据库中。
* return_url：这个URL用来重定向用户当他的支付成功之后。我们使用URL `payment:done`，这个我们接下来会创建。
* cancel_return：这个URL用来重定向用户如果这个支付被取消或者有其他问题。我们使用URL `payment:canceled`，这个我们接下来会创建。

`PayPalpaymentsForm`将会被渲染成一个标准表单带有隐藏的字段，并且用户将来只能看到**Buy now**按钮。当用户点击该按钮，这个表单将会提交到PayPal通过POST渠道。

让我们创建简单的视图（views）给PayPal用来重定向用户当支付成功，或者当支付被取消因为某些原因。添加以下代码到相同的*views.py*文件：

```python
from django.views.decorators.csrf import csrf_exempt
@csrf_exemptdef payment_done(request):    return render(request, 'payment/done.html')
    @csrf_exemptdef payment_canceled(request):    return render(request, 'payment/canceled.html')
```

我们使用`csrf_exempt`装饰器来避免Django期待一个CSRF标记，因为PayPal能重定向用户到以上两个视图（views）通过POST渠道。创建新的文件在*payment*应用目录下并且命名为*urls.py*。添加以下代码：

```python
from django.conf.urls import urlfrom . import viewsurlpatterns = [    url(r'^process/$', views.payment_process, name='process'),    url(r'^done/$', views.payment_done, name='done'),    url(r'^canceled/$', views.payment_canceled, name='canceled'),]
```

这些URL是给支付工作流的。我们已经包含了以下URL模式：

* process：给这个视图（view）用来生成PayPal表单给**Buy now**按钮。
* done：给PayPal用来重定向用户当支付成功的时候。
* canceled：给PayPal用来重定向用户当支付取消的时候。

编辑主的*myshop*项目的*urls.py*文件，包含URL模式给*payment*应用：

    url(r'^payment/', include('payment.urls',namespace='payment')),
    
记住粘贴以上内容在`shop.urls`模式之前用来避免错误的模式匹配。

创建以下文件建构在*payment*应用目录下：

```shell
templates/    payment/        process.html        done.html        canceled.html
```

编辑*payment/process.html*模板（template）并且添加以下代码：

```html
{% extends "shop/base.html" %}
{% block title %}Pay using PayPal{% endblock %}
{% block content %}  <h1>Pay using PayPal</h1>  {{ form.render }}{% endblock %}
```

这个模板（template）会渲染`PayPalPaymentsForm`并且展示**Buy now**按钮。

编辑*payment/done.html*模板（template）并且添加如下代码：

```html
{% extends "shop/base.html" %}{% block content %}    <h1>Your payment was successful</h1>    <p>Your payment has been successfully received.</p>{% endblock %}
```

这个模板（template）的页面给用户重定向当成功支付之后。

编辑*payment/canceled.html*模板（template）并且添加以下代码：

```html
{% extends "shop/base.html" %}{% block content %}    <h1>Your payment has not been processed</h1>    <p>There was a problem processing your payment.</p>{% endblock %}
```

这个模板（template）的页面给用户重定向当有这个支付过程出现问题或者用户取消了这次支付。

让我们尝试完成的支付过程。

##使用PayPal的沙箱

打开 http://developer.paypal.com 在你的浏览器中然后进行登录使用你的PayPal商业账户。点击**Dashboard**菜单项，在左方菜单点击**Accounts**选项在**Sandbox**下方。你会看到你的沙箱测试账户列，如下所示：

![django-8-1](http://ohqrvqrlb.bkt.clouddn.com/django-8-1.png)

一开始，你将会看到一个商业以及一个个人测试账户由PayPal动态创建。你可以创建新的沙箱测试账户通过使用**Create Account**按钮。

点击**Personal Account**在列中扩大它，之后点击**Profile**链接。你会看到一些信息关于这个测试账户包含e-mail和profile信息，如下所示：

![django-8-2](http://ohqrvqrlb.bkt.clouddn.com/django-8-2.png)

在**Funding** tab中，你会找到银行账户，信用卡日期，以及PayPal信用余额。

这些测试账户能够被用来做支付在你的网站中当使用沙箱环境。跳转到**Profile** tab然后点击**Change password**链接。创建一个定制密码给这个测试账户。

打开shell并且启动开发服务器使用命令`python manage.py runserver`。打开 http://127.0.0.1:8000 在你的浏览器中，添加一些产品到购物车中，并且填写结账表单。当你点击**Place order**按钮，这个订单会被保存在数据库中，这个订单ID会被保存在当前的会话中，并且你会被重定向到支付处理页面。这个页面从会话中获取订单并且渲染PayPal表单显示一个**Buy now**按钮，如下所示：

![django-8-3](http://ohqrvqrlb.bkt.clouddn.com/django-8-3.png)

你可以看下HTML源码来看下生成的表单字段。

点击**Buy now**按钮。你会被重定向到PayPal，并且你会看到如下页面：

![django-8-4](http://ohqrvqrlb.bkt.clouddn.com/django-8-4.png)

输入购买者测试账户e-mail和密码然后点击**Log In**按钮。你会被重定向到以下页面：

![django-8-5](http://ohqrvqrlb.bkt.clouddn.com/django-8-5.png)

现在，点击**Pay now**按钮。最后，你会看到批准页面该页面包含你的交易ID。这个页面看上去如下所示：

![django-8-6](http://ohqrvqrlb.bkt.clouddn.com/django-8-6.png)

点击**Return to e-mail@domain.com**按钮。你会被重定向到的URL是你之前在`PayPalPaymentsForm`中的`return_url`字段中定义的。这个URL对应`payment_done`视图（view）。这个页面看上去如下所示：

![django-8-7](http://ohqrvqrlb.bkt.clouddn.com/django-8-7.png)

这个支付已经成功了。然而，PayPal并没有发送一个支付状态通知给我们的应用，因为我们运行我们的项目在我们本地主机，IP是 127.0.0.1 这并不是一个公开地址。我们将要学习如何使我们的站点可以从Internet访问并且接收IPN通知。

##获取支付通知

IPN是一个方法提供自大部分的支付网关用来跟踪实时的购买。一个通知会立即发送到你的服务当这个网关处理了一个支付。这个通知包含所有支付详情，包括状态以及一个支付的签名，该签名可以用来确定这个消息的来源点。这个消息被发送通过一个单独的HTTP请求给你的服务。在出现连接问题的情况下，PayPal将会多次企图通知你的站点。

django-paypal应用内置两种不同的信号给IPNs。如下：

* valid_ipn_received：会被触发当IPN信息获取自PayPal是正确的并且不是一个已存在数据库中的消息的复制。
* invalid_ipn_received：这个信号会触发当IPN获取自PayPal包含无效的数据或者不是一个良好的形式。

我们将要创建一个定制的接受函数并且连接它给`valid_ipn_received`信号用来确定支付。

创建新的文件在*payment*应用目录下，并且命名为*signals.py*，添加如下代码：

```python
from django.shortcuts import get_object_or_404from paypal.standard.models import ST_PP_COMPLETEDfrom paypal.standard.ipn.signals import valid_ipn_receivedfrom orders.models import Order

def payment_notification(sender, **kwargs):    ipn_obj = sender    if ipn_obj.payment_status == ST_PP_COMPLETED:        # payment was successful        order = get_object_or_404(Order, id=ipn_obj.invoice)        # mark the order as paid        order.paid = True        order.save()
        valid_ipn_received.connect(payment_notification)
```

我们连接`payment_notification`接收函数给django-paypal提供的`valid_ipn_received`信号。这个接收函数工作如下：

* 1.我们获取发送对象，该对象是一个*PayPalIPN*模型的实例，位于`paypal.standard.ipn.models`。
* 2.我们检查`payment_status`属性来确保它和django-payapl的完整状态相同。这个状态指示这个支付已经成功处理。
* 3.之后我们使用`get_object_or_404()`快捷函数来拿到订单，该订单的ID匹配`invoice`参数我们之前提供给PayPal。
* 4.我们备注这个订单已经支付通过设置它的`paid`属性为`True`并且保存这个订单对象到数据库中。

你需要确保你的信号方法已经加载，这样这个接收函数会被调用当`valid_ipn_received`信号被触发的时候。The best practice is to load your signals when the application containing them is loaded. **（译者注：谁帮我翻一下，好拗口啊）**。这能够实现通过定义一个定制应用配置，这方面会在下一节进行解释。

##配置我们的应用

你已经学习了关于应用的配置在*第六章 跟踪用户操作*。我们将要定义一个定制配置给我们的*payment*应用为了加载我们的信号接收函数。

创建一个新的文件在*payment*应用目录下命名为*apps.py*。添加如下代码：

```python
from django.apps import AppConfig
class PaymentConfig(AppConfig):    name = 'payment'
    verbose_name = 'Payment'    
    def ready(self):        # import signal handlers        import payment.signals
```

在上述代码中，我们定义了一个定制`AppConfif`类给*payment*应用。`name`参数是这个应用的名字，*verbose_name*包含可读的样式。我们导入信号方法在`ready()`方法中确保它们会被加载当这个应用初始化的时候。

编辑*payment*应用的*__init__.py*文件，添加以下行：

    default_app_config = 'payment.apps.PaymentConfig'
    
以上操作可以使Django动态加载你的定制应用配置类。你可以找到更进一步的信息关于应用配置，通过访问 https://docs.djangoproject.com/en/1.8/ref/applications/ 。

##测试支付通知

由于我们工作在本地环境中，我们需要确保我们的站点可以被PayPal获得。有不少应用允许你使你的开发环境在Internet中可获得。我们将要使用Ngrok，它就是其中一个最著名的。

    ./ngrok http 8000
    
通过这条命名，你告诉Ngrok去创建一条隧道给你的本地主机在端口8000上并且分配一个Internet可访问主机名给它。你可以看到如下类似输出：

```shell
Tunnel Status     onlineVersion           2.0.17/2.0.17Web Interface     http://127.0.0.1:4040Forwarding        http://1a1b50f2.ngrok.io -> localhost:8000Forwarding        https://1a1b50f2.ngrok.io -> localhost:8000
Connnections      ttl     opn     rt1     rt5     p50     p90                  0       0       0.00    0.00    0.00    0.00
```

Ngrok告诉我们关于我们的站点，运行在本地8000端口使用Django开发服务器，已经可以在Internet访问到通过URLs http://1a1b50f2.ngrok.io 以及 https://1a1b50f2.ngrok.io ，前者是HTTP，后者是HTTPS。Ngrok还提供一个URL来访问一个web接口用来显示信息关于发送到这个服务的请求。

打开Ngrok提供的URL在浏览器中；例如，http://1a1b50f2.ngrok.io 。添加一些产品到购物车中，放置一个订单，然后使用你的PayPal测试账户进行支付。这个时候，PayPal将能够拿到这个URL，这个URL由`PayPalPaymentsForm`的`notify_url`字段生成，在`payment_process`视图（view）中。如果你看一下这个渲染过的表单，你会看到这个HTML表单字段看上去如下所示：

```html
<input id="id_notify_url" name="notify_url" type="hidden"value="http://1a1b50f2.ngrok.io/paypal/">
```

在结束支付过程之后，打开 http://127.0.0.1:8000/admin/ipn/paypalipn/ 在你的浏览器中。你会看到一个IPN对象对应最新的支付状态为**Completed**。这个对象包含所有的支付信息，该对象由PayPal发送给你提供给IPN通知的URL。IPN管理列展示页面看上去如下所示：

![django-8-8](http://ohqrvqrlb.bkt.clouddn.com/django-8-8.png)

你还可以启动IPNs通过使用PayPal的IPN模拟器位于 https://developer.paypal.com/developer/ipnSimulator/ 。这个模拟器允许你指定字段和发送的通知类型。

除了PayPal支付标准外，PayPal提供Website Payments Pro，它是一个订购服务允许你接受支付在你的站点中而不需要重定向用户到PayPal。你可以找到更多信息关于如何集成Website Payments Pro，通过访问 http://django-paypal.readthedocs.org/en/v0.2.5/pro/index.html。

##导出订单为CSV文件

有时候，你可能想要导出包含在模型的信息到一个文件中，这样你可以导入它到其他的系统中。其中一个范围最广的格式用来导出/导入数据就是**Comma-Separated Values(CSV)**。一个CSV文件就是一个纯文本文件包含若干记录。There is usually one record per line, and some delimiter character, usually a literal comma, separates the record fields**（译者注：求翻译。。。）**                                                                                                                                     。我们将要定制管理平台站点能够导出订单为CSV文件。

##添加定制操作到管理平台站点中

Django提供你多种不同的选项来定制管理平台站点。我们将要修改对象列视图（view）来包含一个定制的管理操作。

一个管理操作工作如下：一个用户选择对象从管理对象列页面通过复选框，之后选择一个操作去执行在所有被选择的项上，然后执行该操作。以下
-------
展示操作会位于管理页面的哪个地方：

![django-8-9](http://ohqrvqrlb.bkt.clouddn.com/django-8-9.png)

> 创建定制管理操作允许管理人员一次性应用操作多个元素。

你可以创建一个定制操作通过编写一个经常性的函数获取以下参数：

* 当前展示的*ModelAdmin*
* 当前请求对象，一个*HttpRequest*实例
* 一个查询集（QuerySet）给用户所选择的对象

这个函数将会被执行当这个操作被触发在管理平台站点上。

我们将要创建一个定制管理操作来下载订单列表的CSV文件。编辑*orders*应用的*admin.py*文件，添加如下代码在`OrderAdmin`类之前：

```python
import csvimport datetimefrom django.http import HttpResponsedef export_to_csv(modeladmin, request, queryset):
    opts = modeladmin.model._meta    response = HttpResponse(content_type='text/csv')    response['Content-Disposition'] = 'attachment; \           filename={}.csv'.format(opts.verbose_name)    writer = csv.writer(response)    fields = [field for field in opts.get_fields() if not field.many_to_many and not field.one_to_many]    # Write a first row with header information    writer.writerow([field.verbose_name for field in fields])    # Write data rows    for obj in queryset:        data_row = []        for field in fields:            value = getattr(obj, field.name)            if isinstance(value, datetime.datetime):                value = value.strftime('%d/%m/%Y')            data_row.append(value)        writer.writerow(data_row)    return responseexport_to_csv.short_description = 'Export to CSV'
```

在这代码中，我们执行以下任务：

* 1.我们创建一个`HttpResponse`实例包含一个定制`text/csv`内容类型来告诉浏览器这个响应需要处理为一个CSV文件。我们还添加一个`Content-Disposition`头来指示这个HTTP响应包含一个附件。
* 2.我们创建一个CSV `writer`对象，该对象将会被写入`response`对象。
* 3.我们动态的获取`model`字段通过使用模型（moedl）`_meta`选项的`get_fields()`方法。我们排除多对多以及一对多的关系。
* 4.我们编写了一个头行包含字段名。
* 5.我们迭代给予的查询集（QuerySet）并且为每一个查询集中返回的对象写入行。我们注意格式化`datetime`对象因为这个输出值给CSV必须是一个字符串。
* 6.我们定制这个操作的显示名在模板（template）中通过设置一个`short_description`属性给这个函数。

我们已经创建了一个普通的管理操作可以添加到任意的*ModelAdmin*类。

最后，添加新的`export_to_csv`管理操作给`OrderAdmin`类如下所示：

```python
class OrderAdmin(admin.ModelAdmin):    # ...    actions = [export_to_csv]
```

打开 http://127.0.0.1:8000/admin/orders/order/ 在你的浏览器中。管理操作看上去如下所示：

![django-8-10](http://ohqrvqrlb.bkt.clouddn.com/django-8-10.png)

选择一些订单然后选择**Export to CSV**操作从下拉选框中，之后点击**Go**按钮。你的浏览器会下载生成的CSV文件名为*order.csv*。打开下载的文件使用一个文本编辑器。你会看到的内容如以下的格式，包含一个头行以及你之前选择的每行订单对象：

```
ID,first name,last name,email,address,postalcode,city,created,updated,paid3,Antonio,Melé,antonio.mele@gmail.com,Bank Street 33,WS J11,London,25/05/2015,25/05/2015,False...
```

如你所见，创建管理操作是非常简单的。

##扩展管理站点通过定制视图（view）

有时候你可能想要定制管理平台站点，比如处理*ModelAdmin*的配置，管理操作的创建，以及覆盖管理模板（templates）。在这样的场景中，你需要创建一个定制的管理视图（view）。通过一个定制的管理视图（view），你可以构建任何你需要的功能。你只需要确保只有管理用户能访问你的视图并且你维护这个管理的外观和感觉通过你的模板（template）扩展自一个管理模板（template）。

让我们创建一个定制视图（view）来展示关于一个订单的信息。编辑*orders*应用下的*views.py*文件，添加以下代码：

```python
from django.contrib.admin.views.decorators import staff_member_requiredfrom django.shortcuts import get_object_or_404from .models import Order
@staff_member_requireddef admin_order_detail(request, order_id):    order = get_object_or_404(Order, id=order_id)    return render(request,                  'admin/orders/order/detail.html',                  {'order': order})
```

这个`staff_member_required`装饰器检查用户请求这个页面的`is_active`以及`is_staff`字段是被设置为`True`。在这个视图（view）中，我们获取`Order`对象通过给予的id以及渲染一个模板来展示这个订单。

现在，编辑*orders*应用中的*urls.py*文件并且添加以下URL模式：

```python
url(r'^admin/order/(?P<order_id>\d+)/$',    views.admin_order_detail,    name='admin_order_detail'),
```

创建以下文件结构在*orders*应用的*templates/*目录下：

```shell
admin/    orders/        order/            detail.html
```

编辑*detail.html*模板（template），添加以下内容：

```html
{% extends "admin/base_site.html" %}{% load static %}
{% block extrastyle %}     <link rel="stylesheet" type="text/css" href="{% static "css/admin.css" %}" />{% endblock %}
{% block title %}     Order {{ order.id }} {{ block.super }}{% endblock %}
{% block breadcrumbs %}  <div class="breadcrumbs">    <a href="{% url "admin:index" %}">Home</a> &rsaquo;    <a href="{% url "admin:orders_order_changelist" %}">Orders</a>
    &rsaquo;    <a href="{% url "admin:orders_order_change" order.id %}">Order {{ order.id }}</a>    &rsaquo; Detail  </div>{% endblock %}
{% block content %}  <h1>Order {{ order.id }}</h1>  <ul class="object-tools">    <li>      <a href="#" onclick="window.print();">Print order</a>    </li> 
  </ul>  <table> 
    <tr>      <th>Created</th>      <td>{{ order.created }}</td>    </tr>    <tr>      <th>Customer</th>      <td>{{ order.first_name }} {{ order.last_name }}</td>    </tr> 
    <tr>      <th>E-mail</th>      <td><a href="mailto:{{ order.email }}">{{ order.email }}</a></td>    </tr>    <tr>
    <th>Address</th>    <td>{{ order.address }}, {{ order.postal_code }} {{ order.city}}</td>  </tr> 
    <tr>      <th>Total amount</th>      <td>${{ order.get_total_cost }}</td>    </tr>    <tr>      <th>Status</th>      <td>{% if order.paid %}Paid{% else %}Pending payment{% endif %}</td> 
    </tr>  </table>
    <div class="module">    <div class="tabular inline-related last-related">      <table>        <h2>Items bought</h2>        <thead>          <tr>            <th>Product</th>            <th>Price</th>            <th>Quantity</th>            <th>Total</th>          </tr>        </thead>        <tbody>          {% for item in order.items.all %}            <tr class="row{% cycle "1" "2" %}">              <td>{{ item.product.name }}</td>              <td class="num">${{ item.price }}</td>              <td class="num">{{ item.quantity }}</td>              <td class="num">${{ item.get_cost }}</td>            </tr>          {% endfor %}          <tr class="total">            <td colspan="3">Total</td>            <td class="num">${{ order.get_total_cost }}</td>          </tr>        </tbody>      </table>    </div>  </div>{% endblock %}
```

这个模板（template）是用来显示一个订单详情在管理平台站点中。这个模板（template）扩展Djnago的管理平台站点的*admin/base_site.html*模板，它包含管理的主要HTML结构和CSS样式。我们加载定制的静态文件*css/admin.css*。

为了使用静态文件，你需要拿到它们从这章教程的实例代码中。复制位于*orders*应用的*static/*目录下的静态文件然后添加它们到你的项目的相同位置。

我们使用定义在父模板（template）的区块包含我们自己的内容。我们展示信息关于订单和购买的商品。

当你想要扩展一个管理模板（template），你需要知道它的结构以及确定存在的区块。你可以找到所有管理模板（template），通过访问 https://github.com/django/django/tree/1.8.6/django/contrib/admin/templates/admin 。

你也可以重写一个管理模板（template）如果你需要的话。为了重写一个管理模板（template），拷贝它到你的*template*目录保持相同的相对路径以及文件名。Django管理平台站点将会使用你的定制模板（template）替代默认的模板。

最后，让我们添加一个链接给每个*Order*对象在管理平台站点的列展示页面。编辑*orders*应用的*admin.py*文件然后添加以下代码，在`OrderAdmin`类上面：

```python
from django.core.urlresolvers import reversedef order_detail(obj):    return '<a href="{}">View</a>'.format(        reverse('orders:admin_order_detail', args=[obj.id]))order_detail.allow_tags = True
```

这个函数需要一个*Order*对象作为参数并且返回一个HTML链接给`admind_order_detail` URL。Django会避开默认的HTML输出。我们必须设置`allow_tags`属性为`True`来避开auto-escaping。

> 设置`allow_tags`属性为`True`来避免HTML-escaping在一些*Model*方法，*ModelAdmin*方法，以及任何其他的调用中。当你使用`allow_tags`的时候，能确保避开用户输入的跨域脚本。

之后，编辑`OrderAdmin`类来展示链接：

```python
class OrderAdmin(admin.ModelAdmin):    list_display = ['id',
                    'first_name', 
                    # ... 
                    'updated', 
                    order_detail]
```

打开 http://127.0.0.1:8000/admin/orders/order/ 在你的浏览器中。每一行现在都会包含一个**View**链接如下所示：

![django-8-11](http://ohqrvqrlb.bkt.clouddn.com/django-8-11.png)

点击某个订单的**View**链接来加载定制订单详情页面。你会看到一个页面如下所示：

![django-8-12](http://ohqrvqrlb.bkt.clouddn.com/django-8-12.png)

##生成动态的PDF发票

如今我们已经有了一个完整的结账和支付系统，我们可以生成一张PDF发票给每个订单。有几个Python库可以生成PDF文件。一个最流行的生成PDF的Python库是Reportlab。你可以找到关于如何使用Reportlab输出PDF文件的信息，通过访问 https://docs.djangoproject.com/en/1.8/howto/outputting-pdf/ 。

在大部分的场景中，你还需要添加定制样式和格式给你的PDF文件。你会发现渲染一个HTML模板（template）以及转化该模板（template）为一个PDF文件更加的方便，保持Python远离表现层。我们要遵循这个方法并且使用一个模块来生成PDF文件通过Django。我们将要使用WeasyPrint，它是一个Python库可以生成PDF文件从HTML模板中。

##安装WeasyPrint

首先，安装WeasyPrint的依赖给你的OS，这些依赖你可以找到通过访问 http://weasyprint.org/docs/install/#platforms 。

之后，安装WeasyPrint通过*pip*渠道使用如下命令：

    pip install WeasyPrint==0.24
    
##创建一个PDF模板（template）

我们需要一个HTML文档给WeasyPrint输入。我们将要创建一个HTML模板（template），渲染它使用Django，并且传递它给WeasyPrint来生成PDF文件。

创建一个新的模板（template）文件在*orders*应用的*templates/orders/order/目录下命名为*pdf.html*。添加如下内容：

```html
<html><body>     <h1>My Shop</h1>     <p>       Invoice no. {{ order.id }}</br>       <span class="secondary">         {{ order.created|date:"M d, Y" }}       </span>     </p>
     <h3>Bill to</h3>     <p>       {{ order.first_name }} {{ order.last_name }}<br>       {{ order.email }}<br>       {{ order.address }}<br>       {{ order.postal_code }}, {{ order.city }}     </p>
     <h3>Items bought</h3>     <table>       <thead> 
         <tr>           <th>Product</th>           <th>Price</th>           <th>Quantity</th>           <th>Cost</th>         </tr>       </thead>       <tbody>         {% for item in order.items.all %}           <tr class="row{% cycle "1" "2" %}">             <td>{{ item.product.name }}</td>             <td class="num">${{ item.price }}</td>             <td class="num">{{ item.quantity }}</td>             <td class="num">${{ item.get_cost }}</td>           </tr>         {% endfor %}         <tr class="total">           <td colspan="3">Total</td>           <td class="num">${{ order.get_total_cost }}</td>         </tr>       </tbody>     </table>
     
     <span class="{% if order.paid %}paid{% else %}pending{% endif %}">       {% if order.paid %}Paid{% else %}Pending payment{% endif %}     </span></body></html>
```

这个模板（template）就是PDF发票。在这个模板（template）中，我们展示所有订单详情以及一个HTML `<table>` 元素包含所有商品。我们还包含了一条消息来展示如果该订单已经支付或者支付还在进行中。

##渲染PDF文件

我们将要创建一个视图（view）来生成PDF发票给存在的订单通过使用管理平台站点。编辑*order*应用的*views.py*文件添加如下代码：

```python
from django.conf import settingsfrom django.http import HttpResponsefrom django.template.loader import render_to_stringimport weasyprint
@staff_member_requireddef admin_order_pdf(request, order_id):    order = get_object_or_404(Order, id=order_id)    html = render_to_string('orders/order/pdf.html',                            {'order': order})    response = HttpResponse(content_type='application/pdf')    response['Content-Disposition'] = 'filename=\           "order_{}.pdf"'.format(order.id)    weasyprint.HTML(string=html).write_pdf(response,        stylesheets=[weasyprint.CSS(            settings.STATIC_ROOT + 'css/pdf.css')])    return response
```

这个视图（view）用来生成一个PDF发票给一个订单。我们使用`staff_member_required`装饰器来确保只有管理人员能够访问这个视图（view）。我们获取*Order*对象通过给予的ID并且我们使用`rander_to_string()`函数提供自Django来渲染*orders/order/pdf.html*。这个渲染过的HTML会被保存到`html`变量中。之后，我们生成一个新的`HttpResponse`对象指定`application/pdf`的内容类型并且包含`Content-Disposition`头来指定这个文件名。我们使用WeasyPrint来生成一个PDF文件从渲染的HTML代码中并且将该文件写入`HttpResponse`对象中。我们加载它从本地路径通过使用`STATIC_ROOT`设置。最后，我们返回这个生成的响应。

由于我们需要使用`STATIC_ROOT`设置，我们需要添加它到我们的项目中。这个项目将会是静态文件的所在地。编辑*myshop*项目的*settings.py*文件，添加如下设置：

    STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
    
之后，运行命令`python manage.py collectstatic`。你会在输出末尾看到如下输出：

```shell
You have requested to collect static files at the destinationlocation as specified in your settings:       
    code/myshop/staticThis will overwrite existing files!Are you sure you want to do this?
```

输入*yes*然后回车。你会得到一条消息，告知那个静态文件已经复制到`STATIC_ROOT`目录中。

*collectstatic*命令复制所有静态文件从你的应用到定义在`STATIC_ROOT`设置的目录中。这允许每个应用去提供它自己的静态文件通过使用一个*static/*目录来包含它们。你还可以提供额外的静态文件来源在`STATICFILES_DIRS`设置。所有的目录被指定在`STATICFILED_DIRS`列中的都将会被复制到`STATIC_ROOT`目录中当*collectstatic*被执行的时候。

编辑*orders*应用目录下的*urls.py*文件并且添加如下URL模式：

```python
url(r'^admin/order/(?P<order_id>\d+)/pdf/$',    views.admin_order_pdf,    name='admin_order_pdf'),
```

现在，我们可以编辑管理列展示页面给*Order*模型（model）来添加一个链接给PDF文件给每一个结果。编辑*orders*应用的*admin.py*文件并且添加以下代码在`OrderAdmin`类上面：

```python
def order_pdf(obj):    return '<a href="{}">PDF</a>'.format(        reverse('orders:admin_order_pdf', args=[obj.id]))order_pdf.allow_tags = Trueorder_pdf.short_description = 'PDF bill'
```

添加`order_pdf`给`OrderAdmin`类的`list_display`属性：

```python
class OrderAdmin(admin.ModelAdmin):    list_display = ['id',                    # ... 
                    order_detail, 
                    order_pdf]
```

如果你指定一个`short_description`属性给你的调用，Django将会使用它给这个列命名。

打开 http://127.0.0.1:8000/admin/orders/order/ 在你的浏览器中。每一行现在都包含一个PDF链接，如下所示：

![django-8-13](http://ohqrvqrlb.bkt.clouddn.com/django-8-13.png)

点击某一个订单的**PDF**。你会看到一个生成的PDF文件，如下所示一个订单还没有支付完成：

![django-8-14](http://ohqrvqrlb.bkt.clouddn.com/django-8-14.png)

对于支付完成的订单，你会看到如下所示的PDF文件：

![django-8-15](http://ohqrvqrlb.bkt.clouddn.com/django-8-15.png)

##通过e-mail发送PDF文件

让我们发送一封e-mail给我们的顾客包含生成的PDF发表但一个支付被接收的时候。编辑*payment*应用下的*signals.py*文件并且添加如下导入：

```python
from django.template.loader import render_to_stringfrom django.core.mail import EmailMessagefrom django.conf import settingsimport weasyprintfrom io import BytesIO
```

之后添加如下代码在`order.save()`行之后，需要同样的缩进等级：

```python
# create invoice e-mailsubject = 'My Shop - Invoice no. {}'.format(order.id)message = 'Please, find attached the invoice for your recentpurchase.'email = EmailMessage(subject,                    message,                    'admin@myshop.com',                    [order.email])# generate PDFhtml = render_to_string('orders/order/pdf.html', {'order': order})out = BytesIO()
stylesheets=[weasyprint.CSS(settings.STATIC_ROOT + 'css/pdf.css')]weasyprint.HTML(string=html).write_pdf(out,                                        stylesheets=stylesheets)# attach PDF fileemail.attach('order_{}.pdf'.format(order.id),            out.getvalue(),            'application/pdf')# send e-mailemail.send()
```

在这个信号中，我们使用Django提供的`EmailMessage`类来创建一个e-mail对象。之后我们渲染这个模板（template）到`html`变量中。我们生成PDF文件从渲染的模板（template）中，并且我们输出它到一个`BytesIO`实例中，该实例是一个内容字节缓存。之后我们附加这个生成的PDF文件到`EmailMessage`对象通过使用它的`attach()`方法，包含这个`out`缓存的内容。

记住设置你的SMTP设置在项目的*settings.py*文件中来发送e-mail。你可以到**第二章 通过高级特性扩展你的blog**去看下一个SMTP配置的例子。

现在你可以打开Ngrok提供给你的应用的URL然后完成一个新的支付处理为了收到PDF发票到你的e-mail中。

##总结

在这一章中，你集成了一个支付网关到你的项目中。你定制了Django管理平台页面并且学习到了如何动态的生成CSV以及PDF文件。

在下一章中将会给你一个深刻理解关于国际化和本地化给Django项目。你还会学习到创建一个赠券系统已经构建一个产品推荐引擎。

##译者总结

不知不觉，第八章也翻译完成了，还是渣翻，精校之前大家先凑合着看吧，有问题我会及时更新。目前全书翻译已完成三分之二，离不开各位的支持，我们下章再见！对了，本章完成日是三八妇女（女神？）节，各位女看客们节日快乐！













