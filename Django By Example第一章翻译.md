书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**2016年12月10日发布（没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月7日精校完成（断断续续的终于完成了第一章精校，感觉比直接翻译还要累，继续加油）**

（译者注：本人目前在杭州某家互联网公司工作，岗位是测试研发，非常喜欢python，目前已经使用Django为公司内部搭建了几个自动化平台，因为没人教没人带，基本靠野路子自学，走过好多弯路，磕磕碰碰一路过来，前段时间偶尔看到《Django By Example》这本书，瞬间泪流满面，当初怎么没有找到这么好的Django教程。在看书的过程中不知道怎么搞的突然产生了翻译全书的想法，正好网上找了下也没有汉化的版本，所以准备踏上这条不归路。鉴于本人英文水平极低（四级都没过），单纯靠着有道词典和自己对上下文的理解以及对书中每行代码都保证敲一遍并运行的情况下，请各位在读到语句不通的时候或看不懂的地方请告诉我，我会及时进行改正。翻译全书，主要也是为了培养自己的英文阅读水平（口语就算了），谁叫好多最新最有用的计算机文档都是用英文写的，另外也可以培养自己的耐心，还可以分享给其他人，就这样。）

# 第一章
## 创建一个blog应用

在这本书中，你将学习如何创建完整的Django项目，可以在生产环境中使用。假如你还没有安装Django，在本章的第一部分你将学习如何安装。本章会覆盖如何使用Django去创建一个简单的blog应用。本章的目的是使你对该框架的工作有个基本概念，了解不同的组件之间是如何产生交互，并且教你一些技能通过使用一些基本功能方便的创建Djang项目。你会被引导创建一个完整的项目但是不会对所有的细节都进行详细说明。不同的框架组件将在本书接下来的章节中进行介绍。
本章会覆盖以下几点：

* 安装Django并创建你的第一个项目
* 设计模型（models）并且生成模型（model）数据库迁移
* 给你的模型（models）创建一个管理站点
* 使用查询集（QuerySet）和管理器（managers）
* 创建视图（views），模板（templates）和URLs
* 给列表视图（views）添加页码
* 使用Django内置的视图（views）

## 安装Django

如果你已经安装好了Django，你可以直接略过这部分跳到*创建你的第一个项目*。Django是一个Python包因此可以安装在任何的python的环境中。如果你还没有安装Django，这里有一个快速的指南帮助你安装Django用来本地开发。

Django需要在Python2.7或者3版本上才能更好的工作。在本书的例子中，我们将使用Python 3。如果你使用Linux或者Max OSX，你可能已经有安装好的Python。如果你不确定你的计算机中是否安装了Python，你可以在终端中输入 *python* 来确定。如果你看到以下类似的提示，说明你的计算机中已经安装好了Python:

```shell
Python 3.5.0 (v3.5.0:374f501f4567, Sep 12 2015, 11:00:19)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

如果你计算机中安装的Python版本低于3，或者没有安装，下载并安装Python 3.5.0 从http://www.python.org/download/ **（译者注：最新已经是3.6.0了，Django2.0将不再支持pytyon2.7，所以大家都从3版本以上开始学习吧）**。

由于你使用的是Python3，所以你没必要再安装一个数据库。这个Python版本自带SQLite数据库。SQLLite是一个轻量级的数据库，你可以在Django中进行使用用来开发。如果你准备在生产环境中部署你的应用，你应该使用一个更高级的数据库，比如PostgreSQL，MySQL或Oracle。你能获取到更多的信息关于数据库和Django的集成通过访问 https://docs.djangoproject.com/en/1.8/topics/install/#database-installation 。

## 创建一个独立的Python环境

强烈建议你使用virtualenv来创建独立的Python环境，这样你可以使用不同的包版本对应不同的项目，这比直接在真实系统中安装Python包更加的实用。另一个高级之处在于当你使用virtualenv你不需要任何管理员权限来安装Python包。在终端中运行以下命令来安装virtualenv：

    pip install virtualenv

**（译者注：如果你本地有多个python版本，注意Python3的pip命令可能是pip3）**

当你安装好virtualenv之后，通过以下命令来创建一个独立的环境：

    virtualenv my_env

以上命令会创建一个包含你的Python环境的my_env/目录。当你的virtualenv被激活的时候所有已经安装的Python库都会带入 my_env/lib/python3.5/site-packages 目录中。
如果你的系统自带Python2.X然后你又安装了Python3.X，你必须告诉virtualenv使用后者Python3.X。通过以下命令你可以定位Python3的安装路径然后使用该安装路径来创建virtualenv：

```shell
zenx\$ *which python3* 
/Library/Frameworks/Python.framework/Versions/3.5/bin/python3
zenx\$ *virtualenv my_env -p 
/Library/Frameworks/Python.framework/Versions/3.5/bin/python3*
```

通过以下命令来激活你的virtualenv：

    source my_env/bin/activate

shell提示将会附上激活的virtualenv名，被包含在括号中，如下所示：

    (my_evn)laptop:~ zenx$

你可以使用*deactivate*命令随时停用你的virtualenv。

你可以获取更多的信息关于virtualenv通过访问 https://virtualenv.pypa.io/en/latest/ 。

在virtualenv之上，你可以使用virtualenvwrapper工具。这个工具提供一些封装用来方便的创建和管理你的虚拟环境。你可以在 http://virtualenvwrapper.readthedocs.org/en/latest/ 下载该工具。

## 使用pip安装Django
**（译者注：请注意以下的操作都在激活的虚拟环境中使用）**

pip是安装Django的第一选择。Python3.5自带预安装的pip，你可以找到pip的安装指令通过访问 https://pip.pypa.io/en/stable/installing/ 。运行以下命令通过pip安装Django：

    pip install Django==1.8.6

Django将会被安装在你的虚拟环境的Python的*site-packages/*目录下。

现在检查Django是否成功安装。在终端中运行*python*并且导入Django来检查它的版本：

```shell
>>> import django
>>> django.VERSION
DjangoVERSION（1, 8, 5, 'final', 0)
```

如果你获得了以上输出，Django已经成功安装在你的机器中。

Django也可以使用其他方式来安装。你可以找到更多的信息通过访问 https://docs.djangoproject.com/en/1.8/topics/install/ 。

## 创建你的第一个项目

我们的第一个项目将会是一个完整的blog站点。Django提供了一个命令允许你方便的创建一个初始化的项目文件结构。在终端中运行以下命令：

    django-admin startproject mysite

该命令将会创建一个名为*mysite*的项目。
让我们来看下生成的项目结构：

```
mysite/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        wsgi.py
```

让我们来了解一下这些文件：

* *manage.py*:一个实用的命令行，用来与你的项目进行交互。它是一个对*django-admin.py*工具的简单封装。你不需要编辑这个文件。
* *mysite/*:你的项目目录，由以下的文件组成：
    * __init__.py:一个空文件用来告诉Python这个*mysite*目录是一个Python模块。
    * settings.py:你的项目的设置和配置。里面包含一些初始化的设置。
    * urls.py:你的URL模式存放的地方。这里定义的每一个URL都映射一个视图（view）。
    * wsgi.py:配置你的项目运行如同一个WSGI应用。

默认生成的*settings.py*文件包含一个使用一个SQLite数据库的基础配置以及一个Django应用列表，这些应用会默认添加到你的项目中。我们需要为这些初始应用在数据库中创建表。

打开终端执行以下命令：

```shell
cd mysite
python manage.py migrate
```

你将会看到以下的类似输出：

```shell
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
```

这些初始应用表将会在数据库中创建。过一会儿你就会学习到一些关于*migrate*的管理命令。

## 运行开发服务器

Django自带一个轻量级的web服务器来快速运行你的代码，不需要花费额外的时间来配置一个生产服务器。当你运行Django的开发服务器，它会一直检查你的代码变化。当代码有改变，它会自动重启，将你从手动重启中解放出来。但是，它可能无法注意到一些操作，例如在项目中添加了一个新文件，所以你在某些场景下还是需要手动重启。

打开终端，在你的项目主目录下运行以下代码来开启开发服务器：

    python manage.py runserver

你会看到以下类似的输出：

```shell
Performing system checks...
    
