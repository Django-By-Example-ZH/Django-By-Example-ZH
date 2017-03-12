（译者注：祝大家新年快乐，这次带来《Django By Example》第四章的翻译，这章非常的实用，就这样）

#第四章
##创建一个社交网站
在上一章中，你学习了如何创建站点地图（sitemaps）和feeds，你还为你的blog应用创建了一个搜索引擎。在本章中，你将开发一个社交应用。你会为用户创建一些功能，例如：登录，登出，编辑，以及重置他们的密码。你会学习如何为你的用户创建一个定制的profile，你还会为你的站点添加社交认证。

本章将会覆盖一下几点：

* 使用认证（authentication）框架
* 创建用户注册视图（views）
* 通过一个定制的profile模型（model）扩展*User*模型（model）
* 使用*python-social-auth*添加社交认证

让我们开始创建我们的新项目吧。

###创建一个社交网站项目
我们要创建一个社交应用允许用户分享他们在网上找到的图片。我们需要为这个项目构建以下元素：

* 一个用来给用户注册，登录，编辑他们的profile，以及改变或重置密码的认证（authentication）系统
* 一个允许用户来关注其他人的关注系统（这里原文是follow，‘跟随’，感觉用‘关注’更加适合点）
* 为用户从其他任何网站分享过来的图片进行展示和打上书签
* 每个用户都有一个活动流允许他们看到所关注的人上传的内容

本章主要讲述第一点。

###开始你的社交网站项目
打开终端使用如下命令行为你的项目创建一个虚拟环境并且激活它：
​    
    mkdir evn
    virtualenv evn/bookmarks
    source env/bookmarks/bin/activate

shell提示将会展示你激活的虚拟环境，如下所示：

    (bookmarks)laptop:~ zenx$

通过以下命令在你的虚拟环境中安装Django：

    pip install Django==1.8.6

运行以下命令来创建一个新项目：

    django-admin startproject bookmarks

在创建好一个初始的项目结构以后，使用以下命令进入你的项目目录并且创建一个新的应用命名为*account*:

    cd bookmarks/
    django-admin startapp account

请记住在你的项目中激活一个新应用需要在*settings.py*文件中的*INSTALLED_APPS*设置中添加它。将新应用的名字添加在*INSTALLED_APPS*列中的所有已安装应用的最前面，如下所示：

    INSTALLED_APPS = (
        'account',
        # ...
    )

运行下一条命令为*INSTALLED_APPS*中默认包含的应用模型（models）同步到数据库中：

    python manage.py migrate

我们将要使用认证（authentication）框架来构建一个认证系统到我们的项目中。

###使用Django认证（authentication）框架
Django拥有一个内置的认证（authentication）框架用来操作用户认证（authentication），会话（sessions），权限（permissions）以及用户组。这个认证（authentication）系统包含了一些普通用户的操作视图（views），例如：登录，登出，修改密码以及重置密码。

这个认证（authentication）框架位于*django.contrib.auth*，被其他Django的*contrib*包调用。请记住你在*第一章 创建一个Blog应用*中使用过这个认证（authentication）框架并用来为你的blog应用创建了一个超级用户来使用管理站点。

当你使用*startproject*命令创建一个新的Django项目，认证（authentication）框架已经在你的项目设置中默认包含。它是由*django.contrib.auth*应用和你的项目设置中的*MIDDLEWARE_CLASSES*中的两个中间件类组成，如下：

* AuthenticationMiddleware:使用会话（sessions）将用户和请求（requests）进行关联
* SessionMiddleware:通过请求（requests）操作当前会话（sessions）

中间件就是一个在请求和响应阶段带有全局执行方法的类。你会在本书中的很多场景中使用到中间件。你将会在*第十三章 Going Live*中学习如何创建一个定制的中间件（译者注：啥时候能翻译到啊）。

这个认证（authentication）系统还包含了以下模型（models）：

* User：一个包含了基础字段的用户模型（model）；这个模型（model）的主要字段有：username， password, email, first_name, last_name, is_active。
* Group：一个组模型（model）用来分类用户
* Permission：执行特定操作的标识

这个框架还包含默认的认证（authentication）视图（views）和表单（forms），我们之后会用到。

###创建一个log-in视图（view）
我们将要开始使用Django认证（authentication）框架来允许用户登录我们的网站。我们的视图（view）需要执行以下操作来登录用户：

* 1、通过提交的表单（form）获取username和password
* 2、通过存储在数据库中的数据对用户进行认证
* 3、检查用户是否可用
* 4、登录用户到网站中并且开始一个认证（authentication）会话（session）

首先，我们要创建一个登录表单（form）。在你的account应用目录下创建一个新的*forms.py*文件，添加如下代码：

```python
from django import forms
	    
class LoginForm(forms.Form):        
	username = forms.CharField()        
	password = forms.CharField(widget=forms.PasswordInput)
```

这个表单（form）被用来通过数据库认证用户。请注意，我们使用*PsswordInput*控件来渲染HTML`input`元素，包含`type="password`属性。编辑你的*account*应用中的*views.py*文件，添加如下代码：

```python
from django.http import HttpResponse    
from django.shortcuts import render    
from django.contrib.auth import authenticate, login    
from .forms import LoginForm 

   
def user_login(request):        
    if request.method == 'POST':            
        form = LoginForm(request.POST)            
        if form.is_valid():                
            cd = form.cleaned_data                
            user = authenticate(username=cd['username'],
                                password=cd['password'])                
            if user is not None:                    
                if user.is_active:                        
                    login(request, user)                        
                    return HttpResponse('Authenticated successfully')
                else:                        
                    return HttpResponse('Disabled account')
            else:            
                return HttpResponse('Invalid login')        
    else:            
        form = LoginForm()        
    return render(request, 'account/login.html', {'form': form})
```

以上就是我们在视图（view）中所作的基本登录操作：当*user_login*被一个GET请求（request）调用，我们实例化一个新的登录表单（form）并通过`form = LoginForm()`在模板（template）中展示它。当用户通过POST方法提交表单（form）时，我们执行以下操作：

* 1、通过使用`form = LoginForm(request.POST)`使用提交的数据实例化表单（form）
* 2、检查这个表单是否有效。如果无效，我们在模板（template）中展示表单错误信息（举个例子，比如用户没有填写其中一个字段就进行提交）
* 3、如果提交的数据是有效的，我们使用`authenticate()`方法通过数据库对这个用户进行认证（authentication）。这个方法带入一个*username*和一个*password*，如果这个用户成功的进行了认证则返回一个用户对象，否则是*None*。如果用户没有被认证通过，我们返回一个*HttpResponse*展示一条消息。
* 4、如果这个用户认证（authentication）成功，我们使用`is_active`属性来检查用户是否可用。这是一个Django的*User*模型（model)属性。如果这个用户不可用，我们返回一个*HttpResponse*展示信息。
* 5、如果用户可用，我们登录这个用户到网站中。我们通过调用`login()`方法集合用户到会话（session）中然后返回一条成功消息。

> 请注意*authentication*和*login*中的不同点：`authenticate()`检查用户认证信息，如果用户是正确的则返回一个用户对象；`login()`集合用户到当前的会话（session）中。

现在，你需要为这个视图（view）创建一个URL模式。在你的*account*应用目录下创建一个新的*urls.py*文件，添加如下代码：

    from django.conf.urls import url    
    from . import views    
	
    urlpatterns = [        
        # post views        
        url(r'^login/$', views.user_login, name='login'),    
    ]

