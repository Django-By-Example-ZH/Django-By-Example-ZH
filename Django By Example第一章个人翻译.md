（译者注：本人目前在杭州某家互联网公司工作，岗位是测试研发，非常喜欢python，目前已经使用Django为公司内部搭建了几个自动化平台，因为没人教没人带，基本靠野路子自学，走过好多弯路，磕磕碰碰一路过来，前段时间偶尔看到《Django By Example》这本书，瞬间泪流满面，当初怎么没有找到这么好的Django教程。在看书的过程中不知道怎么搞的突然产生了翻译全书的想法，正好网上找了下也没有汉化的版本，所以准备踏上这条不归路。鉴于本人英文水平极低（四级都没过），单纯靠着有道词典和自己对上下文的理解以及对书中每行代码都保证敲一遍并运行的情况下，请各位在读到语句不通的时候或看不懂的地方请告诉我，我会及时进行改正。翻译全书，主要也是为了培养自己的英文阅读水平（口语就算了），谁叫好多最新最有用的计算机文档都是用英文写的，另外也可以培养自己的耐心，还可以分享给其他人，就这样。）

# 第一章
## 创建一个博客应用
在这本书中，你将学习如何创建一个完整的Django项目，并在生产环境中使用。假如你还没有安装Django，在这一章的第一部分你将学习如何安装。本章会覆盖如何使用Django去创建一个简单的博客应用。本章的目的是使你对该框架的工作有个基本概念，懂得不同的组件之间是如何产生交互，并且给你一些简单的技能来创建Django项目通过使用一些基本功能。你会被引导创建好一个完整的项目但是不会对所有的细节进行详细说明。不同的框架组件将在本书以后的章节中进行介绍。
本章会覆盖一下几点：

* 安装Django并创建你的第一个项目
* 设计models并且生成迁移
* 给你的models创建一个管理站点
* QuerySet和managers的操作
* 创建views，templates和URLs
* 给列表views添加页数
* 使用Django的内置views

## 安装Django
如果你已经安装好了Django，你可以直接略过这部分跳到*创建你的第一个项目*。Django是python的一个包因此将在安装在python的环境中。如果你还没有安装Django，这里有一个快速的指南帮助你在本地开发环境安装Django。

Django需要在Python2.7或者3版本上才能更好的工作。在本书的这个例子中，我们使用的将是Python 3。如果你使用LInux或者Max OSX，你可能已经安装好了Python。如果你不确定你的计算机中是否安装了Python，你可以在终端中输入 *python* 来确定。如果你看到以下类似的提示，说明你的计算机中已经安装好了Python:

> Python 3.5.0 (v3.5.0:374f501f4567, Sep 12 2015, 11:00:19)
> [GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
> Type "help", "copyright", "credits" or "license" for more information.
>
> \>\>\>

如果你计算机中安装的Python版本低于3，或者没有安装，下载并安装Python 3.5.0 从http://www.python.org/download/。

由于你将使用Python3，你没必要再安装一个数据库。这个Python版本自带SQLite数据库。SQLLite是一个轻量级的数据库你可以在Django开发环境中使用。如果你准备在生产环境中部署你的应用，你应该使用一个更高级的数据库，比如PostgreSQL，MySQL或Oracle。你能获取到更多的信息关于数据和Django的结合通过访问https://docs.djangoproject.com/en/1.8/topics/install/#database-installation 。

## 创建一个独立的Python环境
强烈建议你使用virtualenv来创建一个独立的Python环境，这样你可以使用不同的包版本对应不同的项目，这比直接安装Python包更加的实用。其他高级的地方是你在virtualenv中你不需要任何管理员权限来安装Python包。在终端中运行以下命令来安装virtualenv：

    pip install virtualenv

当你安装好virtualenv之后，通过以下命令来创建一个独立的环境：

    virtualenv my_env

这个命令会在你的Python环境中创建一个my_env/目录。当你的虚拟环境被激活的时候所有已经存在的Python库都会自动带入 my_env/lib/python3.5/site-packages dircory。
如果你的系统既有Python2.X又有Python3.X，你必须告诉虚拟环境使用哪个版本。通过以下命令你可以定位已安装的Python3来创建虚拟环境：
>zenx\$ *which python3*
/LibraryFrameworks/Python.framework/Versions/3.5/bin/python3
zenx\$ *virtualenv my_env -p /Library/Frameworks/Python.framework/Versions/3.5/bin/python3*

通过以下命令来激活你的虚拟环境：

    source my_env/bin/activate

shell提示将会包含激活的虚拟环境名，像下面一样：

    (my_evn)laptop:~ zenx$

你可以使用*deactivate*命令随时停用你的虚拟环境。
你可以获取更多的信息关于虚拟环境通过访问https://virtualenv.pypa.io/en/latest/。
你可以使用virtualenvwrapper工具来方便的创建和管理你的虚拟环境。你可以在http://virtualenvwrapper.readthedocs.org/en/latest/ 下载该工具。

## 使用pip安装Django
pip是安装Django的第一选择。Python3.5自带pip，你可以找到pip的安装指令通过访问 https://pip.pypa.io/en/stable/installing/
运行以下命令来通过pip安装Django：

    pip install Django==1.8.6

Django将会被安装在虚拟环境的*site-packages/*目录下。
现在检查Django是否成功安装。在终端中运行*python*并且导入Django来检查它的版本：
>\>\>\> import django
\>\>\> django.VERSION
DjangoVERSION（1, 8, 5, 'final', 0)

如果你获得了以上输出，恭喜你，Django已经在你的计算机里成功安装。
Django也可以使用别的方法来安装。你可以找到更多的信息通过访问https://docs.djangoproject.com/en/1.8/topics/install/。

## 创建你的第一个项目
我们的第一个项目是一个完整的博客站点。Django提供了一个命令允许你方便的创建一个初始化的项目文件结构。在终端中运行以下命令：

    django-admin startproject mysite

该命令将会创建一个名为*mysite*的项目。
让我们来看下生成的项目结构：

    mysite/
        manage.py
        mysite/
            __init__.py
            settings.py
            urls.py
            wsgi.py

让我们来了解一下这些文件的功能：
* *manage.py*:一个用来操作项目的使用命令行。它是*django-admin.py*工具的包装。你不需要编辑这个文件。
* *mysite/*:你的项目目录，由以下的文件组成：

    __init__.py:一个空文件告诉Python这个*mysite*目录是一个Python模块。
    settings.py:项目的设置和配置。里面包含一些初始化的设置信息。
    urls.py:这是你存放URL的地方。这里定义的每一个URL都映射到一个view。
    wsgi.py:配置你的项目可以通过WSGI应用运行
settings.py文件包含使用SQLite数据库的基础配置和一个添加到你的项目中的默认Django应用列表。我们需要为初始化的应用创建一些tables在数据库中。
打开终端执行以下命令：

    cd mysite
    python manage.py migrate

你将会看到以下的类似输出：

    Rendering model states... DONE
    Applying contenttypes.ooo1_initial... OK
    Applying auth.0001_initial... OK
    Applying admin.0001_initial... OK
    Applying contenttypes.0002_remove_content_type_name... OK
    Applying auth.0002_alter_permission_name_max_length... OK
    Applying auth.0003_alter_user_email_max_length...OK
    Applying auth.0004_alter_user_username_opts... OK
    Applying auth.0005_alter_user_last_login_null... OK
    Applying auth.0006_require_contenttypes_0002... OK
    Applying sessions.0001_initial... OK
    
这些初始化的应用tables将会在数据库中创建。你马上会学习到一些关于*migrate*的管理命令。

## 运行开发环境
Django自带一个轻量级的web服务来快速运行你的代码，而不需要花额外的时间来配置一个生产用的服务。当你运行Django的开发服务，它会一直检查你的代码变化。它会自动重启，解放你的双手在代码变化后不用手动进行重启。然而，它可能无法注意到一些操作，例如在项目中添加了一个新文件，所以你在某些情况下还是需要手动进行重启操作。

在你的项目主目录下运行一下代码来启动开发服务：

    python manage.py runserver

你会看到以下类似的输出：

    Performing system checks...
    
    System check identified no issues (0 silenced).
    November 5, 2015 - 19:10:54
    Django version 1.8.6, using settings 'mysite.settings'
    Starting development server at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

现在，在浏览器中打开 http://127.0.0.1:8000/ ，你会看到一个告诉你项目成功运行的页面，如下图所示：
![](/Users/xulingfeng/Documents/学习/django-1-1.png
)

你可以指定Django在自定义的host和port上运行开发服务，或者告诉它读取不同的设置文件来运行你的项目。例如：你可以运行以下 manage.py命令：

    python manage.py runserver 127.0.0.1:8001 --settings=mysite.settings

这个命令可以方便的处理多套环境对于不同设置的需求。记住，这个服务只是单纯用来开发使用，不适合在生产环境中使用。为了在生产环境中部署Django，你需要使用真实的web服务让它运行成一个WSGI应用例如Apache，Gunicorn或者uWSGI。你能够获取到更多的信息关于如何在不同的web服务中部署Django，访问 https://docs.djangoproject.com/en/1.8/howto/deployment/wsgi/

##项目设置

让我们打开settings.py文件来看看你的项目配置内容。在文件中有许多设置是Django内置的，但这些只是所有Django可用配置的一部分。你可以看到所有的设置和默认值通过访问 https://docs.djangoproject.com/en/1.8/ref/settings/

以下列出的设置非常值得一看：

* DEBUG 一个布尔型用来开启或关闭项目的debug模式。如果设置为True，Django将会在你的应用抛出没有被捕获的异常时显示一个详细的错误页面。当你准备部署项目到生产环境，请记住一定要关闭debug模式。永远不要在生产环境中打开debug模式因为那会暴露你项目中的敏感数据。
* ALLOWED_HOSTS 当debug模式开启或者运行测试的时候不会起作用。一旦你准备部署你的项目到生产环境并且关闭了debug模式，你就必须添加你的域名或host在这个设置中来允许访问你的Django站点。
* INSTALLED_APPS 这个设置你将会在所有的项目中都进行笔记。这个设置告诉Django有哪些应用会激活在这个项目中。默认是的，Django包含以下应用：
  * django.contrib.admin：这是一个管理站点。
  * django.contrib.auth：这是一个权限框架。
  * django.contrib.contenttypes：这是一个内容类型的框架。
  * django.contrib.sessions：这是一个session会话框架。
  * django.contrib.messages：这是一个消息框架。
  * django.contrib.staticfiles：这是一个用来管理静态文件的框架
* MIDDLEWARE_CLASSES 是一个包含可执行中间件的元祖。
* ROOT_URLCONF 指明你的应用定义的Python模块的主URL模式。
* DATABASES 是一个包含了所有在项目中使用的数据库的设置的字典。里面一定有一个默认的数据库。默认的配置使用的是SQLite3数据库。
* LANGUAGE_CODE 定义Django站点的默认语言。

不要担心你目前还看不懂这些设置的含义。你将会熟悉这些设置在接下里的章节中。

##项目和应用

现在让我们创建你的第一个Django应用。我们将要创建一个博客应用。在你的项目主目录下，运行以下命令：

    python manage.py startapp blog

这个命令会创建应用的基本目录结构，类似下方结构：

    blog/
    	__init__.py
    	admin.py
    	migrations/
    		__init__.py
    	models.py
    	tests.py
    	views.py

这些文件的含义：

- admin.py: 在这儿你可以注册你的models使它们在Django的管理页面中展示。不一定要使用Django的管理页面。
- migrations: 这个目录将会包含你的应用的数据库migrations。Migrations允许Django跟踪你的model变化来同步数据库。
- models.py: 应用的数据models。所有的Django应用都需要有一个models.py，尽管这个文件可以是空的。
- tests.py：在这儿你可以为你的应用创建测试用例。
- views.py：你的应用逻辑处理将会放在这儿。每一个view都会受到一个HTTP请求，处理处理，并且返回一个响应。

##设计博客的数据模型

我们将要为你的博客设计初始的数据模型。每一个model都是一个Python类，继承了django.db.models.model,其中的每一个属性对应了一个数据库字段。Djnago将会为models.py中的每一个定义好的model创建好一张表。当你创建好一个model，Django就能提供一个非常实用的API来方便的查询数据库。

首先，我们定义一个POST model。在blog应用下的models.py文件中添加以下内容：

    from django.db import models
    from django.utils import timezone
    from django.contrib.auth.models import User
    class Post(models.Model):
    STATUS_CHOICES = (
    ('draft', 'Draft'),
    ('published', 'Published'),
    )
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250,
    unique_for_date='publish')
    author = models.ForeignKey(User,
    related_name='blog_posts')
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    status = models.CharField(max_length=10,
    choices=STATUS_CHOICES,
    default='draft')
    class Meta:
    ordering = ('-publish',)
    def __str__(self):
    return self.title

这是一个给博客帖子使用的基础model。让我们来看下刚才在model中定义的各个字段含义：

* title： 这个字段对应帖子的标题。这是一个CharField，在SQL数据库中对应VARCHAR类型
* slug：这是一个用在URLs中的字段。一个slug是一个短标签之包含字母，数字，下划线或连接线。我们将通过使用slug字段来构建一个漂亮的，友好的URLs给我们的博客帖子。我们可以给每个帖子使用日期和slug来构建URLs通过添加 unique_for_date参数给这个字段。
* author：这是一个 ForeignKey。这个字段定义了一个many-to-one（多对一）的关系。我们告诉Django每一个帖子是被一个用户编辑而一个用户又能编辑多个帖子。通过这个字段，Django将会在数据库中通过有关联的model主键来创建一个外键。在这个例子中，我们关联上了django权限系统的User model。我们指定了User和Post之间关联的名字，通过related_name属性。在之后我们将会学习到更多相关的内容。
* body：这是帖子的内容。这个字段是TextField，在SQL数据中对应TEXT类型。
* publish：这个字段存储了帖子发布的时间。我们使用Djnago的 timezone now方法来设定默认值。
* created：这个字段存储了帖子创建的时间。因为我们在这儿使用了auto_now_add，当一个对象被创建的时候这个字段会自动保存当前日期。
* updated：这个字段存储了帖子更新的时间。因为我们在这儿使用了auto_now，当我们保存一个对象的时候当前字段将会自动更新到当前日期。
* status：这个字段是用来展示帖子的状态。我们使用了一个choices参数，所有这个字段的值只能在给予的选择内容中选择。

就像你所看到的的，Django带来了不同的字段类型所以你能够定义你的models。你可以找到所有的字段类型通过访问 https://docs.djangoproject.com/en/1.8/ref/models/fields/

在model中的*Meta*类包含元数据。我们告诉Django查询数据库的时候默认返回的是根据*publish*字段进行降序排列过的结果。我们使用负号来指定降序排列。

*__str__()*方法是对象默认的可读表现。Django将会在很多地方用到它例如管理平台。

> 如果你之前使用过Python2.X，请注意在Python3中所有的strings都使用unicode，因此我们只使用*__str__()*方法。*__unicode__()*方法已经废弃。

在我们处理日期之前，我们需要下载*pytz*模块。这个模块给Python提供时区的定义并且SQLite也需要它来对日期进行操作。在终端中输入以下命令来安装*pytz*：

    pip install pytz

Django内置支持对日期时区的处理。你可以在项目的*settings.py*文件中通过*USE_TZ*来设置激活或停用对时区的支持。当你通过*startproject*命令来创建一个新项目的时候这个设置默认为True。

##激活你的应用
为了让Django能保持跟踪你的应用并且通过models来创建数据表，我们必须激活你的应用。为起到效果，编辑*settings.py*文件在*INSTALLED_APPS*设置中添加*blog*。类似如下显示：

      INSTALLED_APPS = ( 
        'django.contrib.admin',    
        'django.contrib.auth', 
        'django.contrib.contenttypes', 
        'django.contrib.sessions', 
        'django.contrib.messages', 
        'django.contrib.staticfiles',
        'blog',
      )

现在Django已经知道我们项目中的应用是激活的状态并且将会审查其中的models。

##创建和进行数据库迁移
让我们为model在数据库中创建一张表格。Django自带一个迁移系统来跟踪每一次models的变化并且会同步到数据库。*migrate*命令会应用到所有在*INSTALLED_APPS*中的应用，它会根据models来同步数据库。

首先，我们需要为我们刚才创建的心model生成一个迁移。在你的项目主目录下，执行以下命令：

    python manage.py makemigrations blog

你会看到以下输出:

    Migrations for 'blog':
        0001_initial.py;
            - Create model Post

Django已经在blog应用下的migrations目录中创建了一个*0001——initial.py*文件。你可以打开这个文件来看下。

让我们来看下Django根据我们的model将会为在数据库中创建的表格而执行的SQL代码。*sqlmigrate*命令会返回一串SQL而不会去执行。执行以下命令来看下输出：

    python manage.py sqlmigrate blog 0001
    
输出类似如下：

    BEGIN;
    CREATE TABLE "blog_post" ("id" integer NOT NULL PRIMARY KEY
    AUTOINCREMENT, "title" varchar(250) NOT NULL, "slug" varchar(250) NOT
    NULL, "body" text NOT NULL, "publish" datetime NOT NULL, "created"
    datetime NOT NULL, "updated" datetime NOT NULL, "status" varchar(10)
    NOT NULL, "author_id" integer NOT NULL REFERENCES "auth_user" ("id"));
    CREATE INDEX "blog_post_2dbcba41" ON "blog_post" ("slug");
    CREATE INDEX "blog_post_4f331e2f" ON "blog_post" ("author_id");
    COMMIT;
    

这些输出是根据你正在使用的数据库。这些SQL语句是为SQLite准备的。就像你所看见的，Django生成的表名前缀为应用名之后跟上modle的小写（blog_post），但是你也可以自定义表名在models的*Meta*类中通过*db_table*属性。Django会自动为每个model创建一个主键，但是你可以自己指定主键通过设置*primarry_key=True*在期中一个model的字段你上。

让我们根据新的model来同步数据库。运行以下的命令来应用存在的数据迁移：

    python manage.py migrate
    
你应该会看到以下的类似输出：

    Applying blog.0001_initial... OK
    
我们刚刚为*INSTALLED_APPS*中所有的应用进行了数据迁移，包括我们的*blog*应用。在进行了数据迁移之后，数据库将会和我们的models对应。

如果你编辑了models.py文件无论你是添加，删除，还是改变了之前存在的字段，或者你添加了一个新的models，你将必须做一次新的迁移通过使用*makemigrations*命令。数据迁移允许Django来保持对model改变的跟踪。然后你必须通过*migrate*命令来保持数据库与我们的models同步。

##为你的models创建一个管理站点
如今，我们已经定义好了*Post* model，我们将要创建一个简单的管理站点来管理博客的帖子。Django内置了一个非常有用的管理接口来编辑内容。这个Django管理站点会根据你的model元数据进行动态构建并且提供可读的接口来编辑内容。你可以对这个站点进行自由的定制，配置你的models在其中的展示方式。

请记住，*django.contrib.admin*已经被包含在我们项目的*INSTALLED_APPS*设置中，我们不需要再额外添加。

##创建一个超级用户
首先，我们需要创建一个用户来管理这个管理站点。运行以下的命令：
    
    python manage.py createsuperuser
    
你将会看下以下的输出。输入你想要的用户名，邮箱和密码：

    Username (leave blank to use 'admin'): admin
    Email address: admin@admin.com
    Password: ********
    Password (again): ********
    Superuser created successfully.

##Django管理站点
现在，通过`python manage.py runserver`命令来启动开发服务，之后在浏览器中打开 http://127.0.0.1:8000/admin/ 。你会看到管理站点的登录页面，如下所示：
![django-1-2](media/django-1-2.png)

使用你之前创建的超级用户信息登录。你将会看到管理站点的首页，如下所示：
![django-1-3](media/django-1-3.png)

*Group*和*User* models 是Django权限管理框架*django.contrib.auth*的一部份。如果你点击*Users*，你将会看到你之前创建的用户信息。博客应用的*Post*models和*User* model关联在了一起。记住，它们是通过author字段来进行关联的。

##在管理站点中添加你的models
让我们在管理站点中添加你的博客models。编辑博客应用下的admin.py文件，如下所示：

    from django.contrib import admin
    from .models import Post
    admin.site.register(Post)
    
现在，在浏览器中刷新管理站点页面。你会看到你的*Post* model已经在页面中展示，如下所示：
![django-1-4](media/django-1-4.png)

这很简单，对吧？当你在Django的管理页面注册了一个model，你会获得一个非常友好有用的接口通过简单的方式允许你在列表中编辑，创建，删除对象操作。
点击*Posts*右侧的*Add*链接来添加一个新帖子。你将会看到Django为你的model动态生成了一个可编辑输入页面，如下所示：
![django-1-5](media/django-1-5.png)

Django为不同类型的字段使用了不通的控件来展示。即使是复杂的字段例如*DateTimeField*也被展示成一个简单的接口类似一个JavaScript date picker。

填写好表单并点击*Save*按钮。你会被重定向到帖子列表页面获取到一条成功创建帖子的提示，如下所示：
![django-1-6](media/django-1-6.png)


##自定义models的展示形式
现在我们来看下如何自动管理站点。编辑博客应用下的admin.py文件，使之如下所示：
    
    from django.contrib import admin
    from .models import Post
    class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish',
    'status')
    admin.site.register(Post, PostAdmin)
    
我们通过使用继承*ModelAdmin*的自定义类来告诉Django管理站点注册了哪些我们自己的model。在这个类中，我们可以包含一些信息来定制管理页面中如何展示和交互model。*list_display*属性允许你在管理对象的列表页面展示你想展示的model中的字段。

让我们来通过更多的选项来展示model，如使用以下代码：

    class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish',
    'status')
    list_filter = ('status', 'created', 'publish', 'author')
    search_fields = ('title', 'body')
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ('author',)
    date_hierarchy = 'publish'
    ordering = ['status', 'publish']

重新刷新管理站点的页面，现在应该如下所示：
![django-1-7](media/django-1-7.png)

你可以看到在帖子列表页中展示的字段都是你在*list-dispaly*属性中指定的。列表页面现在包含了一个右边栏允许你根据*list_filter*属性中指定的字段来过滤返回结果。在页面还出现了一个搜索框。这是因为我们还定义了一个搜索字段通过使用*search_fields*属性。在搜索框的下方，有个可以通过时间层快速操作的栏。这是通过定义*date_hierarchy*属性。你还能看到这些帖子可以通过*Status*和*Publish*列进行了默认排序。那是因为你指定了默认排序通过使用*ordering*属性。

现在，点击*Add post*链接。你还会在这儿看到更多的改变。当你输入完成新帖子的标题，*slug*字段将会自动填充。我们告诉Django通过输入的标题来填充*slug*字段是通过使用*prepoupulated_fields*属性。同时，如今的*author*字段展示变为了一个搜索控件，这样当你的用户达到成千上万的时候比使用下拉的选择更加的人性化，如下图所示：
![django-1-8](media/django-1-8-1.png)

通过短短的几行代码，我们就在管理站点中自定义了我们的model的展示形式。还有更多的方式可以用来自定义Django的管理站点。在这本书的后面，我们还会进一步讲述。

##使用QuerySet和managers
现在，你已经有个完整的管理站点来管理你的博客内容，是时候学习如何从数据库中检索信息和操作了。Django自带了一个强大的数据库抽象API可以让你轻松的创建，检索，更新以及删除对象。Django的*Object-relational Mapper(ORM)*可以兼容MySQL,PostgreSQL,SQLite以及Oracle。请记住你可以在你项目下的setting.py中编辑*DATABASES*设置来指定数据库。Django可以同时与多个数据库进行工作，这样你可以编写数据库路由来操作数据通过任何你喜欢的方式。

一旦你创建好了你的数据models，Django会提供你一个API来与它们进行交互。你可以找到数据model的官方参考文档通过访问 https://docs.djangoproject.com/en/1.8/ref/models/

##创建对象
打开终端运行以下命令来打开Python shell：

    python manage.py shell
    
然后依次输入以下内容：
    > \>\>\> from django.contrib.auth.models import User
      \>\>\> from blog.models import Post
      \>\>\> user = User.objects.get(username='admin')
      \>\>\> Post.objects.create(title='One more post',
      slug='one-more-post',
      body='Post body.',
      author=user)
      \>\>\> post.save()

让我们来研究下这些代码做了什么。首先，我们取回了一个username为admin的用户对象：
    
    user = User.objects.get(username='admin')

get()方法允许你从数据库取回一个单独的对象。注意这个方法只希望有唯一的一个匹配在查询中。如果在数据库中没有返回结果，这个方法会抛出一个*DoesNotExist*异常，如果数据库返回多个匹配结果，将会抛出一个*MultipleObjectsReturned*异常。当查询执行的时候，所有的异常都是model类的属性。

然后，我们来创建一个拥有自定义标题，slug和内容的*Post*实例，该实例中author关联上我们之前去除的user，如下所示：
    
    post = Post(title='Another post', slug='another-post', body='Postbody.', author=user)

> 这个对象只是存在内存中，并没有执行到数据库中

最后，我们通过使用*save()*方法来保存该对象到数据库中：

    post.save()
    
这步操作将会在之后执行一段SQL的插入语句。我们已经知道如何在内存中创建一个对象并且之后才在数据库中进行插入，但是我们也可以直接在数据库中创建对象通过使用*create()*方法，如下所示：

    Post.objects.create(title='One more post', slug='one-more-post',body='Post body.', author=user)
    
##更新对象
现在，改变这篇帖子的标题并且再次保存对象：

    >>> post.title = 'New title'
    >>> post.save()

这一次，*save()*方法执行了一条更新语句。
> 你对对象的改变一直存在内存中直到你执行到*save()*方法。

##取回对象
Django的*Object-relational mapping(ORM)*是以*QuerySet*为基础。QuerySet是从你的数据库中根据一些过滤条件范围取回的结果进行采集产生的对象。你已经知道通过*get()*方法可以从数据库中取回单独的对象。就像你所看到的：`Post.objects.get()`。每一个Django model至少有一个manager，默认叫做*objects*。你能获得一个QuerySet对象就是通过你的models的manager。获取table中所有的对象，你只需要在默认的objects manager上使用*all()*方法即可：

    >>> all_posts = Post.objects.all()
    
我们创建一个返回数据库中所有对象的QuerySet。注意这个QuerySet并还没有执行。Django的QuerySets是惰性的，他们只会被动的去执行。这样可以保证QuerySet非常效率。如果没有把QuerySet赋值给一个变量，直接在Python shell中编写，这样QuerySet的SQL语句将立马执行因为我们迫使它在输出中返回结果：
    
    >>> Post.objects.all()
    
##使用filter()方法
过滤QuerySet，你可以在manager上使用*filter()*方法。举个例子，我们可以返回所有在2015年发布的帖子，如下所示：
    
    Post.objects.filter(publish__year=2015)
    
你也可以使用多个字段来进行过滤。举个例子，我们可以返回2015年发布的所有作者用户名为admin的帖子，如下所示：

    Post.objects.filter(publish__year=2015, author__username='admin')
    
上面的写法和下面的写法产生的结果是一致的：

    Post.objects.filter(publish__year=2015).filter(author__username='admin')

##使用exclude()
你可以在manager上使用*exclude()*方法来排除某些返回结果。例如：我们可以返回所有2015年发布的帖子但是这些帖子的题目开头不能是*Why*:

    Post.objects.filter(publish__year=2015).exclude(title__startswith='Why')
    
##使用order_by()
你可以对结果进行排序通过在manager上使用*order_by()*方法来对不同的字段进行排序。例如：你可以取回所有对象通过它们的标题进行排序：

    Post.objects.order_by('title')
    
默认是升序。你可以通过负号来指定使用降序：

    Post.objects.order_by('-title')

##删除对象
如果你想删除一个对象，你可以对对象实例进行下面的操作：

    post = Post.objects.get(id=1)    post.delete()
   
> 请注意，删除对象也将删除其中的依赖关系


##QuerySet什么时候会执行
你可以连接许多过滤给QuerySet只要你喜欢而且不会在数据库中执行直到这个QuerySet被执行。QuerySet只有在以下情况中才会执行：
    * 在你第一次迭代它们的时候
    * 当你对它们的实例进行切片：`Post.objects.all()[:3]`
    * 当你对它们进行了打包或缓存
    * 当你对它们调用了*repr()*或*len()*方法
    * 当你明确的对它们调用了*list()*方法
    * 当你再一些声明中测试它们，例如*bool()*, or, and, or if

##创建model manager
我们之前提到过, *objects*是每一个models的默认manager，会返回数据库中所有的结果。但是我们也可以我们的models定义一些自定义的manager。我们准备创建一个自定义的manager来返回所有状态为已发布的帖子。
有两种方式来为你的models添加managers：你可以添加临时的manager方法或者继承manager的QuerySets进行修改。第一种方法类似`Post.objects.my_manager()`,第二种方法类似`Post.my_manager.all()`。我们的manager将会允许我们返回所有帖子通过使用`Post.published`。
编辑你的博客应用下的*models.py文件添加如下代码来创建一个manager:

    class PublishedManager(models.Manager):       def get_queryset(self):           return super(PublishedManager,                        self).get_queryset()\                             .filter(status='published')    class Post(models.Model):       # ...       objects = models.Manager() # The default manager.       published = PublishedManager() # Our custom manager.

*get_queryset()*是返回执行的QuerySet的方法。我们通过使用它来包含我们自定义的过滤在完整的QuerySet中。我们定义我们自定义的manager然后添加到*Post* model中。我们现在可以来执行它。例如，
我们可以返回所有标题开头为*Who*的并且是已经发布的帖子:
    
    Post.published.filter(title__startswith='Who')
    
##构建列表和详情views
现在你学会了一些ORM的基本使用方法，你已经准备好为博客应用创建views了。一个Django view 就是一个Python方法可以接收一个web请求然后返回一个web响应。在vies中通过逻辑处理返回期望的响应。
首先我们会创建一个应用view，然后我们将会为每个view定义一个URL，最后，我们将会为每个views生成的数据通过创建HTML templates来进行渲染。每个view将会通过一个变量来渲染template并且会返回一个经过渲染输出的HTTP响应。

##创建列表和详情views
让我们开始创建一个视图来展示帖子的列表。编辑你的博客应用下中views.py文件，如下所示：

    from django.shortcuts import render, get_object_or_404    from .models import Post    def post_list(request):        posts = Post.published.all()        return render(request,                     'blog/post/list.html',                     {'posts': posts})

你刚创建了你的第一个Django view。*post_list* view将*request*对象作为唯一的参数。记住所有的的view都有需要这个参数。在这个view中，我们获取到了所有状态为已发布的帖子通过使用我们之前创建的*published* manager。

最后，我们使用Django提供的快捷方法*render()*来渲染帖子列表通过给予的template。这个函数将*request*对象，template路径，和给予渲染的变量最为参数。它返回一个渲染文本（一般是HTML代码）*HttpResponse*对象。*render()*方法将许多变量投入template环境中，以便template环境中进行调用。你会在*第三章，扩展你的博客应用*学习到。

让我们创建第二个view来展示一篇单独的帖子。添加如下代码到views.py文件中：
    
    def post_detail(request, year, month, day, post):       post = get_object_or_404(Post, slug=post,                                      status='published',                                      publish__year=year,                                      publish__month=month,                                      publish__day=day)       return render(request,                     'blog/post/detail.html',                     {'post': post})
                    
这是一个帖子详情view。这个view使用year，month，day以及post参数来获取一个发布的帖子通过给予的slug和日期。请注意，当我们创建*Post* model的时候，我们在slgu字段上添加了*unique_for_date*参数。这样我们可以确保只有一个帖子会带有给予的slug，因此，我们能取回单独的帖子通过日期和slug。在这个详情view中，我们通过使用*get_object_or_404()*快捷方法来检索期望的帖子。这个函数能取回匹配给予的参数的对象，或者返回一个HTTP 404异常当没有匹配的对象找到。最后，我们使用*render()*快捷方法来使用template去渲染取回的帖子。

##为你的views添加URL patterns
一个URL pattern 是由一个Python正则表达，一个view，一个全项目范围内的命名组成。Django在运行中会遍历所有URL pattern直到第一个匹配的请求URL才停止。之后，Django导入匹配的URL pattern中的view并且执行它，使用关键字或指定参数来执行一个*HttpRequest*类的实例。
如果你之前没有接触过正则表达式，你需要去稍微了解下，通过访问 https://docs.python.org/3/howto/regex.html 。

在博客应用目录下创建一个*urls.py*文件，输入以下代码：
        
    from django.conf.urls import url    from . import views    urlpatterns = [        # post views        url(r'^$', views.post_list, name='post_list'),        url(r'^(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/'\            r'(?P<post>[-\w]+)/$',            views.post_detail,            name='post_detail'),    ]

第一条URL pattern并没有带入任何参数，它映射*post_list* view。第二条URL pattern带上了一下4个参数映射到*post_detail* view。让我们看下这个URL pattern中的正则表达式：

    * year：需要四位数
    * month：需要两位数。不及两位数，开头带上0，比如 01，02
    * day：需要两位数。不及两位数开头带上0
    * post：可以由单词和连字符组成

> 为每一个应用创建单独的urls.py文件是最好的方法来保证你的应用可以为别的项目再度使用。

现在你需要将你博客中的URL patterns包含到项目的主URL patterns中。编辑你的项目中的mysite文件夹中urls.py文件，如下所示：
    
    from django.conf.urls import include, url    from django.contrib import admin        urlpatterns = [        url(r'^admin/', include(admin.site.urls)), url(r'^blog/', include('blog.urls',
        namespace='blog',
        app_name='blog')),    ]

通过这样的方式，你告诉Django在*blog/*路径下包含了博客应用中的urls.py定义的URL patterns。你可以给他们一个命名空间叫做*blog*这样你可以方便的引用这个URLs组。

##models的标准URLs
你可以使用之前定义的*post_detil* URL对Post对象构建标准URL。Django的惯例是添加*get_absolute_url()*方法给model用来返回一个对象的标准URL。在这个方法中，我们使用*reverse()*方法允许你通过名字和可选的参数来构建URLS。编辑你的models.py文件添加如下代码：
    from django.core.urlresolvers import reverse    Class Post(models.Model):       # ...        def get_absolute_url(self):            return reverse('blog:post_detail',                           args=[self.publish.year,                                self.publish.strftime('%m'),                                self.publish.strftime('%d'),                                self.slug])
请注意，我们通过使用*strftime()*方法来保证个位数的月份和日期需要带上0来构建URL.我们将会在我们的templates中使用*get_absolute_url()*方法。

##为你的views创建templates
我们为我们的应用创建了views和URL patterns。现在该添加模板来展示人性化的帖子了。
在你的博客应用目录下创建一下目录结构和文件：

    templates/    blog/        base.html        post/            list.html            detail.html
            
以上就是我们的templates的文件目录结构。base.html文件将会包含站点主要的HTML结构，分割内容区域和导航栏。list.html和detail.html文件会继承base.html文件来渲染各自的博客帖子列表和详情view。

Django有一个强大的templates语言允许你指定数据的展示形式。它立足于templates tags， 例如 `{% tag %}`, `{{ variable }}`以及templates filters，可以对变量进行过滤，例如 `{{ variable|gilter }}`。你可以找到所有的内置templates tags和filters通过访问 https://docs.djangoproject.com/en/1.8/ ref/templates/builtins/ 。

让我们来编辑base.html文件添加如下代码：

    {% load staticfiles %}    <!DOCTYPE html>    <html>    <head>     <title>{% block title %}{% endblock %}</title>     <link href="{% static "css/blog.css" %}" rel="stylesheet">    </head>    <body>     <div id="content">       {% block content %}       {% endblock %}     </div>     <div id="sidebar">       <h2>My blog</h2>         <p>This is my blog.</p>     </div>    </body>    </html>
    
`{% load staticfiles %}`告诉Django去加载*django.contrib.staticfiles*应用提供的*staticfiles* temaplate tags。通过加载，你可以在这个template中使用`{% static %}`template filter。通过使用这个template filter，你可以包含一些静态文件比如说*blog.css*文件，你可以在本书的范例代码例子中知道，在博客应用的*static/*目录中（译者注：给大家个地址去拷贝 https://github.com/levelksk/django-by-example-book ）拷贝这个目录到你的项目下的相同路径来使用这些静态文件。

你可以看到有两个`{% block %}`tags。这些是用来告诉Django我们想在这个区域中定义一个block。继承这个template的templates可以使用自定义的内容来填充block。我们定义了一个block叫做*title*，另一个block叫做*content*。

让我们编辑*post/list.html*文件使它如下所示：

    {% extends "blog/base.html" %}    {% block title %}My Blog{% endblock %}    {% block content %}     <h1>My Blog</h1>     {% for post in posts %}       <h2>         <a href="{{ post.get_absolute_url }}">           {{ post.title }}         </a>       </h2>       <p class="date">         Published {{ post.publish }} by {{ post.author }}       </p>       {{ post.body|truncatewords:30|linebreaks }}     {% endfor %}    {% endblock %}
    
通过`{% extends %}`template tag，我们告诉Django需要继承*blog/base.html* template。然后我们在*title*和*content* blocks中填充内容。我们通过循环迭代帖子来展示他们的标题，日期，作者和内容，在中标题还包括一个帖子的标准URL链接。在帖子的内容中，我们应用了两个template filters： *truncatewords*用来缩短内容限制在一定的字数内，*linebreaks*用来转换内容中的换行符为HTML的换行符。你可以连接许多tempalte filters只要你喜欢，每一个都会应用到上个输出生成的结果上。

打开终端执行命令`python manage.py runserver`来启动开发环境。浏览器中打开 http://127.0.0.1:8000/blog/ 你会看到运行的结果。注意，你需要添加一些发布状态的帖子才能在这儿看到它们。你会看到如下图所示：
![django-1-9](media/django-1-9.png)

这之后，让我们来编辑*post/detail.html*文件使它如下所示：

    {% extends "blog/base.html" %}    {% block title %}{{ post.title }}{% endblock %}    {% block content %}     <h1>{{ post.title }}</h1>     <p class="date">       Published {{ post.publish }} by {{ post.author }}     </p>     {{ post.body|linebreaks }}    {% endblock %}
    
现在，你可以返回你的浏览器中点击其中一个帖子的标题来看帖子的详细view。你会看到类似的以下输出：
![django-1-10](media/django-1-10.png)

##添加页码
当你开始给你的博客添加内容，你很快会意识到你需要将帖子分页显示。Django有一个内置的pagination类允许你方便的管理分页。

编辑博客应用下的views.py文件导入Django的paginator类修改*post_list*如下所示：

    from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger    def post_list(request):        object_list = Post.published.all()        paginator = Paginator(object_list, 3) # 3 posts in each page        page = request.GET.get('page')        try:            posts = paginator.page(page)        except PageNotAnInteger:            # If page is not an integer deliver the first page            posts = paginator.page(1)        except EmptyPage:            # If page is out of range deliver last page of results            posts = paginator.page(paginator.num_pages)        return render(request,                     'blog/post/list.html',                     {'page': page, 'posts': posts})

pagination是如何工作的：
    * 1. 我们通过希望在每页中显示的对象的数量实例化了*Paginator*类。
    * 2. 我们获取到*page* GET参数来致命页码     
    * 3. 我们通过调用Paginator的 *page()*方法在期望的页面中获得了对象。
    * 4. 如果*page*参数不是一个整形数字，我们就返回第一页的结果。如果这个参数的数字超出了最后的页数，我们就展示最后一页的结果。
    * 5. 我们传递页数并且取到对象给template。

现在，我们必须创建一个template来展示分页处理，它可以被任意的template包含来使用页码。在博客应用的templates文件夹下创建一个新文件命名为*pagination.html*。在文件中添加如下HTML代码：

    <div class="pagination">     <span class="step-links">       {% if page.has_previous %}         <a href="?page={{ page.previous_page_number }}">Previous</a>       {% endif %}       <span class="current">         Page {{ page.number }} of {{ page.paginator.num_pages }}.       </span>         {% if page.has_next %}           <a href="?page={{ page.next_page_number }}">Next</a>         {% endif %}     </span>    </div>    
    
这个分页template期望一个Page对象为了渲染上一页与下一页的链接并且展示页面数据和所有页面的结果。让我们回到*blog/post/list.html*tempalte中将*pagination.html*template包含在`{% content %}`block中，如下所示：

    {% block content %}     ...     {% include "pagination.html" with page=posts %}    {% endblock %}

我们传递给template的Page对象叫做*posts*，我们将分页tempalte包含在帖子列表template中指定参数来对它进行渲染。这是一种方法你可以重复利用你的分页template对不同的models views进行分页处理。

现在，在你的浏览器中打开 http://127.0.0.1:8000/blog/。 你会看到帖子列表的底部有分页信息：
![django-1-11](media/django-1-11.png)

##使用基于类的views
当一个view被调用通过web请求并且返回一个web响应，你还可以将你的views定义成类方法。Django为此定义了基础的view类。它们都从View类继承而来，View类可以操控HTTP方法分派和其他的功能。这是一个可代替的方法来创建你的views。

我们准备使我们的*post_list* view转变为一个基于类的视图通过使用Django提供的通用*ListView*。这个基础view允许你对任意的对象进行排列。

编辑你的博客应用下的views.py文件，如下所示：

    from django.views.generic import ListView    class PostListView(ListView):        queryset = Post.published.all()        context_object_name = 'posts'        paginate_by = 3        template_name = 'blog/post/list.html'
        
这个基于类的的view类似与之前的*post_list* view。这儿，我们告诉ListView做了以下操作：
* 使用一个特定的QuerySet代替取回所有的对象。代替定义一个*queryset*属性，我们可以指定 `model = Post`并且Django将会构建一个通过的*Post.objects.all()* QuerySet给我们。
* 使用环境变量*posts*给查询结果。如果我们不指定任意的*context_object_name*默认的变量将会是*object_list*。
* 对结果进行分页处理每页只显示3个对象。
* 使用自定义的template来渲染页面。如果我们不设置默认的template，*ListView*将会使用`blog/post_list.html`。

现在，打开你的博客应用下的urls.py文件，在*post_list* URL pattern之前添加一个新的URL pattern使用*PostlistView*类，如下所示：
    
    urlpatterns = [       # post views    # url(r'^$', views.post_list, name='post_list'),    url(r'^$', views.PostListView.as_view(),name='post_list'),
    url(r'^(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/'\           r'(?P<post>[-\w]+)/$',           views.post_detail,           name='post_detail'),    ]

为了保持分页处理能工作，我们必须将正确的页面对象传递给tempalte。Django的*ListView*通过叫做*page_obj*的变量来传递被选择的页面，所以你必须编辑你的*post_list_html* template因此去包含使用了正确的变量的分页处理，如下所示：
    
    {% include "pagination.html" with page=page_obj %}
    
在你的浏览器中打开 http://127.0.0.1:8000/blog/ 然后检查每一样事情是否都和之前的*post_list* view一样工作。这是一个简单的基于类的view例子通过使用Django提供的通用类。你将会学到更多的基于类的views在*第十章，创建一个在线学习平台*和相关联的章节。
#总结
在本章中，你学习了Django web框架的基础通过创建一个基础的博客应用。你为你的项目设计了数据models并且进行了数据迁移。你为你的博客创建了views，templates以及URLs，还包括对象分页。

在下一章中，你会学习到如何增强你的博客应用，例如评论系统，tag功能，并且允许你的用户通过邮件来分享帖子。

##译者总结
终于将第一章勉强翻译完成了，很多翻译的句子我自己都读不懂 - -|||
大家看到有错误有歧义的地方请帮忙指出，之后还会随时进行修改保证基本能读懂。
按照第一章的翻译速度，全书都翻译下来估计要2，3个月，这是非常非常乐观的估计，每天只有中午休息和下班后大概有两三小时的翻译时间。