System check identified no issues (0 silenced).
November 5, 2015 - 19:10:54
Django version 1.8.6, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

现在，在浏览器中打开 http://127.0.0.1:8000/ ，你会看到一个告诉你项目成功运行的页面，如下图所示：
![django-1-1](http://upload-images.jianshu.io/upload_images/3966530-e5ab238aa9acbef9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你可以指定Django在定制的host和端口上运行开发服务，或者告诉它你想要运行你的项目通过读取一个不同的配置文件。例如：你可以运行以下 manage.py命令：

```shell
python manage.py runserver 127.0.0.1:8001 \
--settings=mysite.settings
```

这个命令迟早会对处理需要不同设置的多套环境启到作用。记住，这个服务器只是单纯用来开发，不适合在生产环境中使用。为了在生产环境中部署Django，你需要使用真实的web服务让它运行成一个WSGI应用例如Apache，Gunicorn或者uWSGI**（译者注：强烈推荐 nginx+uwsgi+Django）**。你能够获取到更多的信息关于如何在不同的web服务中部署Django，访问 https://docs.djangoproject.com/en/1.8/howto/deployment/wsgi/ 。

本书外额外的需要下载的章节*第十三章,Going Live*包含为你的Django项目设置一个生产环境。

##项目设置

让我们打开settings.py文件来看看你的项目的配置。在该文件中有许多设置是Django内置的，但这些只是所有Django可用配置的一部分。你可以看到所有的设置和它们默认的值通过访问 https://docs.djangoproject.com/en/1.8/ref/settings/ 。

以下列出的设置非常值得一看：

* DEBUG 一个布尔型用来开启或关闭项目的debug模式。如果设置为True，Django将会显示一个详细的错误页面当你的应用抛出一个未被捕获的异常。当你准备部署项目到生产环境，请记住一定要关闭debug模式。永远不要在生产环境中部署一个打开debug模式的站点因为那会暴露你的项目中的敏感数据。
* ALLOWED_HOSTS 当debug模式开启或者运行测试的时候不会起作用**（译者注：最新的Django版本中，不管有没有开启debug模式该设置都会启作用）**。一旦你准备部署你的项目到生产环境并且关闭了debug模式，你就必须添加你的域或host在这个设置中为了允许你的域或host访问你的Django项目。
* INSTALLED_APPS 这个设置你在所有的项目中都需要编辑。这个设置告诉Django有哪些应用会激活在这个项目中。默认的，Django包含以下应用：
  * django.contrib.admin：这是一个管理站点。
  * django.contrib.auth：这是一个权限框架。
  * django.contrib.contenttypes：这是一个内容类型的框架。
  * django.contrib.sessions：这是一个会话（session）框架。
  * django.contrib.messages：这是一个消息框架。
  * django.contrib.staticfiles：这是一个用来管理静态文件的框架
* MIDDLEWARE_CLASSES 是一个包含可执行中间件的元组。
* ROOT_URLCONF 指明你的应用定义的主URL模式存放在哪个Python模块中。
* DATABASES 是一个包含了所有在项目中使用的数据库的设置的字典。里面一定有一个默认的数据库。默认的配置使用的是SQLite3数据库。
* LANGUAGE_CODE 定义Django站点的默认语言编码。

不要担心你目前还看不懂这些设置的含义。你将会熟悉这些设置在之后的章节中。

##项目和应用
贯穿全书，你会反复的读到项目和应用的地位。在Django中，一个项目被认为是一个安装了一些设置的Django；一个应用是一个包含模型（models），视图（views），模板（templates）以及URLs的组合。应用之间的交互通过Django框架提供的一些特定功能，并且应用可能被各种各样的项目重复使用。你可以认为项目就是你的网站，这个网站包含多续的应用，例如blog，wiki或者论坛，这些应用都可以被其他的项目使用。**（译者注：我去，我竟然漏翻了这一节- -|||，罪过罪过，阿米头发）**

##创建一个应用

现在让我们创建你的第一个Django应用。我们将要创建一个勉强凑合的blog应用。在你的项目主目录下，运行以下命令：

    python manage.py startapp blog

这个命令会创建blog应用的基本目录结构，如下所示：

```shell
blog/
    __init__.py
    admin.py
    migrations/
    	__init__.py
    models.py
    tests.py
    views.py
```

这些文件的含义：

* admin.py: 在这儿你可以注册你的模型（models）并将它们包含到Django的管理页面中。使用Django的管理页面是可选的。
* migrations: 这个目录将会包含你的应用的数据库迁移。Migrations允许Django跟踪你的模型（model）变化并因此来同步数据库。
* models.py: 你的应用的数据模型（models）。所有的Django应用都需要拥有一个*models.py*文件，但是这个文件可以是空的。
* tests.py：在这儿你可以为你的应用创建测试。
* views.py：你的应用逻辑将会放在这儿。每一个视图（view）都会接受一个HTTP请求，处理该请求，最后返回一个响应。

##设计blog数据架构

我们将要开始为你的blog设计初始的数据模型（models）。一个模型（model）就是一个Python类，该类继承了*django.db.models.model*,在其中的每一个属性表示一个数据库字段。Djnago将会为*models.py*中的每一个定义的模型（model）创建一张表。当你创建好一个模型（model），Django会提供一个非常实用的API来方便的查询数据库。

首先，我们定义一个*POST*模型（model）。在blog应用下的*models.py*文件中添加以下内容：

```python
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
```

这就是我们给blog帖子使用的基础模型（model）。让我们来看下刚才在这个模型（model）中定义的各个字段含义：

* title： 这个字段对应帖子的标题。它是*CharField*，在SQL数据库中会被转化成VARCHAR。
* slug：这个字段将会在URLs中使用。slug就是一个短标签，该标签只包含字母，数字，下划线或连接线。我们将通过使用slug字段给我们的blog帖子构建漂亮的，友好的URLs。我们给该字段添加了*unique_for_date*参数，这样我们就可以使用日期和帖子的slug来为所有帖子构建URLs。我们可以给每个帖子使用日期和slug来构建URLs通过添加 unique_for_date参数给这个字段。在相同的日期中Django会阻止多篇帖子拥有相同的slug。
* author：这是一个*ForeignKey*。这个字段定义了一个多对一（many-to-one）的关系。我们告诉Django一篇帖子只能由一名用户编写，一名用户能编写多篇帖子。根据这个字段，Django将会在数据库中通过有关联的模型（model）主键来创建一个外键。在这个场景中，我们关联上了Django权限系统的*User*模型（model）。我们通过*related_name*属性指定了从*User*到*Post*的反向关系名。我们将会在之后学习到更多关于这方面的内容。
* body：这是帖子的主体。它是*TextField*，在SQL数据库中被转化成*TEXT*。
* publish：这个日期表明帖子什么时间发布。我们使用Djnago的*timezone*的*now*方法来设定默认值。This is just a timezone-aware datetime.now**（译者注：这句该咋翻译好呢）**。
* created：这个日期表明帖子什么时间创建。因为我们在这儿使用了*auto_now_add*，当一个对象被创建的时候这个字段会自动保存当前日期。
* updated：这个日期表明帖子什么时候更新。因为我们在这儿使用了*auto_now*，当我们更新保存一个对象的时候这个字段将会自动更新到当前日期。
* status：这个字段表示当前帖子的展示状态。我们使用了一个*choices*参数，这样这个字段的值只能是给予的选择参数中的某一个值。**（译者注：传入元组，比如`(1,2)`，那么该字段只能选择1或者2，没有其他值可以选择）**

就像你所看到的的，Django内置了许多不同的字段类型给你使用，这样你就能够定义你自己的模型（models）。你可以找到所有的字段类型通过访问 https://docs.djangoproject.com/en/1.8/ref/models/fields/

在模型（model）中的类*Meta*包含元数据。我们告诉Django查询数据库的时候默认返回的是根据*publish*字段进行降序排列过的结果。我们使用负号来指定进行降序排列。

*__str__()*方法是当前对象默认的可读表现。Django将会在很多地方用到它例如管理站点中。

> 如果你之前使用过Python2.X，请注意在Python3中所有的strings都使用unicode，因此我们只使用*__str__()*方法。*__unicode__()*方法已经废弃。**（译者注：Python3大法好，Python2别再学了，直接学Python3吧）**

在我们处理日期之前，我们需要下载*pytz*模块。这个模块给Python提供时区的定义并且SQLite也需要它来对日期进行操作。在终端中输入以下命令来安装*pytz*：

    pip install pytz

Django内置对时区日期处理的支持。你可以在你的项目中的*settings.py*文件中通过*USE_TZ*来设置激活或停用对时区的支持。当你通过*startproject*命令来创建一个新项目的时候这个设置默认为*True*。

##激活你的应用
为了让Django能保持跟踪你的应用并且根据你的应用中的模型（models）来创建数据库表，我们必须激活你的应用。因此，编辑*settings.py*文件在*INSTALLED_APPS*设置中添加*blog*。看上去如下所示：
```python
INSTALLED_APPS = ( 
    'django.contrib.admin',    
    'django.contrib.auth', 
    'django.contrib.contenttypes', 
    'django.contrib.sessions', 
    'django.contrib.messages', 
    'django.contrib.staticfiles',
    'blog',
 )
```
**（译者注：该设置中应用的排列顺序也会对项目的某些方面产生影响，具体情况后几章会有介绍，这里提醒下）**

现在Django已经知道在项目中的我们的应用是激活状态并且将会对其中的模型（models）进行自审。

##创建和进行数据库迁移
让我们为我们的模型（model）在数据库中创建一张数据表格。Django自带一个数据库迁移（migration）系统来跟踪你对模型（models）的修改，然后会同步到数据库。*migrate*命令会应用到所有在*INSTALLED_APPS*中的应用，它会根据当前的模型（models）和数据库迁移（migrations）来同步数据库。

首先，我们需要为我们刚才创建的新模型（model）创建一个数据库迁移（migration）。在你的项目主目录下，执行以下命令：

    python manage.py makemigrations blog

你会看到以下输出:
```shell
Migrations for 'blog':
    0001_initial.py;
        - Create model Post
```

Django刚才在blog应用下的migrations目录中创建了一个*0001——initial.py*文件。你可以打开这个文件来看下一个数据库迁移的内容。

让我们来看下Django根据我们的模型（model）将会为在数据库中创建的表格而执行的SQL代码。*sqlmigrate*命令带上数据库迁移（migration）的名字将会返回它们的SQL，但不会立即去执行。运行以下命令来看下输出：

    python manage.py sqlmigrate blog 0001
    
输出类似如下：
```shell
BEGIN;
CREATE TABLE "blog_post" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "title" varchar(250) NOT NULL, "slug" varchar(250) NOT NULL, "body" text NOT NULL, "publish" datetime NOT NULL, "created" datetime NOT NULL, "updated" datetime NOT NULL, "status" varchar(10) NOT NULL, "author_id" integer NOT NULL REFERENCES "auth_user" ("id"));
CREATE INDEX "blog_post_2dbcba41" ON "blog_post" ("slug");
CREATE INDEX "blog_post_4f331e2f" ON "blog_post" ("author_id");
COMMIT;
```    

Django会根据你正在使用的数据库进行以上精准的输出。以上SQL语句是为SQLite数据库准备的。如你所见，Django生成的表名前缀为应用名之后跟上模型（modle）的小写（blog_post），但是你也可以指定表名通过在模型（models）的*Meta*类中使用*db_table*属性进行指定。Django会自动为每个模型（model）创建一个主键，但是你也可以自己指定主键通过设置*primarry_key=True*在模型（model）中的某个字段上。

让我们根据新模型（model）来同步数据库。运行以下的命令来应用存在的数据迁移（migrations）：

    python manage.py migrate
    
你应该会看到以下行跟在输出的末尾：

    Applying blog.0001_initial... OK
    
我们刚刚为*INSTALLED_APPS*中所有的应用进行了数据库迁移（migrations），包括我们的*blog*应用。在进行了数据库迁移（migrations）之后，数据库会映射我们模型的当前状态。

如果你编辑了*models.py*文件为了添加，删除，还是改变了存在的模型（models）中字段，或者你添加了新的模型（models），你都需要做一次新的数据库迁移（migration）通过使用*makemigrations*命令。数据库迁移（migration）允许Django来保持对模型（model）改变的跟踪。之后你必须通过*migrate*命令来保持数据库与我们的模型（models）同步。

##为你的模型（models）创建一个管理站点
现在我们已经定义好了*Post*模型（model），我们将要创建一个简单的管理站点来管理blog帖子。Django内置了一个管理接口，该接口对编辑内容非常的有用。这个Django管理站点会根据你的模型（model）元数据进行动态构建并且提供一个可读的接口来编辑内容。你可以对这个站点进行自由的定制，配置你的模型（models）在其中如何进行显示。

请记住，*django.contrib.admin*已经被包含在我们项目的*INSTALLED_APPS*设置中，我们不需要再额外添加。

##创建一个超级用户
首先，我们需要创建一名用户来管理这个管理站点。运行以下的命令：
    
    python manage.py createsuperuser
    
你会看下以下输出。输入你想要的用户名，邮箱和密码：

```shell
Username (leave blank to use 'admin'): admin
Email address: admin@admin.com
Password: ********
Password (again): ********
Superuser created successfully.
```

##Django管理站点
现在，通过`python manage.py runserver`命令来启动开发服务器，之后在浏览器中打开 http://127.0.0.1:8000/admin/ 。你会看到管理站点的登录页面，如下所示：
![django-1-2](http://upload-images.jianshu.io/upload_images/3966530-e2479f1cb20b1d5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用你在上一步中创建的超级用户信息进行登录。你将会看到管理站点的首页，如下所示：
![django-1-3](http://upload-images.jianshu.io/upload_images/3966530-80c83f3a385f13c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*Group*和*User* 模型（models） 是Django权限管理框架的一部分位于*django.contrib.auth*。如果你点击*Users*，你将会看到你之前创建的用户信息。你的blog应用的*Post*模型（model）和*User*（model）关联在了一起。记住，它们是通过*author*字段进行关联的。

##在管理站点中添加你的模型（models）
让我们在管理站点中添加你的blog模型（models）。编辑blog应用下的*admin.py*文件，如下所示：

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```
    
现在，在浏览器中刷新管理站点。你会看到你的*Post*模型（model）已经在页面中展示，如下所示：
![django-1-4](http://upload-images.jianshu.io/upload_images/3966530-52e408aa45e35c1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这很简单，对吧？当你在Django的管理页面注册了一个模型（model），Django会通过对你的模型（models）进行内省然后提供给你一个非常友好有用的接口，这个接口允许你非常方便的排列，编辑，创建，以及删除对象。

点击*Posts*右侧的*Add*链接来添加一篇新帖子。你将会看到Django根据你的模型（model）动态生成了一个表单，如下所示：
![django-1-5](http://upload-images.jianshu.io/upload_images/3966530-392e41ac7cb34cda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Django给不同类型的字段使用了不同的表单控件。即使是复杂的字段例如*DateTimeField*也被展示成一个简单的接口类似一个JavaScript日期选择器。

填写好表单然后点击*Save*按钮。你会被重定向到帖子列页面并且得到一条帖子成功创建的提示，如下所示：
![django-1-6](http://upload-images.jianshu.io/upload_images/3966530-879a2158c1c80ae9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##定制models的展示形式
现在我们来看下如何定制管理站点。编辑blog应用下的*admin.py*文件，使之如下所示：

 ```python   
from django.contrib import admin
from .models import Post

class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish',
                    'status')
admin.site.register(Post, PostAdmin)
```
    
我们使用继承了*ModelAdmin*的定制类来告诉Django管理站点中需要注册我们自己的模型（model）。在这个类中，我们可以包含一些信息关于如何在管理站点中展示模型（model）以及如何与该模型（model）进行交互。*list_display*属性允许你在设置一些你想要在管理对象列表页面显示的模型（model）字段。

让我们来通过更多的选项来定制管理模型（model），如使用以下代码：

```python
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'author', 'publish',
                    'status')
    list_filter = ('status', 'created', 'publish', 'author')
    search_fields = ('title', 'body')
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ('author',)
    date_hierarchy = 'publish'
    ordering = ['status', 'publish']
```

回到浏览器刷新管理站点页面，现在应该如下所示：
![django-1-7](http://upload-images.jianshu.io/upload_images/3966530-3b8a79f28e1a04de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你可以看到帖子列页面中展示的字段都是你在*list-dispaly*属性中指定的。这个列页面现在包含了一个右侧边栏允许你根据*list_filter*属性中指定的字段来过滤返回结果。一个搜索框也在应用在页面中。这是因为我们还定义了一个搜索字段列通过使用*search_fields*属性。在搜索框的下方，有个可以通过时间层快速导航的栏，该栏通过定义*date_hierarchy*属性出现。你还能看到这些帖子默认的通过*Status*和*Publish*列进行排序。这是因为你指定了默认排序通过使用*ordering*属性。

现在，点击*Add post*链接。你还会在这儿看到一些改变。当你输入完成新帖子的标题，*slug*字段将会自动填充。我们通过使用*prepoupulated_fields*属性告诉Django通过输入的标题来填充*slug*字段。同时，如今的*author*字段展示显示为了一个搜索控件，这样当你的用户量达到成千上万级别的时候比再使用下拉框进行选择更加的人性化，如下图所示：
![django-1-8](http://upload-images.jianshu.io/upload_images/3966530-f89c73f0b51aba4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过短短的几行代码，我们就在管理站点中自定义了我们的模型（model）的展示形式。还有更多的方式可以用来定制Django的管理站点。在这本书的后面，我们还会进一步讲述。

##使用查询集（QuerySet）和管理器（managers）
现在，你已经有了一个个完整功能的管理站点来管理你的blog内容，是时候学习如何从数据库中检索信息并且与这些信息进行交互了。Django自带了一个强大的数据库抽象API可以让你轻松的创建，检索，更新以及删除对象。Django的*Object-relational Mapper(ORM)*可以兼容MySQL,PostgreSQL,SQLite以及Oracle。请记住你可以在你项目下的*setting.py*中编辑*DATABASES*设置来指定数据库。Django可以同时与多个数据库进行工作，这样你可以编写数据库路由来操作数据通过任何你喜欢的方式。

一旦你创建好了你的数据模型（models），Django会提供你一个API来与它们进行交互。你可以找到数据模型（model）的官方参考文档通过访问 https://docs.djangoproject.com/en/1.8/ref/models/ 。

##创建对象
打开终端运行以下命令来打开Python shell：

    python manage.py shell
    
然后依次输入以下内容：
```shell
>>> from django.contrib.auth.models import User
>>> from blog.models import Post
>>> user = User.objects.get(username='admin')
>>> post = Post.objects.create(title='One more post',
                        slug='one-more-post',
                        body='Post body.',
                        author=user)
>>> post.save()
```

让我们来研究下这些代码做了什么。首先，我们取回了一个username是*admin*的用户对象：
    
    user = User.objects.get(username='admin')

`get()`方法允许你从数据库取回一个单独的对象。注意这个方法只希望有唯一的一个匹配在查询中。如果在数据库中没有返回结果，这个方法会抛出一个*DoesNotExist*异常，如果数据库返回多个匹配结果，将会抛出一个*MultipleObjectsReturned*异常。当查询执行的时候，所有的异常都是模型（model）类的属性。

接着，我们来创建一个拥有定制标题标题，slug和内容的*Post*实例，然后我们设置之前取回的user胃这篇帖子的作者如下所示：
    
    post = Post(title='Another post', slug='another-post', body='Postbody.', author=user)

> 这个对象只是存在内存中不会执行到数据库中

最后，我们通过使用*save()*方法来保存该对象到数据库中：

    post.save()
    
这步操作将会执行一段SQL的插入语句。我们已经知道如何在内存中创建一个对象并且之后才在数据库中进行插入，但是我们也可以直接在数据库中创建对象通过使用*create()*方法，如下所示：

    Post.objects.create(title='One more post', slug='one-more-post',body='Post body.', author=user)
    
##更新对象
现在，改变这篇帖子的标题并且再次保存对象：

```shell
>>> post.title = 'New title'
>>> post.save()
```

这一次，*save()*方法执行了一条更新语句。

> 你对对象的改变一直存在内存中直到你执行到*save()*方法。

##取回对象
Django的*Object-relational mapping(ORM)*是基于查询集（QuerySet）。查询集（QuerySet）是从你的数据库中根据一些过滤条件范围取回的结果对象进行的采集。你已经知道如何通过*get()*方法从数据库中取回单独的对象。如你所见：我们通过`Post.objects.get()`来使用这个方法。每一个Django模型（model）至少有一个管理器（manager），默认管理器（manager）叫做*objects*。你能获得一个查询集（QuerySet）对象就是通过使用你的模型（models）的管理器（manager）。获取一张表中的所有对象，你只需要在默认的*objects*管理器（manager）上使用*all()*方法即可，如下所示：

    >>> all_posts = Post.objects.all()
    
这就是我们如何创建一个返回数据库中所有对象的查询集（QuerySet）。注意这个查询集（QuerySet）并还没有执行。Django的查询集（QuerySets）是惰性（lazy）的，它们只会被动的去执行。这样的行为可以保证查询集（QuerySet）非常效率。如果我们没有把查询集（QuerySet）设置给一个变量，而是直接在Python shell中编写，这样查询集（QuerySet）的SQL语句将立马执行因为我们迫使它输出结果：
    
    >>> Post.objects.all()
    
##使用filter()方法
为了过滤查询集（QuerySet），你可以在管理器（manager）上使用`filter()`方法。例如，我们可以返回所有在2015年发布的帖子，如下所示：
    
    Post.objects.filter(publish__year=2015)
    
你也可以使用多个字段来进行过滤。例如，我们可以返回2015年发布的所有作者用户名为*admin*的帖子，如下所示：

    Post.objects.filter(publish__year=2015, author__username='admin')
    
上面的写法和下面的写法产生的结果是一致的：

    Post.objects.filter(publish__year=2015).filter(author__username='admin')
    
> 我们构建了字段的查找方法查询通过使用两个下划线`(publish__year)`，除此以外我们也可以访问关联的模型（model）字段通过使用两个下划线`(author__username)`。

##使用exclude()
你可以在管理器（manager）上使用*exclude()*方法来排除某些返回结果。例如：我们可以返回所有2015年发布的帖子但是这些帖子的题目开头不能是*Why*:

    Post.objects.filter(publish__year=2015).exclude(title__startswith='Why')
    
##使用order_by()
你可以对结果进行排序通过在管理器（manager）上使用*order_by()*方法来对不同的字段进行排序。例如：你可以取回所有对象通过它们的标题进行排序：

    Post.objects.order_by('title')
    
默认是升序。你可以通过负号来指定使用降序，如下所示：

    Post.objects.order_by('-title')

##删除对象
如果你想删除一个对象，你可以对对象实例进行下面的操作：

```python
post = Post.objects.get(id=1)
post.delete()
```
   
> 请注意，删除对象也将删除任何的依赖关系


##查询集（QuerySet）什么时候会执行
只要你喜欢，你可以连接许多的过滤给查询集（QuerySet）而且不会立马在数据库中执行直到这个查询集（QuerySet）被执行。查询集（QuerySet）只有在以下情况中才会执行：
    * 在你第一次迭代它们的时候
    * 当你对它们的实例进行切片：例如`Post.objects.all()[:3]`
    * 当你对它们进行了打包或缓存
    * 当你对它们调用了`repr()`或`len()`方法
    * 当你明确的对它们调用了`list()`方法
    * 当你在一个声明中测试它，例如*bool()*, or, and, or if

##创建model manager
我们之前提到过, *objects*是每一个模型（models）的默认管理器（manager），它会返回数据库中所有的对象。但是我们也可以为我们的模型（models）定义一些定制的管理器（manager）。我们准备创建一个定制的管理器（manager）来返回所有状态为已发布的帖子。

有两种方式可以为你的模型（models）添加管理器（managers）：你可以添加额外的管理器（manager）方法或者继承管理器（manager）的查询集（QuerySets）进行修改。第一种方法类似`Post.objects.my_manager()`,第二种方法类似`Post.my_manager.all()`。我们的管理器（manager）将会允许我们返回所有帖子通过使用`Post.published`。

编辑你的blog应用下的*models.py*文件添加如下代码来创建一个管理器（manager）:

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super(PublishedManager,
                    self).get_queryset().filter(status='published')
    
class Post(models.Model):
    # ...
    objects = models.Manager() # The default manager.
    published = PublishedManager() # Our custom manager.
```

`get_queryset()`是返回执行过的查询集（QuerySet）的方法。我们通过使用它来包含我们定制的过滤到完整的查询集（QuerySet）中。我们定义我们定制的管理器（manager）然后添加它到*Post* 模型（model）中。我们现在可以来执行它。例如，我们可以返回所有标题开头为*Who*的并且是已经发布的帖子:
    
    Post.published.filter(title__startswith='Who')
    
##构建列和详情视图（views）
现在你已经学会了一些如何使用ORM的基本知识，你已经准备好为blog应用创建视图（views）了。一个Django视图（view）就是一个Python方法，它可以接收一个web请求然后返回一个web响应。在视图（views）中通过所有的逻辑处理返回期望的响应。

首先我们会创建我们的应用视图（views），然后我们将会为每个视图（view）定义一个URL模式，我们将会创建HTML模板（templates）来渲染这些视图（views）生成的数据。每一个视图（view）都会渲染模板（template）传递变量给它然后会返回一个经过渲染输出的HTTP响应。

##创建列和详情views
让我们开始创建一个视图（view）来展示帖子列。编辑你的blog应用下中*views.py*文件，如下所示：

```python
from django.shortcuts import render, get_object_or_404
from .models import Post
def post_list(request):
    posts = Post.published.all()
    return render(request,
                  'blog/post/list.html',
                  {'posts': posts})
```

你刚创建了你的第一个Django视图（view）。*post_list*视图（view）将*request*对象作为唯一的参数。记住所有的的视图（views）都有需要这个参数。在这个视图（view）中，我们获取到了所有状态为已发布的帖子通过使用我们之前创建的*published*管理器（manager）。

最后，我们使用Django提供的快捷方法*render()*来渲染帖子列通过给予的模板（template）。这个函数将*request*对象作为参数，模板（template）路径以及变量来渲染的给予的模板（template）。它返回一个渲染文本（一般是HTML代码）*HttpResponse*对象。*render()*方法考虑到了请求内容，这样任何模板（template）内容处理器设置的变量都可以带入给予的模板（template）中。你会在*第三章，扩展你的blog应用*学习到如何使用它们。

让我们创建第二个视图（view）来展示一篇单独的帖子。添加如下代码到*views.py*文件中：
    
```python
def post_detail(request, year, month, day, post):
    post = get_object_or_404(Post, slug=post,
                                   status='published',
                                   publish__year=year,
                                   publish__month=month,
                                   publish__day=day)
    return render(request,
                  'blog/post/detail.html',
                  {'post': post})
```
                    
这是一个帖子详情视图（view）。这个视图（view）使用*year，month，day*以及*post*作为参数来获取到一篇已经发布的帖子通过给予slug和日期。请注意，当我们创建*Post*模型（model）的时候，我们给slgu字段添加了*unique_for_date*参数。这样我们可以确保在给予的日期中只有一个帖子会带有一个slug，因此，我们能取回单独的帖子通过日期和slug。在这个详情视图（view）中，我们通过使用*get_object_or_404()*快捷方法来检索期望的*Post*。这个函数能取回匹配给予的参数的对象，或者返回一个HTTP 404（Not found）异常当没有匹配的对象。最后，我们使用*render()*快捷方法来使用一个模板（template）去渲染取回的帖子。

##为你的视图（views）添加URL模式

一个URL模式是由一个Python正则表达，一个视图（view），一个全项目范围内的命名组成。Django在运行中会遍历所有URL模式直到第一个匹配的请求URL才停止。之后，Django导入匹配的URL模式中的视图（view）并执行它，使用关键字或指定参数来执行一个*HttpRequest*类的实例。
如果你之前没有接触过正则表达式，你需要去稍微了解下，通过访问 https://docs.python.org/3/howto/regex.html 。

在blog应用目录下创建一个*urls.py*文件，输入以下代码：
```python        
from django.conf.urls import url
from . import views
urlpatterns = [
    # post views
    url(r'^$', views.post_list, name='post_list'),
    url(r'^(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/'\
        r'(?P<post>[-\w]+)/$',
        views.post_detail,
        name='post_detail'),
]
```

第一条URL模式没有带入任何参数，它映射到*post_list*视图（view）。第二条URL模式带上了以下4个参数映射到*post_detail*视图（view）中。让我们看下这个URL模式中的正则表达式：

* year：需要四位数
* month：需要两位数。不及两位数，开头带上0，比如 01，02
* day：需要两位数。不及两位数开头带上0
* post：可以由单词和连字符组成

> 为每一个应用创建单独的*urls.py*文件是最好的方法可以保证你的应用能给别的项目再度使用。

现在你需要将你blog中的URL模式包含到项目的主URL模式中。编辑你的项目中的*mysite*文件夹中*urls.py*文件，如下所示：
   
```python 
from django.conf.urls import include, url
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)), 
    url(r'^blog/', include('blog.urls',
        namespace='blog',
        app_name='blog')),
]
```
通过这样的方式，你告诉Django在*blog/*路径下包含了blog应用中的*urls.py*定义的URL模式。你可以给它们一个命名空间叫做*blog*这样你可以方便的引用这个URLs组。

##模型（models）的标准URLs

你可以使用之前定义的*post_detil* URL给*Post*对象构建标准URL。Django的惯例是添加*get_absolute_url()*方法给模型（model）用来返回一个对象的标准URL。在这个方法中，我们使用*reverse()*方法允许你通过它们的名字和可选的参数来构建URLS。编辑你的*models.py*文件添加如下代码：

```python
from django.core.urlresolvers import reverse
Class Post(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse('blog:post_detail',
                        args=[self.publish.year,
                              self.publish.strftime('%m'),
                              self.publish.strftime('%d'),
                              self.slug])
```

请注意，我们通过使用*strftime()*方法来保证个位数的月份和日期需要带上0来构建URL**（译者注：也就是01,02,03）**。我们将会在我们的模板（templates）中使用*get_absolute_url()*方法。

##为你的视图（views）创建模板（templates）

我们为我们的应用创建了视图（views）和URL模式。现在该添加模板（templates）来展示界面友好的帖子了。

在你的blog应用目录下创建以下目录结构和文件：

```
templates/
    blog/
        base.html
        post/
            list.html
            detail.html
```
            
以上就是我们的模板（templates）的文件目录结构。*base.html*文件将会包含站点主要的HTML结构以及分割内容区域和一个导航栏。*list.html*和*detail.html*文件会继承*base.html*文件来渲染各自的blog帖子列和详情视图（view）。

Django有一个强大的模板（templates）语言允许你指定数据的如何进行展示。它基于模板标签（templates tags）， 例如 `{% tag %}`, `{{ variable }}`以及模板过滤器（templates filters），可以对变量进行过滤，例如 `{{ variable|gilter }}`。你可以找到所有的内置模板标签（templates tags）和过滤器（filters）通过访问 https://docs.djangoproject.com/en/1.8/ ref/templates/builtins/ 。

让我们来编辑*base.html*文件添加如下代码：

```html
{% load staticfiles %}
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}{% endblock %}</title>
  <link href="{% static "css/blog.css" %}" rel="stylesheet">
</head>
<body>
  <div id="content">
    {% block content %}
    {% endblock %}
  </div>
  <div id="sidebar">
    <h2>My blog</h2>
      <p>This is my blog.</p>
  </div>
</body>
</html>
```   

`{% load staticfiles %}`告诉Django去加载*django.contrib.staticfiles*应用提供的*staticfiles* 模板标签（temaplate tags）。通过加载它，你可以在这个模板（template）中使用`{% static %}`模板过滤器（template filter）。通过使用这个模板过滤器（template filter），你可以包含一些静态文件比如说*blog.css*文件，你可以在本书的范例代码例子中找到该文件，在blog应用的*static/*目录中**（译者注：给大家个地址去拷贝 https://github.com/levelksk/django-by-example-book ）**拷贝这个目录到你的项目下的相同路径来使用这些静态文件。

你可以看到有两个`{% block %}`标签（tags）。这些是用来告诉Django我们想在这个区域中定义一个区块（block）。继承这个模板（template）的模板们（templates）可以使用自定义的内容来填充区块（block）。我们定义了一个区块（block）叫做*title*，另一个区块（block）叫做*content*。

让我们编辑*post/list.html*文件使它如下所示：

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
  <h1>My Blog</h1>
  {% for post in posts %}
    <h2>
      <a href="{{ post.get_absolute_url }}">
        {{ post.title }}
      </a>
    </h2>
    <p class="date">
      Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|truncatewords:30|linebreaks }}
  {% endfor %}
{% endblock %}
```    

通过`{% extends %}`模板标签（template tag），我们告诉Django需要继承*blog/base.html* 模板（template）。然后我们在*title*和*content*区块（blocks）中填充内容。我们通过循环迭代帖子来展示它们的标题，日期，作者和内容，在标题中还集成了帖子的标准URL链接。在帖子的内容中，我们应用了两个模板过滤器（template filters）： *truncatewords*用来缩短内容限制在一定的字数内，*linebreaks*用来转换内容中的换行符为HTML的换行符。只要你喜欢你可以连接许多模板标签（tempalte filters），每一个都会应用到上个输出生成的结果上。

打开终端执行命令`python manage.py runserver`来启动开发服务器。在浏览器中打开 http://127.0.0.1:8000/blog/ 你会看到运行结果。注意，你需要添加一些发布状态的帖子才能在这儿看到它们。你会看到如下图所示：
![django-1-9](http://upload-images.jianshu.io/upload_images/3966530-a8f162f31d72f0dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这之后，让我们来编辑*post/detail.html*文件使它如下所示：

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
  <h1>{{ post.title }}</h1>
  <p class="date">
    Published {{ post.publish }} by {{ post.author }}
  </p>
  {{ post.body|linebreaks }}
{% endblock %}
```
    
现在，你可以返回你的浏览器中点击其中一篇帖子的标题来看帖子的详细视图（view）。你会看到类似以下页面：
![django-1-10](http://upload-images.jianshu.io/upload_images/3966530-6c9f869e3aaad43d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##添加页码

当你开始给你的blog添加内容，你很快会意识到你需要将帖子分页显示。Django有一个内置的*Paginator*类允许你方便的管理分页。

编辑blog应用下的*views.py*文件导入Django的页码类修改*post_list*如下所示：

```python
from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger

def post_list(request):
    object_list = Post.published.all()
    paginator = Paginator(object_list, 3) # 3 posts in each page
    page = request.GET.get('page')
    try:
        posts = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer deliver the first page
        posts = paginator.page(1)
    except EmptyPage:
        # If page is out of range deliver last page of results
        posts = paginator.page(paginator.num_pages)
    return render(request,
                  'blog/post/list.html',
                  {'page': page, 
                   'posts': posts})
```

*Paginator*是如何工作的：

* 我们使用希望在每页中显示的对象的数量实例化了*Paginator*类。
* 我们获取到*page* GET参数来指明页数   
* 我们通过调用*Paginator*的 *page()*方法在期望的页面中获得了对象。
* 如果*page*参数不是一个整数，我们就返回第一页的结果。如果这个参数数字超出了最大的页数，我们就展示最后一页的结果。
* 我们传递页数并且获取对象给这个模板（template）。

现在，我们必须创建一个模板（template）来展示分页处理，它可以被任意的模板（template）包含来使用分页。在blog应用的*templates*文件夹下创建一个新文件命名为*pagination.html*。在该文件中添加如下HTML代码：

```html
<div class="pagination">
  <span class="step-links">
    {% if page.has_previous %}
      <a href="?page={{ page.previous_page_number }}">Previous</a>
    {% endif %}
    <span class="current">
      Page {{ page.number }} of {{ page.paginator.num_pages }}.
    </span>
      {% if page.has_next %}
        <a href="?page={{ page.next_page_number }}">Next</a>
      {% endif %}
  </span>
</div>    
```
    
这个分页模板（template）期望一个*Page*对象为了渲染上一页与下一页的链接并且展示当前页面和所有页面的结果。让我们回到*blog/post/list.html*模板（tempalte）中将*pagination.html*模板（template）包含在`{% content %}`区块（block）中，如下所示：

```html
{% block content %}
  ...
  {% include "pagination.html" with page=posts %}
{% endblock %}
```

我们传递给模板（template）的*Page*对象叫做*posts*，我们将分页模板（tempalte）包含在帖子列模板（template）中指定参数来对它进行正确的渲染。这种方法你可以反复使用，用你的分页模板（template）对不同的模型（models）视图（views）进行分页处理。

现在，在你的浏览器中打开 http://127.0.0.1:8000/blog/。 你会看到帖子列的底部已经有分页处理：
![django-1-11](http://upload-images.jianshu.io/upload_images/3966530-6fbd1a4b2e409533.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##使用基于类的视图（views）

因为一个视图（view）的调用就是得到一个web请求并且返回一个web响应，你可以将你的视图（views）定义成类方法。Django为此定义了基础的视图（view）类。它们都从*View*类继承而来，*View*类可以操控HTTP方法调度以及其他的功能。这是一个可替代的方法来创建你的视图（views）。

我们准备使我们的*post_list*视图（view）转变为一个基于类的视图通过使用Django提供的通用*ListView*。这个基础视图（view）允许你对任意的对象进行排列。

编辑你的blog应用下的*views.py*文件，如下所示：

```python
from django.views.generic import ListView
class PostListView(ListView):
    queryset = Post.published.all()
    context_object_name = 'posts'
    paginate_by = 3
    template_name = 'blog/post/list.html'
```        
        
这个基于类的的视图（view）类似与之前的*post_list*视图（view）。在这儿，我们告诉*ListView*做了以下操作：

* 使用一个特定的查询集（QuerySet）代替取回所有的对象。代替定义一个*queryset*属性，我们可以指定`model = Post`然后Django将会构建*Post.objects.all()* 查询集（QuerySet）给我们。
* 使用环境变量*posts*给查询结果。如果我们不指定任意的*context_object_name*默认的变量将会是*object_list*。
* 对结果进行分页处理每页只显示3个对象。
* 使用定制的模板（template）来渲染页面。如果我们不设置默认的模板（template），*ListView*将会使用`blog/post_list.html`。

现在，打开你的blog应用下的*urls.py*文件，注释到之前的*post_list*URL模式，在之后添加一个新的URL模式来使用*PostlistView*类，如下所示：
   
```python    
urlpatterns = [
    # post views
    # url(r'^$', views.post_list, name='post_list'),
    url(r'^$', views.PostListView.as_view(),name='post_list'),
    url(r'^(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/'\
        r'(?P<post>[-\w]+)/$',
        views.post_detail,
        name='post_detail'),
]
```

为了保持分页处理能工作，我们必须将正确的页面对象传递给模板（tempalte）。Django的*ListView*通过叫做*page_obj*的变量来传递被选择的页面，所以你必须编辑你的*post_list_html*模板（template）去包含使用了正确的变量的分页处理，如下所示：
    
    {% include "pagination.html" with page=page_obj %}
    
在你的浏览器中打开 http://127.0.0.1:8000/blog/ 然后检查每一样功能是否都和之前的*post_list*视图（view）一样工作。这是一个简单的基于类的视图（view）例子通过使用Django提供的通用类。你将会学到更多的基于类的视图（views）在*第十章，创建一个在线学习平台*以及相关的章节中。

#总结

在本章中，你通过创建一个基础的blog应用学习了Django web框架的基础。你为你的项目设计了数据模型（models）并且进行了数据库迁移。你为你的blog创建了视图（views），模板（templates）以及URLs，还包括对象分页。

在下一章中，你会学习到如何增强你的blog应用，例如评论系统，标签（tag）功能，并且允许你的用户通过邮件来分享帖子。

##译者总结

终于将第一章勉强翻译完成了，很多翻译的句子我自己都读不懂 - -|||
大家看到有错误有歧义的地方请帮忙指出，之后还会随时进行修改保证基本能读懂。
按照第一章的翻译速度，全书都翻译下来估计要2，3个月，这是非常非常乐观的估计，每天只有中午休息和下班后大概有两三小时的翻译时间。

**2016年12月10日发布（没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月7日精校完成（断断续续的终于完成了第一章精校，感觉比直接翻译还要累，继续加油）**