编辑位于你的*bookmarks*项目目录下的*urls.py*文件，将*account*应用下的URL模式包含进去：

    from django.conf.urls import include, url    
    from django.contrib import admin
	    
    urlpatterns = [        
        url(r'^admin/', include(admin.site.urls)),
        url(r'^account/',include('account.urls')),    
    ]

这个登录视图（view）现在已经可以通过URL进行访问。现在是时候为这个视图（view）创建一个模板。因为之前你没有这个项目的任何模板，你可以开始创建一个主模板（template）能够被登录模板（template）继承使用。在*account*应用目录中创建以下文件和结构：

```shell
    templates/
        account/
            login.html
        base.html
```

编辑*base.html*文件添加如下代码：

    {% load staticfiles %}    
    <!DOCTYPE html>    
    <html>    
        <head>      
            <title>{% block title %}{% endblock %}</title>      
            <link href="{% static "css/base.css" %}" rel="stylesheet">    
        </head>    
        <body>      
            <div id="header">        
                <span class="logo">Bookmarks</span>      
            </div>      
            <div id="content">        
                {% block content %}        
                {% endblock %}      
            </div>    
        </body>    
    </html>

以上将是这个网站的基础模板（template）。就像我们在上一章项目中做过的一样，我们在这个主模板（template）中包含了CSS样式。你可以在本章的示例代码中找到这些静态文件。复制示例代码中的*account*应用下的*static/*目录到你的项目中的相同位置，这样你就可以使用这些静态文件了。

基础模板（template）定义了一个*title*和一个*content*区块可以让被继承的子模板（template）填充内容。

让我们为我们的登录表单（form）创建模板（template）。打开*account/login.html*模板（template）添加如下代码：

    {% extends "base.html" %}
    {% block title %}Log-in{% endblock %}
    {% block content %}
        <h1>Log-in</h1>
        <p>Please, use the following form to log-in:</p>
        <form action="." method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Log-in"></p>
        </form>
    {% endblock %}
    
这个模板（template）包含了视图（view）中实例化的表单（form）。因为我们的表单（form）将会通过POST方式进行提交，所以我们包含了`{% csrf_token %}`模板（template）标签(tag)用来通过CSRF保护。你已经在*第二章 使用高级特性扩展你的博客应用*中学习过CSRF保护。

目前还没有用户在你的数据库中。你首先需要创建一个超级用户用来登录管理站点来管理其他的用户。打开命令行执行`python manage.py createsuperuser`。填写username, e-mail以及password。之后通过命令`python manage.py runserver`运行开发环境，然后在你的浏览器中打开 http://127.0.0.1:8000/admin/ 。使用你刚才创建的username和passowrd来进入管理站点。你会看到Django管理站点包含Django认证（authentication）框架中的*User*和*Group*模型（models）。如下所示：

![django-4-1](http://ohqrvqrlb.bkt.clouddn.com/django-4-1.png)

使用管理站点创建一个新的用户然后打开 http://127.0.0.1:8000/account/login/ 。你会看到被渲染过的模板（template），包含一个登录表单（form）：

![django-4-2](http://ohqrvqrlb.bkt.clouddn.com/django-4-2.png)

现在，只填写一个字段保持另外一个字段为空进行表单（from）提交。在这例子中，你会看到这个表单（form）是无效的并且显示了一个错误信息：

![django-4-3](http://ohqrvqrlb.bkt.clouddn.com/django-4-3.png)

如果你输入了一个不存在的用户或者一个错误的密码，你会得到一个**Invalid login**信息。

如果你输入了有效的认证信息，你会得到一个**Authenticated successfully**消息：

![django-4-4](http://ohqrvqrlb.bkt.clouddn.com/django-4-4.png)

###使用Django认证（authentication）视图（views）
Django包含了一些表单（forms）和视图（views）在认证（authentication）框架中让你可以立刻(straightaway)使用。你之前创建的登录视图（view）是一个非常的练习用来理解Django中的用户认证（authentication）过程。无论如何，你可以使用默认的Django认证（authentication）视图（views）在大部分的例子中。

Django提供以下视图（views）来处理认证（authentication）：

* login：操作表单（form）中的登录然后登录一个用户
* logout：登出一个用户
* logout_then_login：登出一个用户然后重定向这个用户到登录页面

Django提供以下视图（views）来操作密码修改：

* password_change:操作一个表单（form）来修改用户密码
* password_change_done:当用户成功修改他的密码后提供一个成功提示页面

Django还包含了以下视图（views）允许用户重置他们的密码：

* password_reset:允许用户重置他的密码。它会生成一条带有一个token的一次性使用链接然后发送到用户的邮箱中。
* password_reset_done:告知用户已经发送了一封可以用来重置密码的邮件到他的邮箱中。
* password_reset_complete:当用户重置完成他的密码后提供一个成功提示页面。

以上的视图（views）可以帮你节省很多时间当你创建一个带有用户账号的网站。这些视图（views）使用的默认值你可以进行覆盖，例如需要渲染的模板位置或者视图（view）需要使用到的表单（form）。

你可以通过访问 https://docs.
djangoproject.com/en/1.8/topics/auth/default/#module-django.contrib.auth.views 获取更多内置的认证（authentication）视图（views）信息。

###登录和登出视图（views）

编辑你的*account*应用下的*urls.py*文件，如下所示：

    from django.conf.urls import url
    from . import views
    urlpatterns = [
        # previous login view
        # url(r'^login/$', views.user_login, name='login'),
        # login / logout urls
        url(r'^login/$',
            'django.contrib.auth.views.login',
            name='login'),
        url(r'^logout/$',
            'django.contrib.auth.views.logout',
            name='logout'),
        url(r'^logout-then-login/$',
            'django.contrib.auth.views.logout_then_login',
            name='logout_then_login'),
    ]
    
我们将之前创建的*user_login*视图（view）URL模式进行注释，然后使用Django认证（authentication）框架提供的*login*视图（view）。

在你的account应用中的template目录下创建一个新的目录命名为*registration*。这个路径是Django认证（authentication）视图（view）期望你的认证（authentication）模块（template）默认的存放路径。在这个新目录中创建一个新的文件，命名为*login.html*，然后添加如下代码：

    {% extends "base.html" %}
    {% block title %}Log-in{% endblock %}
    {% block content %}
      <h1>Log-in</h1>
      {% if form.errors %}
        <p>
          Your username and password didn't match.
          Please try again.
        </p>
      {% else %}
        <p>Please, use the following form to log-in:</p>
      {% endif %}
      <div class="login-form">
        <form action="{% url 'login' %}" method="post">
          {{ form.as_p }}
          {% csrf_token %}
          <input type="hidden" name="next" value="{{ next }}" />
          <p><input type="submit" value="Log-in"></p>
        </form>
      </div>
     {% endblock %}

这个登录模板（template）和我们之前创建的基本类似。Django默认使用位于*django.contrib.auth.forms*中的*AuthenticationForm*。这个表单（form）会尝试对用户进行认证如果登录不成功就会抛出一个验证错误。在这个例子中，我们可以在模板（template）中使用`{% if form.errors %}`来找到错误如果用户提供了错误的认证信息。请注意，我们添加了一个隐藏的HTML`<input>`元素来提交叫做*next*的变量值。这个变量是登录视图（view）的首个设置当你在请求（request）中传递一个*next*参数（举个例子：http://127.0.0.1:8000/account/login/?next=/account/）。

*next*参数必须是一个URL。当这个参数被给予的时候，Django登录视图（view）将会在用户登录完成后重定向到给予的URL。

现在，在*registrtion*模板（template）目录下创建一个*logged_out.html*模板（template）添加如下代码：

    {% extends "base.html" %}
    {% block title %}Logged out{% endblock %}
    {% block content %}
      <h1>Logged out</h1>
      <p>You have been successfully logged out. You can <a href="{% url "login" %}">log-in again</a>.</p>
    {% endblock %}
    
这个模板（template）Django会在用户登出的时候进行展示。

在添加了URL模式以及登录和登出视图（view）的模板之后，你的网站已经准备好让用户使用Django认证（authentication）视图进行登录。

> 请注意，在我们的地址配置中包含的*logtou_then_login*视图（view）不需要任何模板（template），因为它执行了一个重定向到登录视图（view）。

现在，我们要创建一个新的视图（view）来显示一个dashboard给用户当他或她登录他们的账号。打开你的*account*应用中的*views.py*文件，添加以下代码：

    from django.contrib.auth.decorators import login_required
    @login_required
    def dashboard(request):
        return render(request,
                     'account/dashboard.html',
                     {'section': 'dashboard'})

我们使用认证（authentication）框架的*login_required*装饰器（decorator）装饰我们的视图（view）。*login_required*装饰器（decorator）会检查当前用户是否通过认证,如果用户通过认证，它会执行装饰的视图（view），如果用户没有通过认证，它会把用户重定向到带有一个名为*next*的GET参数的登录URL，该GET参数保存的变量为用户当前尝试访问的页面URL。通过这些动作，登录视图（view）会将登录成功的用户重定向到用户登录之前尝试访问过的URL。请记住我们在登录模板（template）中的登录表单（form）中添加的隐藏`<input>`就是为了这个目的。

我们还定义了一个*section*变量。我们会使用该变量来跟踪用户在站点中正在查看的页面。多个视图（views）可能会对应相同的section。这是一个简单的方法用来定义每个视图（view）对应的section。

现在，你需要创建一个给dashboard视图（view）使用的模板（template）。在*templates/account/*目录下创建一个新文件命名为*dashboard.html*。添加如下代码：

    {% extends "base.html" %}
    {% block title %}Dashboard{% endblock %}
    {% block content %}
      <h1>Dashboard</h1>
      <p>Welcome to your dashboard.</p>
    {% endblock %}
    
之后，为这个视图（view）在*account*应用中的*urls.py*文件中添加如下URL模式：

    urlpatterns = [
        # ...
        url(r'^$', views.dashboard, name='dashboard'),
    ]
    
编辑你的项目的*settings.py*文件，添加如下代码：

    from django.core.urlresolvers import reverse_lazy
    LOGIN_REDIRECT_URL = reverse_lazy('dashboard')
    LOGIN_URL = reverse_lazy('login')
    LOGOUT_URL = reverse_lazy('logout')
    
这些设置的意义：

* LOGIN_REDIRECT_URL：告诉Django用户登录成功后如果*contrib.auth.views.login*视图（view）没有获取到*next*参数将会默认重定向到哪个URL。
* LOGIN_URL：重定向用户登录的URL（例如：使用*login_required*装饰器（decorator））。
* LOGOUT_URL：重定向用户登出的URL。

我们使用*reverse_laze()*来动态构建URL通过它们的名字。*reverse_laze()*方法reverses URLs就像*reverse()*所做的一样，但是你可以使用它当你需要reverse URLs在你项目的URL配置读取之前。

让我们总结下目前为止我们都做了哪些工作：

* 你在项目中添加了Django内置的认证（authentication）登录和登出视图（views）。
* 你为每一个视图（view)创建了定制的模板（templates），并且定义了一个简单的视图（view）让用户登录后进行重定向。
* 最后，你配置了你的Django设置使用默认的URLs。

现在，我们要给我们的主模板（template）添加登录和登出链接将所有的一切都连接起来。

为了做到这点，我们必须确定无论用户是已登录状态还是没有登录的时候，都会显示适当的链接。通过认证（authentication）中间件当前的用户被集合在HTTP请求（request)对象中。你可以访问用户信息通过使用`request.user`。你会防线一个用户对象在请求（request)中，即使这个用户并没有认证通过。一个非认证的用户在请求（request）被设置成一个*AnonymousUser*的实例。一个最好的方法来检查当前的用户是否通过认证是通过调用`request.user.is_authenticated()`。

编辑你的*base.html*文件修改ID为*header*的`<div>`元素，如下所示：

    <div id="header">
    <span class="logo">Bookmarks</span>
    {% if request.user.is_authenticated %}
         <ul class="menu">
          <li {% if section == "dashboard" %}class="selected"{% endif %}>
            <a href="{% url "dashboard" %}">My dashboard</a>
          </li>
          <li {% if section == "images" %}class="selected"{% endif %}>
            <a href="#">Images</a>
          </li>
          <li {% if section == "people" %}class="selected"{% endif %}>
            <a href="#">People</a>
           </li>
          </ul>
         {% endif %}
         <span class="user">
           {% if request.user.is_authenticated %}
             Hello {{ request.user.first_name }},
             <a href="{% url "logout" %}">Logout</a>
           {% else %}
             <a href="{% url "login" %}">Log-in</a>
           {% endif %}
        </span>
    </div>
    
就像你所看到的，我们只给通过认证（authentication）的用户显示站点菜单。我们还检查当前的section来给对应的`<li>`组件添加一个*selected*的class属性来使当前section在菜单中进行高亮显示通过使用CSS。我们还显示用户的第一个名字和一个登出的链接如果用户已经通过认证（authentication），否则，就是一个登出链接。

现在，在浏览器中打开 http://127.0.0.1:8000/account/login/ 。你会看到登录页面。输入可用的用户名和密码然后点击**Log-in**按钮。你会看到如下图所示：

![django-4-5](http://ohqrvqrlb.bkt.clouddn.com/django-4-5.png)
    
你能看到*My Dashboard* section 通过CSS的作用高亮显示因为它拥有一个*selected* class。因为当前用户已经通过了认证（authentication）所有用户的第一个名字在右上角进行了显示。点击**Logout**链接。你会看到如下图所示：

![django-4-6](http://ohqrvqrlb.bkt.clouddn.com/django-4-6.png)

在这个页面中，你能看到用户已经登出，然后，你无法看到当前网站的任何菜单。在右上角现在显示的是**Log-in**链接。

如果你在你的登出页面中看到了Django管理站点的登出页面，检查项目*settings.py*中的*INSTALLED_APPS*,确保*django.contrib.admin*在*account*应用的后面。每个模板（template）被定位在同样的相对路径时，Django模板（template）读取器将会使用它找到的第一个应用中的模板（templates）。

###修改密码视图（views）
我们还需要我们的用户在登录成功后可以进行修改密码。我们要集成Django认证（authentication）视图（views）来修改密码。打开*account*应用中的*urls.py*文件，添加如下URL模式：

    # change password urls
    url(r'^password-change/$',
       'django.contrib.auth.views.password_change',
       name='password_change'),
    url(r'^password-change/done/$',
       'django.contrib.auth.views.password_change_done',
       name='password_change_done'),

*password_change*视图（view）将会操作表单（form）进行修改密码，*password_change_done*将会显示一条成功信息当用户成功的修改他的密码。让我们为每个视图（view）创建一个模板（template）。

在你的*account*应用*templates/registration/*目录下添加一个新的文件命名为*password_form.html*，在文件中添加如下代码：

    {% extends "base.html" %}
    {% block title %}Change you password{% endblock %}
    {% block content %}
      <h1>Change you password</h1>
      <p>Use the form below to change your password.</p>
      <form action="." method="post">
        {{ form.as_p }}
        <p><input type="submit" value="Change"></p>
        {% csrf_token %}
      </form>
    {% endblock %}

这个模板（template）包含了修改密码的表单（form）。现在，在相同的目录下创建另一个文件，命名为*password_change_done.html*，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Password changed{% endblock %}
    {% block content %}
      <h1>Password changed</h1>
      <p>Your password has been successfully changed.</p>
     {% endblock %}
     
这个模板（template）只包含显示一条成功的信息 当用户成功的修改他们的密码。

在浏览器中打开 http://127.0.0.1:8000/account/password-change/ 。如果你的用户没有登录，浏览器会重定向你到登录页面。在你成功认证（authentication）登录后，你会看到如下图所示的修改密码页面：

![django-4-7](http://ohqrvqrlb.bkt.clouddn.com/django-4-7.png)

在表单（form）中填写你的旧密码和新密码，然后点击**Change**按钮。你会看到如下所示的成功页面：

![django-4-8](http://ohqrvqrlb.bkt.clouddn.com/django-4-8.png)

登出再使用新的密码进行登录来验证每件事是否如预期一样工作。

###重置密码视图（views）
在*account*应用*urls.py*文件中为密码重置添加如下URL模式：

    # restore password urls
     url(r'^password-reset/$',
        'django.contrib.auth.views.password_reset',
        name='password_reset'),
     url(r'^password-reset/done/$',
         'django.contrib.auth.views.password_reset_done',
         name='password_reset_done'),
      url(r'^password-reset/confirm/(?P<uidb64>[-\w]+)/(?P<token>[-\w]+)/$',
           'django.contrib.auth.views.password_reset_confirm',
         name='password_reset_confirm'),
     url(r'^password-reset/complete/$',
         'django.contrib.auth.views.password_reset_complete',
          name='password_reset_complete'),

在你的*account*应用*templates/registration/*目录下添加一个新的文件命名为*password_reset_form.html*，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Reset your password{% endblock %}
    {% block content %}
      <h1>Forgotten your password?</h1>
      <p>Enter your e-mail address to obtain a new password.</p>
      <form action="." method="post">
        {{ form.as_p }}
        <p><input type="submit" value="Send e-mail"></p>
        {% csrf_token %}
      </form>
    {% endblock %}
    
现在，在相同的目录下添加另一个文件命名为*password_reset_email.html*，为它添加如下代码：

    Someone asked for password reset for email {{ email }}. Follow the link below:
    {{ protocol }}://{{ domain }}{% url "password_reset_confirm" uidb64=uid token=token %}
    Your username, in case you've forgotten: {{ user.get_username }}
    
这个模板（template）会被用来渲染发送给用户的重置密码邮件。

在相同目录下添加另一个文件命名为*password_reset_done.html，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Reset your password{% endblock %}
    {% block content %}
      <h1>Reset your password</h1>
      <p>We've emailed you instructions for setting your password.</p>
      <p>If you don't receive an email, please make sure you've entered the address you registered with.</p>
    {% endblock %}
    
再创建另一个模板（template）命名为*passowrd_reset_confirm.html*，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Reset your password{% endblock %}
    {% block content %}
      <h1>Reset your password</h1>
      {% if validlink %}
        <p>Please enter your new password twice:</p>
        <form action="." method="post">
          {{ form.as_p }}
          {% csrf_token %}
          <p><input type="submit" value="Change my password" /></p>
        </form>
      {% else %}
        <p>The password reset link was invalid, possibly because it has already been used. Please request a new password reset.</p>
      {% endif %}
    {% endblock %}
    
在以上模板中，如果重置链接是有效的我们将会进行检查。Django重置密码视图（view）会设置这个变量然后将它带入这个模板（template）的上下文环境中。如果重置链接有效，我们展示用户密码重置表单（form）。

创建另一个模板（template）命名为*password_reset_complete.html*，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Password reset{% endblock %}
    {% block content %}
      <h1>Password set</h1>
      <p>Your password has been set. You can <a href="{% url "login" %}">log in now</a></p>
    {% endblock %}
    
最后，编辑*account*应用中的*/registration/login.html*模板（template），添加如下代码在`<form>`元素之后：

    <p><a href="{% url "password_reset" %}">Forgotten your password?</a></p>
    
