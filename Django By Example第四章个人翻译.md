#第四章
##创建一个社区网站
在上一章中，你学习了如何创建站点地图（sitemaps）和feeds，你还为你的blog应用创建了一个搜索引擎。在本章中，你将开发一个社区应用。你会创建为用户创建一些功能，例如：登录，登出，编辑，以及重置他们的密码。你会学习如何为你的用户创建一个定制的profile，你还会为你的站点添加社交认证。

本章将会覆盖一下几点：
* 使用认证（authentication）框架
* 创建用户注册视图（views）
* 通过一个定制的profile模型（model）扩展*User*模型（model）
* 使用*python-social-auth*添加社交认证

让我们开始创建我们的新项目吧。

###创建一个社交网站项目
我们将要创建一个社交应用允许用户分享他们在网上找到的图片。我们需要为这个项目构建以下元素：
* 一个用来给用户注册，登录，编辑他们的profile，以及改变或重置密码的认证（authentication）系统
* 一个允许用户用来关注其他人的关注系统（这里原文是follow，‘跟随’，感觉用‘关注’更加适合点）
* 为用户从其他任何网站分享过来的图片进行展示和打上书签
* 每个用户都有一个活动流允许用户看到他们关注的人上传的内容

本章主要讲述第一点。

###开始你的社交网站项目
打开终端使用如下命令行为你的项目创建一个虚拟环境并且激活它：
    
    mkdir evn
    virtualenv evn/bookmarks
    source env/bookmarks/bin/activate
    
shell提示将会展示你激活的虚拟环境，如下所示：

    (bookmarks)laptop:~ zenx$
    
通过以下命令在你的虚拟环境中安装Django：

    pip install Django==1.8.6
    
运行以下命令来创建一个新项目：

    django-admin statproject bookmarks

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

这个认证（authentication）框架位于*django.contrib.auth*，被其他Django的*contrib*包调用。请记住你使用过这个认证（authentication）框架在*第一章 创建一个Blog应用*中用来为你的blog应用创建了一个超级用户来使用管理站点。

当你使用*startproject*命令创建一个新的Django项目，认证（authentication）框架已经在你的项目设置中默认包含。它是由*django.contrib.auth*应用和你的项目设置中的*MIDDLEWARE_CLASSES*中的两个中间件类组成，如下：
* AuthenticationMiddlwware:使用会话（sessions）将用户和请求（requests）进行关联
* SessionMiddleware:通过请求（requests）操作当前会话（sessions）

一个中间件一个是在请求和响应阶段带有全局执行方法的类。你会在本书中的很多场景中使用到中间件。你将会学习如何创建一个定制的中间件在*第十三章 Going Live*。

这个认证（authentication）系统还包含了以下模型（models）：
* User：一个用户模型（model）包含基础字段；这个模型（model）的主要字段有：username,password,email,first_name,last_name,is_active。
* Group：一个组模型（model）用来分类用户
* Permission：执行特定操作的标识

这个框架还包含默认的认证（authentication）视图（views）和表单（forms），我们之后会用到。

###创建一个log-in视图（view）
我们将要开始使用Django认证（authentication）框架允许用户登录我们的网站。。我们的视图（view）需要执行以下操作来登录一个用户：
* 1、通过提交的表单（form）获取username和password
* 2、通过存储在数据库中的数据对用户进行认证
* 3、检查用户是否可用
* 4、登录用户到网站中并且开始一个认证（authentication）会话（session）

首先，我们要创建一个登录表单（form）。在你的account应用目录下创建一个新的*forms.py*文件，添加如下代码：

    from django import forms    class LoginForm(forms.Form):        username = forms.CharField()        password = forms.CharField(widget=forms.PasswordInput)

这个表单（form）被用来通过数据库认证用户。请注意，我们使用*PsswordInput*控件来渲染HTML`input`元素，包含`type="password`属性。编辑你的*account*应用中的*views.py*文件，添加如下代码：

    from django.http import HttpResponse    from django.shortcuts import render    from django.contrib.auth import authenticate, login    from .forms import LoginForm    def user_login(request):        if request.method == 'POST':            form = LoginForm(request.POST)            if form.is_valid():                cd = form.cleaned_data                user = authenticate(username=cd['username'],                                    password=cd['password'])                if user is not None:                    if user.is_active:                        login(request, user)                        return HttpResponse('Authenticated '\                                            'successfully')
                    else:                        return HttpResponse('Disabled account')
                else:            return HttpResponse('Invalid login')        else:            form = LoginForm()        return render(request, 'account/login.html', {'form': form})
        
        
以上就是我们在视图（view）中所作的基本登录操作：当*user_login*被一个GET请求（request）调用，我们实例化一个新的登录表单（form）通过`form = LoginForm()`在模板（template）中展示它。当用户通过POST方法提交表单（form），我们执行以下操作：
* 1、使用提交的数据实例化表单（form）通过使用`form = LoginForm(request.POST)`
* 2、检查这个表单是否有效。如果无效，我们在模板（template）中展示表单错误信息（举个例如，比如用户没有填写其中一个字段就进行提交）
* 3、如果提交的数据是有效的，我们通过数据库对这个用户进行认证（authentication）通过使用`authenticate()`方法。这个方法带入一个*username*和一个*password*并且返回一个用户对象如果这个用户成功的进行了认证，或者是*None*。如果用户没有被认证通过，我们返回一个*HttpResponse*展示一条消息。
* 4、如果这个用户认证（authentication）成功，我们使用`is_active`属性来检查用户是否可用。这是一个Django的*User*模型（model)属性。如果这个用户不可用，我们返回一个*HttpResponse*展示信息。
* 5、如果用户可用，我们登录这个用户到网站中。我们通过调用`login()`方法集合用户到会话（session）中然后返回一条成功消息。

> 请注意*authentication*和*login*中的不同点：`authenticate()`检查用户认证信息然后返回一个用户对象如果用户是正确的；`login()`集合用户到当前的会话（session）中。

现在，你需要为这个视图（view）创建一个URL模式。在你的*account*应用目录下创建一个新的*urls.py*文件，添加如下代码：

    from django.conf.urls import url    from . import views    urlpatterns = [        # post views        url(r'^login/$', views.user_login, name='login'),    ]

编辑位于你的*bookmarks*项目目录下的*urls.py*文件，将*account*应用下的URL模式包含进去：

    from django.conf.urls import include, url    from django.contrib import admin    urlpatterns = [        url(r'^admin/', include(admin.site.urls)),
        url(r'^account/',include('account.urls')),    ]
    
这个登录视图（view）现在已经可以通过URL进行访问。现在是时候为这个视图（view）创建一个模板。因为之前你没有这个项目的任何模板，你可以开始创建一个主模板（template）可以被登录模板（template）继承使用。创建以下文件和结构在*account*应用目录中：

    templates/        account/            login.html        base.html
        
编辑*base.html*文件添加如下代码：

    {% load staticfiles %}    <!DOCTYPE html>    <html>    <head>      <title>{% block title %}{% endblock %}</title>      <link href="{% static "css/base.css" %}" rel="stylesheet">    </head>    <body>      <div id="header">        <span class="logo">Bookmarks</span>      </div>      <div id="content">        {% block content %}        {% endblock %}      </div>    </body>    </html>
    
以上将是这个网站的基础模板（template）。就像我们在上一章项目中做过的一样，我们在这个住模板（template）中包含了CSS样式。你可以在