现在，在浏览器中打开 http://127.0.0.1:8000/account/login/ 然后点击**Forgotten your password?**链接。你会看到如下图所示页面：

![django-4-9](http://ohqrvqrlb.bkt.clouddn.com/django-4-9.png)

在这部分，你需要在你项目中的*settings.py*中添加一个SMTP配置，这样Django才能发送e-mails。我们已经学习过如何添加e-mail配置在*第二章 使用高级特性来优化你的blog*。当然，在开发期间，我们可以配置Django在标准输出中输出e-mail内容来代替通过SMTP服务发送邮件。Django提供一个e-mail后端来输出e-mail内容到控制器中。编辑项目中*settings.py*文件，添加如下代码：

    EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
    
*EMAIL_BACKEND*设置这个类用来发送e-mails。

回到你的浏览器中，填写一个存在用户的e-mail地址，然后点击**Send e-mail**按钮。你会看到如下图所示页面：

![django-4-10](http://ohqrvqrlb.bkt.clouddn.com/django-4-10.png)

当你运行开发服务的时候看眼控制台输出。你会看到如下所示生成的e-mail：

    IME-Version: 1.0
     Content-Type: text/plain; charset="utf-8"
    Content-Transfer-Encoding: 7bit
    Subject: Password reset on 127.0.0.1:8000
    From: webmaster@localhost
    To: user@domain.com
    Date: Thu, 24 Sep 2015 14:35:08 -0000
    Message-ID: <20150924143508.62996.55653@zenx.local>
    Someone asked for password reset for email user@domain.com. Follow the link below:
    http://127.0.0.1:8000/account/password-reset/confirm/MQ/45f-9c3f30caafd523055fcc/
    Your username, in case you've forgotten: zenx

这封e-mail被我们之前创建的*password_reset_email.html*模板给渲染。这个给你重置密码的URL带有一个Django动态生成的token。在浏览器中打开这个链接，你会看到如下所示页面：

![django-4-11](http://ohqrvqrlb.bkt.clouddn.com/django-4-11.png)

这个页面用来设置新密码对应*password_reset_confirm.html*模板（template）。填写新的密码然后点击**Change my password button**。Django会创建一个新的加密后密码保存进数据库，你会看到如下图所示页面：

![django-4-12](http://ohqrvqrlb.bkt.clouddn.com/django-4-12.png)

现在你可以使用你的新密码登录你的账号。每个用来设置新密码的token只能使用一次。如果你再次打开你之前获取的链接，你会得到一条信息告诉你这个token已经无效了。

你在项目中集成了Django认证（authentication）框架的视图（views）。这些视图（views）对大部分的例子都适合。当然，你能创建你的自己视图如果你需要一种不同的行为。

##用户注册和用户profiles
现有的用户已经可以登录，登出，修改他们的密码，以及当他们忘记密码的时候重置他们的密码。现在，我们需要构建一个视图（view）允许访问者创建他们的账号。

###用户注册
让我们创建一个简单的视图（view）允许用户在我们的网站中进行注册。首先，我们需要创建一个表单（form）让用户填写用户名，他们的真实姓名以及密码。编辑*account*应用新目录下的*forms.py*文件添加如下代码：

    from django.contrib.auth.models import User
    class UserRegistrationForm(forms.ModelForm):
        password = forms.CharField(label='Password',
                                   widget=forms.PasswordInput)
        password2 = forms.CharField(label='Repeat password',
                                    widget=forms.PasswordInput)
        class Meta:
            model = User
            fields = ('username', 'first_name', 'email')
        def clean_password2(self):
            cd = self.cleaned_data
            if cd['password'] != cd['password2']:
                raise forms.ValidationError('Passwords don\'t match.')
            return cd['password2']
            
我们为*User*模型（model）创建了一个model表单（form）。在我们的表单（form）中我们只包含了模型（model）中的*username,first_name,email*字段。这些字段会在它们对应的模型（model）字段上进行验证。例如：如果用户选择了一个已经存在的用户名，将会得到一个验证错误。我们还添加了两个额外的字段*password*和*password2*给用户用来填写他们的新密码和确定密码。我们定义了一个`clean_password2()`方法去检查第二次输入的密码是否和第一次输入的保持一致，如果不一致这个表单将会是无效的。这个检查会被执行当我们验证这个表单（form）通过调用它的`is_valid()`方法。你可以提供一个`clean_<fieldname>()`方法给任何一个你的表单（form）字段用来清理值或者抛出表单（from）指定的字段的验证错误。表单（forms）还包含了一个`clean()`方法用来验证表单（form）的所有内容，这是非常有用的用来验证需要依赖其他字段的字段。

Django还提供一个*UserCreationForm*表单（form）给你使用，它位于*django.contrib.auth.forms*非常类似与我们刚才创建的表单（form）。

编辑*account*应用中的*views.py*文件，添加如下代码：

    from .forms import LoginForm, UserRegistrationForm
    def register(request):
        if request.method == 'POST':
            user_form = UserRegistrationForm(request.POST)
            if user_form.is_valid():
                # Create a new user object but avoid saving it yet
                new_user = user_form.save(commit=False)
                # Set the chosen password
                new_user.set_password(
                    user_form.cleaned_data['password'])
                # Save the User object
                new_user.save()
                return render(request,
                             'account/register_done.html',
                             {'new_user': new_user})
        else:
            user_form = UserRegistrationForm()
        return render(request,
                     'account/register.html',
                     {'user_form': user_form})

这个创建用户账号的视图（view）非常简单。为了保护用户的隐私，我们使用*User*模型（model）的`set_password()`方法将用户的原密码进行加密后再进行保存操作。

现在，编辑*account*应用中的*urls.py*文件，添加如下URL模式：

    url(r'^register/$', views.register, name='register'),
    
最后，创建一个新的模板（template）在*account/*模板（template）目录下，命名为*register.html*,为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Create an account{% endblock %}
    {% block content %}
      <h1>Create an account</h1>
      <p>Please, sign up using the following form:</p>
      <form action="." method="post">
        {{ user_form.as_p }}
        {% csrf_token %}
        <p><input type="submit" value="Create my account"></p>
      </form>
    {% endblock %}
    
在相同的目录中添加一个模板（template）文件命名为*register_done.html*，为它添加如下代码：

    {% extends "base.html" %}
    {% block title %}Welcome{% endblock %}
    {% block content %}
      <h1>Welcome {{ new_user.first_name }}!</h1>
      <p>Your account has been successfully created. Now you can <a href="{% url "login" %}">log in</a>.</p>
    {% endblock %}
    
现在，在浏览器中打开 http://127.0.0.1:8000/account/register/ 。你会看到你创建的注册页面：

![django-4-13](http://ohqrvqrlb.bkt.clouddn.com/django-4-13.png)

填写用户信息然后点击**Create my account**按钮。如果所有的字段都验证都过，这个用户将会被创建然后会得到一条成功信息，如下所示：

![django-4-14](http://ohqrvqrlb.bkt.clouddn.com/django-4-14.png)

点击**log-in**链接输入你的用户名和密码来验证账号是否成功创建。

现在，你还可以添加一个注册链接在你的登录模板（template）中。编辑*registration/login.html*模板（template）然后替换以下内容：

    <p>Please, use the following form to log-in:</p>
    
为：

    <p>Please, use the following form to log-in. If you don't have an account <a href="{% url "register" %}">register here</a></p>
    
如此，我们就可以从登录页面进入注册页面。

###扩展User模型（model）
当你需要处理用户账号，你会发现Django认证（authentication）框架的*User*模型（model）只适应一般的案例。无论如何，*User*模型（model）只有一些最基本的字段。你可能希望扩展*User*模型包含额外的数据。最好的办法就是创建一个*profile*模型（model）包含所有额外的字段并且和Django的*User*模型（model）做一对一的关联。

编辑*account*应用中的*model.py*文件，添加如下代码：

    from django.db import models
    from django.conf import settings
    class Profile(models.Model):
        user = models.OneToOneField(settings.AUTH_USER_MODEL)
        date_of_birth = models.DateField(blank=True, null=True)
        photo = models.ImageField(upload_to='users/%Y/%m/%d', blank=True)
    def __str__(self):
        return 'Profile for user {}'.format(self.user.username)

> 为了保持你的代码通用化，使用`get_user_model()`方法来取回用户模型（model），当定义了模型（model）和用户模型的关系使用*AUTH_USER_MODEL*设置来引用它，替代直接引用auth的*User*模型（model)。

*user*一对一字段允许我们关联用户和profiles。*photo*字段是一个*ImageField*字段。你需要安装一个Python包来管理图片，使用**PIL(Python Imaging Library)**或者**Pillow**（PIL的分叉）,在shell中运行一下命令来安装*Pillow*：

    pip install Pillow==2.9.0
    
为了Django能在开发服务中管理用户上传的多媒体文件，在项目*setting.py*文件中添加如下设置：

    MEDIA_URL = '/media/'
    MEDIA_ROOT = os.path.join(BASE_DIR, 'media/')

MEDIA_URL 是管理用户上传的多媒体文件的主URL，MEDIA_ROOT是这些文件在本地保存的路径。我们动态的构建这些路径相对我们的项目路径来确保我们的代码更通用化。

现在，编辑*bookmarks*项目中的主*urls.py*文件，修改代码如下所示：

    from django.conf.urls import include, url
    from django.contrib import admin
    from django.conf import settings
    from django.conf.urls.static import static
    urlpatterns = [
        url(r'^admin/', include(admin.site.urls)),
        url(r'^account/', include('account.urls')),
    ]
    if settings.DEBUG:
        urlpatterns += static(settings.MEDIA_URL,
                            document_root=settings.MEDIA_ROOT)
    
在这种方法中，Django开发服务器将会在开发时改变对多媒体文件的服务。
    
*static()*帮助函数最适合在开发环境中使用而不是在生产环境使用。绝对不要在生产环境中使用Django来服务你的静态文件。
    
打开终端运行以下命令来为新的模型（model）创建数据库迁移：

    python manage.py makemigrations
    
你会获得以下输出：

    Migrations for 'account':
        0001_initial.py:
            - Create model Profile
            
接着，同步数据库通过以下命令：

    python manage.py migrate
    
你会看到包含以下内容的输出：

    Applying account.0001_initial... OK
    
编辑*account*应用中的*admin.py*文件，在管理站点注册*Profiel*模型（model），如下所示：

    from django.contrib import admin
    from .models import Profile
    class ProfileAdmin(admin.ModelAdmin):
        list_display = ['user', 'date_of_birth', 'photo']
    admin.site.register(Profile, ProfileAdmin)
    
使用`python manage.py runnserver*命令重新运行开发服务。现在，你可以看到*Profile*模型已经存在你项目中的管理站点中，如下所示：

![django-4-15](http://ohqrvqrlb.bkt.clouddn.com/django-4-15.png)

现在，我们要让用户可以在网站编辑它们的*profile*。添加如下的模型（model）表单（forms）到*account*应用中的*forms.py*文件：

    from .models import Profile
    class UserEditForm(forms.ModelForm):
        class Meta:
            model = User
            fields = ('first_name', 'last_name', 'email')
    class ProfileEditForm(forms.ModelForm):
        class Meta:
            model = Profile
            fields = ('date_of_birth', 'photo')

这两个表单（forms）的功能：

* UserEditForm：允许用户编辑它们的*first name,last name, e-mail*,这些储存在*User*模型（model）中的内置字段。
* ProfileEditForm：允许用户编辑我们存储在定制的*Profile*模型（model）中的额外数据。用户可以编辑它们的生日数据以及为他们的*profile*上传一张图片。

编辑*account*应用中的*view.py*文件，导入*Profile*模型（model），如下所示：

    from .models import Profile
   
然后添加如下内容到*register*视图（view）中的`new_user.save()`下方：

    # Create the user profile
    profile = Profile.objects.create(user=new_user)
    
当用户在我们的站点中注册，我们会创建一个对应的空的*profile*给他们。你需要为之前创建的用户们一个个手动创建一个*Profile*对象在管理站点中。

现在，我们要让用于能够编辑他们的*pfofile*。添加如下代码到相同文件中：

    from .forms import LoginForm, UserRegistrationForm, \
    UserEditForm, ProfileEditForm
    @login_required
    def edit(request):
        if request.method == 'POST':
            user_form = UserEditForm(instance=request.user,
                                    data=request.POST)
            profile_form = ProfileEditForm(instance=request.user.profile,
                                            data=request.POST,
                                            files=request.FILES)
            if user_form.is_valid() and profile_form.is_valid():
                user_form.save()
                profile_form.save()
        else:
            user_form = UserEditForm(instance=request.user)
            profile_form = ProfileEditForm(instance=request.user.profile)
        return render(request,
                     'account/edit.html',
                     {'user_form': user_form,
                     'profile_form': profile_form})

我们使用*login_required*装饰器*decorator*是因为用户编辑他们的*profile*必须是认证通过的状态。在这个例子中，我们使用两个模型（model）表单（forms）：*UserEditForm*用来存储数据到内置的*User*模型（model）中，*ProfileEditForm*用来存储额外的*profile*数据。为了验证提交的数据，我们检查每个表单（forms）通过`is_valid()`方法是否都返回*True*。在这个例子中，我们保存两个表单（form）来更新数据库中对应的对象。

在*account*应用中的*urls.py*文件中添加如下URL模式：

    url(r'^edit/$', views.edit, name='edit'),

最后，在*templates/account/*中创建一个新的模板（template）命名为*edit.html*，为它添加如下内容：

    {% extends "base.html" %}
    {% block title %}Edit your account{% endblock %}
    {% block content %}
        <h1>Edit your account</h1>
        <p>You can edit your account using the following form:</p>
        <form action="." method="post" enctype="multipart/form-data">
            {{ user_form.as_p }}
            {{ profile_form.as_p }}
            {% csrf_token %}
        <p><input type="submit" value="Save changes"></p>
        </form>
    {% endblock %}

> 我们在表单（form）中包含`enctype="multipart/form-data"`用来支持文件上传。我们使用一个HTML表单来提交两个*user_form*和*profile_form*表单（forms）。

注册一个新用户然后打开 http://127.0.0.1:8000/account/edit/。你会看到如下所示页面：

![django-4-16](http://ohqrvqrlb.bkt.clouddn.com/django-4-16.png)

现在，你可以编辑dashboard页面包含编辑*profile*的页面链接和修改密码的页面链接。打开*account/dashboard.html*模板（model）替换如下代码：

    <p>Welcome to your dashboard.</p>

为：

    <p>Welcome to your dashboard. You can <a href="{% url "edit" %}">edit your profile</a> or <a href="{% url "password_change" %}">change your password</a>.</p>
    
用户现在可以从他们的dashboard访问编辑他们的*profile*的表单。

###使用一个定制*User*模型（model）
Django还提供一个方法可以使用你自己定制的模型（model）来替代整个*User*模型（model）。你自己的用户类需要继承Django的*AbstractUser*类，这个类提供了一个抽象的模型（model）用来完整执行默认用户。你可访问  https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#substituting-a-custom-user-model来获得这个方法的更多信息。

使用一个定制的用户模型（model）将会带给你很多的灵活性，但是它也可能给一些需要与*User*模型（model）交互的即插即用的应用集成带来一定的困难。

##使用messages框架
当处理用户的操作时，你可能想要通知你的用户关于他们操作的结果。Django有一个内置的messages框架允许你给你的用户显示一次性的提示。messages框架位于*django.contrib.messages*，当你使用`python manage.py startproject`命令创建一个新项目的时候，messages框架就被默认包含在*settings.py*文件中的*INSTALLED_APPS*中。你会注意到你的设置文件包含了一个名为*django.contrib.messages.middleware.MessageMiddleware*的中间件在*MIDDLEWARE_CLASSES*设置中。messages框架提供了一个简单的方法添加消息给用户。消息被存储在数据库中并且会在用户的下一次请求中展示。你可以在你的视图（views）中导入*messages*模块使用消息messages框架，用简单的快捷方式添加新的messages，如下所示：

    from django.contrib import messages    messages.error(request, 'Something went wrong')

你可以使用`add_message()`方法创建新的messages或用下方任意一个快捷方法：

* success()：当操作成功后显示成功的messages
* info()：展示messages
* warning()：某些还没有达到失败的程度但已经包含有失败的风险，警报用
* error()：操作没有成功或者某些事情失败
* debug()：在生产环境中这种messages会移除或者忽略

让我们显示messages给用户。因为messages框架是被项目全局应用，我们可以在主模板（template）诶用户展示messages。打开*base.html*模板（template）在id为*header*的`<div>`和id为*content*的`<div>`之间添加如下内容：

    {% if messages %}     <ul class="messages">       {% for message in messages %}         <li class="{{ message.tags }}">        {{ message|safe }}            <a href="#" class="close"> </a>         </li>       {% endfor %}     </ul>    {% endif %}
    
messages框架带有一个上下文环境（context）处理器用来添加一个*messages*变量给请求的上下文环境（context）。所以你可以在模板（template）中使用这个变量用来给用户显示当前的messages。

现在，让我们修改*edit*视图（view）来使用messages框架。编辑应用中的*views.py*文件，使*edit*视图（view）如下所示：

    from django.contrib import messages    @login_required    def edit(request):        if request.method == 'POST':        # ...            if user_form.is_valid() and profile_form.is_valid():                user_form.save()                profile_form.save()                messages.success(request, 'Profile updated '\                                         'successfully')            else:                messages.error(request, 'Error updating your profile')        else:            user_form = UserEditForm(instance=request.user)            # ...    
            
当用户成功的更新他们的profile时我们就添加了一条成功的message，但如果某个表单（form）无效，我们就添加一个错误message。

在浏览器中打开 http://127.0.0.1:8000/account/edit/ 编辑你的profile。当profile更新成功，你会看到如下message：

![django-4-17](http://ohqrvqrlb.bkt.clouddn.com/django-4-17.png)

当表单（form）是无效的，你会看到如下message：

![django-4-18](http://ohqrvqrlb.bkt.clouddn.com/django-4-18.png)

##创建一个定制的认证（authentication）后台
Django允许你通过不同的来源进行认证（authentication）。*AUTHENTICATION_BACKENDS*设置包含了所有的认证（authentication）后台给你的项目。默认的，这个设置如下所示：

    ('django.contrib.auth.backends.ModelBackend',)
    
这个默认的*ModelBackend*认证（authentication）用户通过数据库使用的是*django.contrib.auth*中的*User*模型（model）。这适用于你的大部分项目。当然，你还可以创建定制的后台用来认证你的用户通过其他的来源例如一个*LDAP*目录或者其他任何系统。

你可以通过访问 https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#other-authentication-sources 获得更多的信息关于自定义的认证（authentication）。

当你使用*django.contrib.auth*的*authenticate()*函数，Django会尝试认证（authentication）用户一个接一个通过每一个定义在*AUTHENTICATION_BACKENDS*中的后台，直到其中有一个后台成功的认证该用户才会停止进行认证。只有所有的后台都无法进行用户认证（authentication），他或她才不会在你的站点中认证（authentication）通过。

Django提供了一个简单的方法来定义你自己的认证（authentication）后台。一个认证（authentication）后台就是一个类提供了如下两种方法：

* authenticate()：将用户信息当成参数，如果用户成功的认证（authentication）就需要返回*True*，反之，需要返回*False*。
* get_user()：将用户的ID当成参数然后需要返回一个用户对象。

创建一个定制认证（authentication）后台非常容易就是编写一个Python类实现上面两个方法。我们要创建一个认证（authentication）后台让用户在我们的站点中使用他们e-mail替代他们的用户名来进行认证（authentication）。

在你的*account*应用中创建一个新的文件命名为*authentication.py*，为它添加如下代码：

    from django.contrib.auth.models import User    class EmailAuthBackend(object):        """        Authenticate using e-mail account.        """        def authenticate(self, username=None, password=None):            try:                user = User.objects.get(email=username)                if user.check_password(password):                    return user                return None            except User.DoesNotExist:                return None        def get_user(self, user_id):            try:                return User.objects.get(pk=user_id)            except User.DoesNotExist:    return None

这是一个简单的认证（authentication）后台。`authenticate()`方法接收了*username*和*password*两个可选参数。我们可以使用不同的参数，但是我们需要使用*username*和*password*来确保我们的后台可以立马在认证（authentication）框架视图（views）中工作。以上代码完成了以下工作内容：

* authenticate()：我们尝试获取一个用户通过给予的e-mail地址和使用*User*模型（model）中内置的`check_password()`方法来检查密码。这个方法会对给予密码进行哈希化来和数据库中存储的加密密码进行匹配。
* get_user()：我们获取一个用户通过*user_id*参数。Django使用这个后台来认证用户之后取回*User*对象放置到持续的用户会话中。

编辑项目中的*settings.py*文件添加如下设置：

    AUTHENTICATION_BACKENDS = (       'django.contrib.auth.backends.ModelBackend',       'account.authentication.EmailAuthBackend',    )
    
我们保留默认的*ModelBacked*用来保证用户仍然可以通过用户名和密码进行认证，接着我们包含进了我们自己的email-based认证（authentication）后台。现在，在浏览器中打开 http://127.0.0.1:8000/account/login/ 。请记住，Django会对每个后台都尝试进行用户认证（authentication），所以你可以使用用户名或者使用email来进行无缝登录。

> *AUTHENTICATION_BACKENDS*设置中的后台排列顺序。如果相同的认证信息在多个后台都是有效的，Django会停止在第一个成功认证（authentication）通过的后台，不再继续进行认证（authentication）。

##为你的站点添加社交认证（authentication）
你可以希望给你的站点添加一些社交认证（authentication）服务，例如 *Facebook*，*Twitter*或者*Google*(国内就算了- -|||)。*Python-social-auth*是一个Python模块提供了简化的处理为你的网站添加社交认证（authentication）。通过使用这个模块，你可以让你的用户使用他们的其他服务的账号来登录你的网站。你可以访问 https://github.com/omab/python-social-auth 得到这个模块的代码。

这个模块自带很多认证（authentication）后台给不同的Python框架，其中就包含Django。

使用pip来安装这个包，打开终端运行如下命令：

    pip install python-social-auth==0.2.12
    
安装成功后，我们需要在项目*settings.py*文件中的*INSTALLED_APPS*设置中添加*social.apps.django_app.default*：

    INSTALLED_APPS = (        #...        'social.apps.django_app.default',    )

这个*default*应用会在Django项目中添加*python-social-auth*。现在，运行以下命令来同步*python-social-auth*模型（model）到你的数据库中：

    python manage.py migrate
    
你会看到如下*default*应用的数据迁移输出：

    Applying default.0001_initial... OK    Applying default.0002_add_related_name... OK    Applying default.0003_alter_email_max_length... OK

*python-social-auth*包含了很多服务的后台。你可以访问 https://python-social-auth.readthedocs.org/en/latest/backends/index.html#supported-backends 看到所有的后台支持。

我们要包含的认证（authentication）后台包括*Facebook*，*Twitter*，*Google*。

你需要在你的项目中添加社交登录URL模型。打开*bookmarks*项目中的主*urls.py*文件，添加如下URL模型：

    url('social-auth/',        include('social.apps.django_app.urls', namespace='social')),

为了确保社交认证（authentication）可以工作，你还需要配置一个*hostname*，因为有些服务不允许重定向到*127.0.0.1*或*localhost*。为了解决这个问题，在*Linux*或者*Mac OSX*下，编辑你的*/etc/hosts*文件添加如下内容：

    127.0.0.1 mysite.com
    
这是用来告诉你的计算机指定*mysite.com* *hostname*指向你的本地机器。如果你使用*Windows*,你的hosts文件在 *C:\Winwows\ System32\Drivers\etc\hosts*。

为了验证你的*host*重定向是否可用，在浏览器中打开 http://mysite.com:8000/account/login/ 。如果你看到你的应用的登录页面，*host*重定向已经可用。

###使用Facebook认证（authentication）
**（译者注：以下的几种社交认证操作步骤可能已经过时，请根据实际情况操作）**
为了让你的用户能够使用他们的Facebook账号来登录你的网站，在项目*settings.py*文件中的*AUTHENTICATION_BACKENDS*设置中添加如下内容：

    'social.backends.facebook.Facebook2OAuth2',
    
为了添加Facebook的社交认证（authentication），你需要一个Facebook开发者账号，然后你必须创建一个新的Facebook应用。在浏览器中打开 https://developers.facebook.com/apps/?action=create 点击**Add new app**按钮。点击**Website**平台然后为你的应用取名为*Bookmarks*，输入 http://mysite.com:8000/ 作为你的网站URL。跟随快速开始步骤然后点击**Create App ID**。

回到网站的Dashboard。你会看到类似下图所示的：

![django-4-19](http://ohqrvqrlb.bkt.clouddn.com/django-4-19.png)

拷贝**App ID**和**App Secret**关键值，将它们添加在项目中的*settings.py**文件中，如下所示：

    SOCIAL_AUTH_FACEBOOK_KEY = 'XXX' # Facebook App ID    SOCIAL_AUTH_FACEBOOK_SECRET = 'XXX' # Facebook App Secret
    
此外，你还可以定义一个*SOCIAL_AUTH_FACEBOOK_SCOPE*设置如果你想要访问Facebook用户的额外权限，例如：

    SOCIAL_AUTH_FACEBOOK_SCOPE = ['email']
    
最后，打开*registration/login.html*模板（template）然后添加如下代码到*content* block中：

    <div class="social">      <ul>        <li class="facebook"><a href="{% url "social:begin" "facebook" %}">Sign in with Facebook</a></li>      </ul> 
    </div>

在浏览器中打开 http://mysite.com:8000/account/login/ 。现在你的登录页面会如下图所示：

![django-4-20](http://ohqrvqrlb.bkt.clouddn.com/django-4-20.png)

点击**Login with Facebook*按钮。你会被重定向到Facebook，然后你会看到一个对话询问你的权限是否让*Bookmarks*应用访问你的公共Facebook profile：

![django-4-21](http://ohqrvqrlb.bkt.clouddn.com/django-4-21.png)

点击**Okay**按钮。Python-social-auth会对认证（authentication）进行操作。如果每一步都没有出错，你会登录成功然后被重定向到你的网站的dashboard页面。请记住，我们已经使用过这个URL在*LOGIN_REDIRECT_URL*设置中。就像你所看到的，在你的网站中添加社交认证（authentication）是非常简单的。

###使用Twitter认证（authentication）
为了使用Twitter进行认证（authentication），在项目*settings.py*中的*AUTHENTICATION_BACKENDS*设置中添加如下内容：

    'social.backends.twitter.TwitterOAuth',
    
你需要在你的Twitter账户中创建一个新的应用。在浏览器中打开 https://apps.twitter.com/app/new 然后输入你应用信息，包含以下设置：

* Website：http://mysite.com:8000/
* Callback URL：http://mysite.com:8000/social-auth/complete/twitter/

确保你勾选了复选款**Allow this application to be used to Sign in with Twitter**。之后点击**Keys and Access Tokens**。你会看到如下所示信息：

![django-4-22](http://ohqrvqrlb.bkt.clouddn.com/django-4-22.png)

拷贝**Consumer Key**和**Consumer Secret**关键值，将它们添加到项目*settings.py*的设置中，如下所示：

    SOCIAL_AUTH_TWITTER_KEY = 'XXX' # Twitter Consumer Key    SOCIAL_AUTH_TWITTER_SECRET = 'XXX' # Twitter Consumer Secret
    
现在，编辑*login.html*模板（template），在`<ul>`元素中添加如下代码：

    <li class="twitter"><a href="{% url "social:begin" "twitter" %}">Login with Twitter</a></li>

在浏览器中打开 http://mysite.com:8000/account/login/ 然后点击**Login with Twitter**链接。你会被重定向到Twitter然后它会询问你授权给应用，如下所示：

![django-4-23](http://ohqrvqrlb.bkt.clouddn.com/django-4-23.png)

点击**Authorize app**按钮。你会登录成功并且重定向到你的网站dashboard页面。

###使用Google认证（authentication）
Google提供*OAuth2*认证（authentication）。你可以访问 https://developers.google.com/accounts/docs/OAuth2 获得关于Google OAuth2的信息。

首先，你徐闯创建一个*API key*在你的Google开发者控制台。在浏览器中打开 https://console.developers.google.com/project 然后点击**Create project**按钮。输入一个名字然后点击**Create**按钮，如下所示：

![django-4-24](http://ohqrvqrlb.bkt.clouddn.com/django-4-24.png)

在项目创建之后，点击在左侧菜单的**APIs & auth**链接，然后点击**Credentials**部分。点击**Add credentials**按钮，然后选择**OAuth2.0 client ID**，如下所示：

![django-4-25](http://ohqrvqrlb.bkt.clouddn.com/django-4-25.png)

Google首先会询问你配置同意信息页面。这个页面将会展示给用户告知他们是否同意使用他们的Google账号来登录访问你的网站。点击*Configure consent screen*按钮。选择你的e-mail地址，填写*Bookmarks*为**Product name**，然后点击**Save**按钮。这个给你的项目使用的同意信息页面将会配置完成然后你会被重定向去完成创建你的Client ID。

在表单（form）中填写以下内容：

* Application type： 选择**Web application**
* Name： 输入*Bookmarks*
* Authorized redirect URLs：输入 http://mysite.com:8000/social-auth/complete/google-oauth2/

这表单（form）将会如下所示：

![django-4-26](http://ohqrvqrlb.bkt.clouddn.com/django-4-26.png)

点击**Create**按钮。你将会获得**Client ID**和**Client Secret**关键值。在你的*settings.py*中添加它们，如下所示：

    SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = '' # Google Consumer Key    SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = '' # Google Consumer Secret
    
在Google开发者控制台的左方菜单，**APIs & auth**部分的下方，点击**APIs**链接。你会看到包含所有Google Apis的列表。点击 **Google+ API**然后点击**Enable API**按钮在以下页面中：

![django-4-27](http://ohqrvqrlb.bkt.clouddn.com/django-4-27.png)

编辑*login.html*模板（template）在`<ul>`元素中添加如下代码：

    <li class="google"><a href="{% url "social:begin" "google" %}">Login with Google</a></li>

在浏览器中打开 http://mysite.com:8000/account/login/ 。登录页面将会看上去如下图所示：

![django-4-28](http://ohqrvqrlb.bkt.clouddn.com/django-4-28.png)

点击**Login with Google**按钮。你将会被重定向到Google并且被询问权限通过我们之前配置的同意信息页面：

![django-4-29](http://ohqrvqrlb.bkt.clouddn.com/django-4-29.png)

点击**Accept**按钮。你会登录成功并重定向到你的网站的dashboard页面。

我们已经添加了社交认证（authentication）到我们的项目中。python-social-auth模块还包含更多其他非常热门的在线服务。

###总结
在本章中，你学习了如何创建一个认证（authentication）系统到你的网站并且创建了定制的用户profile。你还为你的网站添加了社交认证（authentication）。

在下一章中，你会学习如何创建一个图片收藏系统（image bookmarking system），生成图片缩微图，创建AJAX视图（views）。

###译者总结
终于写到了这里，呼出一口气，第四章的页数是前几章的两倍，在翻译之前还有点担心会不会坚持不下去，不过看样子我还是坚持了下来，而且发现一旦翻译起来就不想停止（- -|||莫非心中真的有翻译之魂！？）。这一章还是比较基础，主要介绍了集成用户的认证系统到网站中，比较有用的是通过第三方的平台账户登录，可惜3个平台Facbook，Twitter，Google国内都不好访问，大家练习的时候还是用国内的QQ，微信，新浪等平台来练习吧。第五章的翻译不清楚什么时候能完成，也许过年前也可能过年后，反正不管如何，这本书我一定要翻译到最后！

最后，请保佑我公司年会让我抽到特大奖，集各位祈祷之力，哈哈哈哈！


