书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

（译者注：翻译本章过程中几次想放弃，但是既然都到第十章了，怎么能放弃！）

#第十章

##创建一个在线学习平台（e-Learning Platform）

在上一章，你添加国际化到你的在线商店项目中。你还构建了一个优惠券系统和一个商品推荐引擎。在这章中，你会创建一个新项目。你将构建一个在线学习平台创建一个定制内容管理系统。

在这章中，你会学习以下操作：

* 创建fixtures给你的模型
* 使用模型继承
* 创建定制模型字段
* 使用基于类的视图和mixins
* 构建formsets
* 管理组合权限
* 创建一个内容管理系统

##创建一个在线学习平台

我们最实际的项目将会是一个在线学习平台。在本章中，我们将要构建一个灵活的**内容管理系统（CMS）**用来允许教师来创建课程和管理它们的内容。

首先，创建一个虚拟环境给你的新项目并且激活它通过以下命令：

```shell
mkdir env
virtualenv env/educa
source env/educa/bin/activate
```

安装Django到你的虚拟环境中通过以下命令：

    pip install Django==1.8.6
    
我们将要管理图片上传在我们的项目中，所以我们还需要安装Pillow通过以下命令：

    pip install Pillow==2.9.0
    
创建一个新项目使用以下命令：

    django-admin startproject educa
    
进入这个新的*educa*目录并且创建一个新应用使用以下命令：

```shell
cd educa
django-admin startapp courese
```

编辑*educa*项目的*settings.py*文件并且添加*courses*到`INSTALLED_APPS`设置中如下所示：

```python
INSTALLED_APPS = ( 
    'courses',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
)
```

*courses*应用现在已经在这个项目中激活。让我们定义模型给课程以及课程内容。

##构建课程模型

我们的在线学习平台将会提供课程在许多主题中。每一个课程都将会划分为一个可配置的模块编号，并且每个模块将会包含一个可配置的内容编号。将会有许多类型的内容：文本，文件，图片，或者视频。下面的例子展示了我们的课程目录的数据结构：

```
Subject 1
    Course 1
        Module 1
            Content 1 (images)
            Content 3 (text)
        Module 2
            Content 4 (text)
            Content 5 (file)
            Content 6 (video)
            ...
```

让我们来构建课程模型。编辑*courses*应用的*models.py*文件并且添加如下代码：

```python
from django.db import models
from django.contrib.auth.models import User
class Subject(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    class Meta:
        ordering = ('title',)
    def __str__(self):
        return self.title
        
class Course(models.Model):
    owner = models.ForeignKey(User,
                                 related_name='courses_created')
    subject = models.ForeignKey(Subject,
                                   related_name='courses')
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    overview = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    class Meta:
        ordering = ('-created',)
    def __str__(self):
        return self.title
        
class Module(models.Model):
    course = models.ForeignKey(Course, related_name='modules')
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    def __str__(self):
        return self.title
```

这些是最初的*Subject,Course,*以及*Module*模型。*Course*模型字段如下所示：

* owner：创建这个课程的教师。
* subject：这个课程属于的主题。这是一个*ForeingnKey*字段指向*Subject*模型。
* title：课程标题。
* slug：课程的slug。之后它将会被用在URLs中。
* overview：这是一个*TextFied*列用来包含一个关于课程的概述。
* created：课程被创建的日期和时间。它将会被Django自动设置当创建一个新的对象，因为`auto_now_add=True`。

每一个课程都被划分为多个模块。因此，*Module*模型包含一个*ForeignKey*字段用来指向*Course*模型。

打开shell并且运行一下命令来给这个应用创建最初的迁移：

    python manange.py makemigrations
    
你将会看到以下输出：

```
Migrations for 'courses':
     0001_initial.py:
       - Create model Course
       - Create model Module
       - Create model Subject
       - Add field subject to course
```

之后，运行一下命令来应用所有的迁移到数据库中：

    python manage.py migrate
    
你将会看到一个输出包含所有应用的迁移，包括Django的那些。这个输出将会包含以下行：

    Applying courses.0001_initial... OK
    
这告诉我们那个我们的*courses*引用模型已经同步到了数据库中。

##注册模型到管理平台中

我们将要添加课程模型到管理平台中。编辑*courses*应用目录下的*admin.py*文件并且添加以下代码：

```python
from django.contrib import admin
from .models import Subject, Course, Module

@admin.register(Subject)
class SubjectAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug']
    prepopulated_fields = {'slug': ('title',)}
   
class ModuleInline(admin.StackedInline):
    model = Module
@admin.register(Course)

class CourseAdmin(admin.ModelAdmin):
    list_display = ['title', 'subject', 'created']
    list_filter = ['created', 'subject']
    search_fields = ['title', 'overview']
    prepopulated_fields = {'slug': ('title',)}
    inlines = [ModuleInline]
```

课程应用的模型现在已经在管理平台中注册。我们使用`@admin.register()`装饰器替代了`admin.site.register()`方法。它们都提供了相同的功能。

##提供最初数据给模型

有时候你可能想要预装你的数据库通过使用硬编码数据。这是很有用的，当自动包含最初数据在项目设置中用来替代手工去添加数据。Django自带一个简单的方法来加载以及转储数据库中的数据到字段中，这被称为fixtures。

Django支持fixtures在JSON,XML,或者YAML格式中。我们将要创建一个fixture用来包含一些最初的*Subject*对象给我们的项目。

首先，创建一个超级用户使用如下命令：

    python manage.py createsuperuser
    
之后，运行开发服务器使用以下命令：

    python manage.py runserver
    
现在，打开 http://127.0.0.1:8000/admin/courses/subject/ 在你的浏览器中。创建一些主题通过使用管理平台。列页面看上去如下所示：

![django-10-1](http://ohqrvqrlb.bkt.clouddn.com/django-10-1.png)

运行一下命令在shell中：

    python manage.py dumpdata courese --indent=2
    
你会看到类似以下的输出：

```JSON
[
{
  "fileld:": {
    "title": "Programming",
    "slug": "programming"
  },
  "model": "courses.subject",
  "pk": 1
},
{
"fields": {
    "title": "Mathematics",
    "slug": "mathematics"
  },
  "model": "courses.subject",
  "pk": 2
}, 
{
"fields": {
    "title": "Physics",
    "slug": "physics"
  },
  "model": "courses.subject",
  "pk": 3
}, {
  "fields": {
    "title": "Music",
    "slug": "music"
  },
  "model": "courses.subject",
  "pk": 4
} 
]
```

*dumpdata*命令从数据库中转储数据到标准输出中，默认序列化为JSON。这串数据结构包含的信息关于这个模型以及它的字段将会被Django用来加载它到数据库中。

你可以提供应用名给这命令或者指定模型给输出数据使用`app.Model`格式。你还可以指定格式通过使用`--format`标记。默认的，*dumpdata*输出序列化数据给标准输出。当然，你可以表明一个输出文件通过使用`--output`标记。`--indent`标记允许你指定缩进。更多信息关于*udmpdata*参数，运行`python manage.py dumpdata --help`。

保存这个转储为一个fixtures文件到*orders*应用的*fixtures/*目录中，通过使用如下命令：

```shell
mkdir courses/fixtures
python manage.py dumpdata courses --indent=2 --output=courses/fixtures/
subjects.json
```

使用管理平台去移除你之前创建的主题。之后加载fixture到数据库中通过使用以下命令：

    python manage.py loaddata subjects.json
    
所有包含在fixture中的`subject`对象都会加载到数据库中。

默认的，Django会寻找每一个应用的*fixtures/*目录下的文件，但是你可以指定fixture文件的完整路径给*loaddata*命令。你还可以使用`FIXTURE_DIRS`设置来告诉Django去额外的目录寻找fixtures。

>Fixtures并不只对初始化数据有用，还可以提供简单的数据给你的应用或者数据请求给你的测试用例。

你可以找到更多关于如何使用fixtures在测试中，通过访问 https://docs.djangoproject.com/en/1.8/topics/testing/tools/#topics-testing-fixtures 。

如果你想要加载fixturres在模型迁移中，去看下Django的文档关于数据迁移。请记住，我们创建了一个定制迁移在**第九章，扩展你的商店**来迁移存在的数据在修改给翻译的模型之后。你可以找到迁移数据的文档，通过访问 https://docs.djangoproject.com/en/1.8/topics/migrations/#data-migrations 。

##给不同的内容创建模型

我们打算添加各种不同的内容类型给课程模块，例如文本，图片，文件以及视屏。我们需要一个通用的数据模型可以允许我们去存储不同的内容。在**第六章，跟踪用户行为**中，你已经学习过有关使用通用关系方便的创建外键能够指向任何模型的对象。我们将要创建一个*content*模型相当于模块内容以及定义一个通用关系来连接任意种类的内容。

编辑*courses*应用下的*models.py*文件并且添加如下导入：

```python
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey
```

之后添加如下代码到文件后面：

```python
class Content(models.Model):
    module = models.ForeignKey(Module, related_name='contents')
    content_type = models.ForeignKey(ContentType)
    object_id = models.PositiveIntegerField()
    item = GenericForeignKey('content_type', 'object_id')
```

这就是一个*Content*模型。一个模块包含多种内容，所有我们定义了一个`ForeignKey`字段给*module*模型。我们还设置了一个通用关系来连接对象从不同的模型中相当于不同的内容类型。请记住，我们需要三种不同的字段来设置一个通用关系。在我们的*Content*模型中，它们是：

* content_type：一个*ForeignKey*字段指向*ContentType*模型
* object_id：这是*PositiveIntegerField*用来存储有关联对象的关键字
* item：一个*GenericForeignKey*字段指向被关联的对象通过结合前两个字段

只有`content_type`和`object_id`字段有一个对应列在这个模型的数据库表中。`item`字段允许你去检索或者直接设置关联对象，并且它的功能是简历在其他两个字段之上。

我们将要使用一个不同的模型给每一种内容。我们的内容模型将会有很多共有的字段，但是它们将会有不同之处在它们存储的真实内容中。

##使用模型继承

Django支持模型继承。类似与Python中的标准类继承。Django提供以下三种方式来使用模型继承：

* Abstract models：非常有用当你想要安置一些公用信息到多个模型中。没有数据库表会被创建给抽象模型。
* Multi-table model inheritance：可适当的利用当每个模型经过慎重考虑都是一个它自身的完整的模型。每个模型都会创建一个数据库表。
* Proxy models：非常有用当你需要改变一个模型行为，比如说，包含额外的方法，修改默认管理器，或者使用不同的元选项。没有数据表会被创建给代理模型。

让我们对以上三者都来一次近距离的实践。

##抽象模型

一个抽象模型就是一个基础类，你定义在其中的字段就是你想要包含到所有子模型中的字段。Djnago不会创建任何数据库表给抽象模型。每个子模型都会创建一张数据库表，包含有继承自抽象类的字段以及在子模型中自己定义的字段。

为了抽象一个模型，你需要在`Meta`类中包含`abstract=True`。Django将会认出这个模型是一个抽象模型并且不会给它创建数据库表。为了创建子模型，你只需要基于这个抽象模型。下面就是一个例子关于一个抽象的*Content*模型和一个子的`Text`模型：

```python
from django.db import models

class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        abstract = True
        
class Text(BaseContent):
    body = models.TextField()
```

在这个例子中，Django将只会给`Text`模型创建表，包含`title`,`created`以及`body`字段。

##多表模型继承

在多表模型继承中，每个模型对应一个张数据库表。Django创建一个*OneToOneField*字段给子模型创建关系指向它的父模型。

为了使用多表继承，你必须基于一个存在的模型。djnago将会创建一张数据表给每个源头模型以及子模型。以下例子展示多表继承：

```python
from django.db import models

class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    
class Text(BaseContent):
    body = models.TextField()
```

Django将会包含一个自动生成的*OneToOneField*字段在`Text`模型中并且给每个模型创建一张数据库表。

##代理模型

代理模型被用于改变一个模型的行为，举个例子，包含额外的方法或者不同的元选项。每个模型对源头模型的数据库表起作用。为了创建一个代理模型，在这个模型的`Meta`类中添加`proxy=True`。

以下例子说明如何创建一个代理模型：

```python
from django.db import models
from django.utils import timezone
   
class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    
class OrderedContent(BaseContent):
    class Meta:
        proxy = True
        ordering = ['created']
        
    def created_delta(self):
        return timezone.now() - self.created
```

这里，我们定义了一个*OrderedContent*模型这是一个代理模型给*Content*模型使用。这个模型提供了一个默认的排序给查询集并且一个额外的`create_delta()`方法。这两个模型，`Content`和`OrderedContent`，对同一个数据库表起作用，并且通过任一一个模型都能通过ORM渠道连接到对象。

##创建内容模型

我们的*courses*应用的*Content*模型包含一个通用关系来连接不同类型的内容给该应用。我们将要创建一个不同的模型给每种类型的内容。所有内容模型将会有一些公用的字段，以及额外的字段去存储定制数据。我们将会创建一个抽象模型来提供公用字段给所有内容模型。

编辑*courses*应用的*models.py*文件，并且添加以下代码：

```python
class ItemBase(models.Model):
    owner = models.ForeignKey(User,
                                related_name='%(class)s_related')
    title = models.CharField(max_length=250)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True
    def __str__(self):
        return self.title
   
class Text(ItemBase):
    content = models.TextField()
   
class File(ItemBase):
    file = models.FileField(upload_to='files')
   
class Image(ItemBase):
    file = models.FileField(upload_to='images')
   
class Video(ItemBase):
    url = models.URLField() 
```

在这串代码中，我们定义了一个抽象模型命名为`ItemBase`。除此以外，我们在`Meta`类中设置`abstract=True`。在这个模型中，我们定义`owner,title,created`，以及`updated`字段。这些公用字段将会被所有的内容类型使用到。`owner`字段允许我们去存储哪个用户创建了这个内容。因为和这个字段是被定义在一个抽象类中，我们需要不同的`related_name`给每个子模型。Django允许我们去指定一个占位符给`model`类名在`related_name`属性类似`%(class)s`。为了做到这些，`related_name`对每个子模型都会自动生成。因为我们使用`%(class)s_related`作为`related_name`，给子模型的相对关系将各自是`text_related,file_related,image_related,`以及`vide0_related`。

我们已经定义了四种不同的内容模型，它们都继承自`ItemBase`抽象模型，它们是：

* Text：用来存储文本内容。
* File：用来存储文件，例如PDF。
* Image：用来存储图片文件。
* Video：用来存储视频。我们使用一个`URLField`字段来提供一个视频URL为了嵌入该视频。

每个子模型包含定义在`ItemBase`类中的字段以及它自己的字段。`text_related,file_related,image_related,`以及`vide0_related`都会各自创建一张数据库表。不会有数据库表连接到`ItemBase`模型，因为它是一个抽象模型。

编辑你之前创建的*Content*模型，修改它的`content_type`字段如下所示：

```python
content_type = models.ForeignKey(ContentType,
                      limit_choices_to={'model__in':('text',
                                           'video',
                                           'image',
                                           'file')})
```

我们添加一个`limit_choices_to`参数来限制`ContentType`对象可以被通用关系使用。我们使用`model__in`字段查找过滤这个查询给`ContentType`对象通过一个`model`属性就像'text','video','image',或者'file'。

让我们创建一个迁移来包含这些新的模型我们之前添加的。运行以下命令：

    python manage.py makemigrations
    
你会看到以下输出：

```shell
Migrations for 'courses':
     0002_content_file_image_text_video.py:
       - Create model Content
       - Create model File
       - Create model Image
       - Create model Text
       - Create model Video
```

之后，运行一下命令来应用新的迁移：

    python manage.py migrate
    
你会在输出结果看到以下内容：

```shell
Running migrations:
     Rendering model states... DONE
     Applying courses.0002_content_file_image_text_video... OK
```

我们之前创建的模型对于添加不同的内容给课程模块是很合适的。但是，仍然有一些东西是被遗漏的在我们的模型中。课程模块和内容应当跟随一个特定的顺序。我们需要一个字段，这个字段允许我们简单的排序它们。

##创建定制模型字段

Django自带一个完整的模型字段采集能让你用来构建你的模型。当然，你也可以创建你自己的模型字段来存储定制数据或者改变现有字段的行为。

我们需要一个字段允许我们给对象们定义次序。如果你想通过Djanog提供的一个字段来方便的处理这点，你大概会想到添加一个`PositiveIntegerField`给你的模型。这是一个好的起点。我们可以创建一个定制字段，该字段继承自`PositiveIntegerField`并且提供额外的行为。

有两种相关的功能我们将构建到我们的次序字段中：

* 自动分配一个次序值当没有指定的次序被提供的时候。当没有次数被提供的时候存储一个对象，我们的字段将自动分配下一个次序，该次序基于最后存在次序的对象。如果有两个对象，分别是次序1和次序2，当保存第三个对象的时候，我们会自动分配次序3给第三个对象如果没有给予指定的次序。
* 次序对象关于其他的字段。课程模块将按照它们所属的课程和相关模块的内容进行排序。

创建一个新的*fields.py*文件到*courses*应用目录下，然后添加以下代码：

```python
from django.db import models
from django.core.exceptions import ObjectDoesNotExist

class OrderField(models.PositiveIntegerField):

    def __init__(self, for_fields=None, *args, **kwargs):
        self.for_fields = for_fields
        super(OrderField, self).__init__(*args, **kwargs)
        
    def pre_save(self, model_instance, add):
        if getattr(model_instance, self.attname) is None:
            # no current value
            try:
                qs = self.model.objects.all()
                if self.for_fields:
                    # filter by objects with the same field values
                    # for the fields in "for_fields"
                    query = {field: getattr(model_instance, field) for field in self.for_fields}
                    qs = qs.filter(**query)
                # get the order of the last item
                last_item = qs.latest(self.attname)
                value = last_item.order + 1
            except ObjectDoesNotExist:
                value = 0
            setattr(model_instance, self.attname, value)
            return value
        else:
            return super(OrderField,
                        self).pre_save(model_instance, add)                    
```

这就是我们的定制`OrderField`.它继承自Django提供的`PositiveIntegerField`字段。我们的`OrderField`字段需要一个可选的`for_fields`参数，这个参数允许我们表明次序根据这些字段进行计算。

我们的字段覆盖`PositiveIntegerField`字段的`pre_save()`方法，这字段会在保存这个字段到数据库之前进行执行。在这个方法中，我们做了以下操作：

* 1 我们检查在模型实例中的字段是否已有一个值。我们是`self.attname`，它是在这个模型中给予这个字段的属性名。如果在这个属性的值不同于`None`，我们就会进行如下操作来计算出一个次序给它：

    * 1 我们构建一个查询集去检索所有对象给这个字段的模型。我们通过访问`self.model`来检索该字段所属的模型类。
    * 2 我们通过模型字段中的那些被定义在`for_fields`参数中的字段的当前值来过滤这个查询集（如果有的话）。为了做到这点，我们通过给予的字段来计算次序。
    * 3 我们从数据库中使用最高的次序来检索对象通过是用`last_item = qs.latest(self.attname)`。如果没有找到对象，我们假定这个对象是第一个并且分配次序0给它。
    * 4 如果找到一个对象，我们给找到的最高次序增加1。
    * 5 我们分配计算过的次序给在模型实例中的字段的值通过使用`setattr()`并且返回它。

* 2 如果这个模型实例有一个值给当前的字段，我们不需要做任何事情。

> 当你创建定制模型字段，使它们通过。避免硬编码数据被依赖一个指定模型或者字段。你的字段才能在任意模型中起效。

你可以找到更多的信息关于编写定制模型字段，通过访问 https://docs.djangoproject.com/en/1.8/howto/custom-model-fields/ 。

让我们添加新的字段给我们的模型。编辑*courses*应用的*models.py*文件，导入新的字段如下所示：

    from .fields import OrderField
    
之后，添加以下`OrderField`字段给`Module`模型：

    order = OrderField(blank=True, for_fields=['course'])

我们命名新的字段为`order`，并且我们指定该字段的次序根据课程计算通过设置`for_fields=['course']`。这意味着新的模块的次序将会是最后的同样的*Course*对象模块的次序增加1。现在你可以编辑`Module`模型的`__str__()`方法来包含它的次序如下所示：

```python
def __str__(self):
    return '{}. {}'.format(self.order, self.title)
```

模块内容也需要跟随一个特定的次序。添加一个`OrderField`字段给`Content`模型如下所示：

    order = OrderField(blank=True, for_fields=['module'])
    
这一次，我们指定这个次序根据`moduel`字段进行计算。最后，让我们给这两个模型都添加一个默认的序列。添加如下`Meta`类给`Module`和`Content`模型：

```python
class Meta:
    ordering = ['order']
```

`Module`和`Content`模型现在看上去如下所示：

```python
class Module(models.Model):
    course = models.ForeignKey(Course,related_name='modules')
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    order = OrderField(blank=True, for_fields=['course'])
    
    class Meta:
        ordering = ['order']
    def __str__(self):
        return '{}. {}'.format(self.order, self.title)
        
class Content(models.Model):
    module = models.ForeignKey(Module, related_name='contents')
    content_type = models.ForeignKey(ContentType,
                    limit_choices_to={'model__in':('text',
                                                      'video',
                                                      'file')})
    item = GenericForeignKey('content_type', 'object_id')
    order = OrderField(blank=True, for_fields=['module'])
    class Meta:
        ordering = ['order']        
```

让我们创建一个新模型迁移来体现新的次序字段。打开shell并且运行如下命令：

    python manage.py makemigrations courses
    
你会看到如下输出：

```shell
You are trying to add a non-nullable field 'order' to content without a default; we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows)
 2) Quit, and let me add a default in models.py
Select an option:
```

Django正在告诉我们由于我们添加了一个新的字段给已经存在的模型，我们必须提供一个默认值给数据库中已经存在的各行记录。如果这个字段有`null=True`，它将会采用空值并且Django将会创建这个迁移而不会找我们要一个默认值。我们可以指定一个默认值或者取消这次迁移然后在创建这个迁移之前去`models.py`文件中给`order`字段添加一个`default`属性。

输入 1 然后按下回车来提供一个默认值给已经存在的记录。你将会看到如下输出：

```shell
Please enter the default value now, as valid Python
The datetime and django.utils.timezone modules are available, so you can do e.g. timezone.now()
>>>
```

输入 0 作为给已经存在的记录的默认值然后按下回车。Djanog将会询问你还需要一个默认值给`Module`模型。选择第一个选项然后再次输入 0 作为默认值。最后，你将会看到如下类似的输入：

```shell
Migrations for 'courses':
 0003_auto_20150701_1851.py:
    - Change Meta options on content
    - Change Meta options on module
    - Add field order to content
    - Add field order to module
```

之后，应用新的迁移通过以下命令：

    python manage.py migrate
    
这个命令的输出将会通知你这次迁移成功的应用，如下所示：

    Applying courses.0003_auto_20150701_1851... OK

让我们测试我们新的字段。打开shell使用`python manage.py shell`然后创建一个新的课程如下所示：

```shell
>>> from django.contrib.auth.models import User
>>> from courses.models import Subject, Course, Module
>>> user = User.objects.latest('id')
>>> subject = Subject.objects.latest('id')
>>> c1 = Course.objects.create(subject=subject, owner=user,
title='Course 1', slug='course1')
```

我们已经在数据库中创建了一个课程。现在让我们给课程添加模块然后看下模块的次序是如何自动计算的。我们创建一个初始模板然后检查它的次序：

```shell
>>> m1 = Module.objects.create(course=c1, title='Module 1')
>>> m1.order
0
```

`OrderField`设置这个模块的值为 0，因为这个模块是这个课程的第一个`Module`对象。现在我们创建第二个对象给这个课程：

```shell
>>> m2 = Module.objects.create(course=c1, title='Module 2')
>>> m2.order
1
```

`OrderField`计算出下一个次序值是已经存在的对象中最高的次序值加上 1。让我们创建第三个模块强制指定一个次序：

```shell
>>> m3 = Module.objects.create(course=c1, title='Module 3', order=5)
>>> m3.order
5
```

如果我们指定了一个定制次序，`OrderField`字段将不会进行干涉，然后`order`的值将会使用指定的次序。

让我们添加第四个模块：

```shell
>>> m4 = Module.objects.create(course=c1, title='Module 4')
>>> m4.order
6
```

这第四个模块的次序会被自动设置。我们的`OrderField`字段不会保证所有的次序值是连续的。无论如何，它会根据已经存在的次序值并且分配下一个次序基于已经存在的最高次序。

让我们创建第二个课程并且添加一个模块给它：

```shell
>>> c2 = Course.objects.create(subject=subject, title='Course 2', slug='course2', owner=user)
>>> m5 = Module.objects.create(course=c2, title='Module 1')
>>> m5.order
0
```

为了计算这个新模块的次序，该字段只需要考虑基于同一课程的已经存在的模块。由于这是第二个课程的第一个模块，次序的结果值就是 0 。这是因为我们指定`for_fields=['course']`在`Module`模型的`order`字段中。

恭喜你！你已经成功的创建了你的第一个定制模型字段。

##创建一个内容管理系统

到现在我们已经创建了一个多功能数据模型，我们将要构建一个内容管理系统（CMS）。这个CMS将允许教师去创建课程以及管理课程的内容。我们需要提供以下功能：

* 登录CMS。
* 排列教师创建的课程。
* 创建，编辑以及删除课程。
* 添加模块到一个课程中并且重新排序它们。
* 添加不同类型的内容给每个模块并且重新排序内容。

##添加认证系统

我们将要使用Django的认证框架到我们的平台中。教师和学生都将会是Django *User*模型的一个实例。从而，他们将能够登录这个站点通过使用`django.contrib.auth`的认证视图。

编辑*educa*项目的主*urls.py*文件然后包含Django认证框架的`login`和`logout`视图：

```python
from django.conf.urls import include, url
from django.contrib import admin
from django.contrib.auth import views as auth_views

urlpatterns = [
    url(r'^accounts/login/$', auth_views.login, name='login'),
    url(r'^accounts/logout/$', auth_views.logout, name='logout'),
    url(r'^admin/', include(admin.site.urls)),
]
```

##创建认证模板

在*courses*应用目录下创建如下文件结构：

```shell
templates/
    base.html
    registration/
        login.html
        logged_out.html
```

在构建认证模板之前，我们需要给我们的项目准备好基础模板。编辑*base.html*模板文件然后添加以下内容：

```html
{% load staticfiles %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>{% block title %}Educa{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
      <a href="/" class="logo">Educa</a>
       <ul class="menu">
         {% if request.user.is_authenticated %}
           <li><a href="{% url "logout" %}">Sign out</a></li>
         {% else %}
           <li><a href="{% url "login" %}">Sign in</a></li>
         {% endif %}
       </ul>
     </div>
     <div id="content">
       {% block content %}
       {% endblock %}
     </div>
     <script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
     <script>
       $(document).ready(function() {
         {% block domready %}
         {% endblock %}
       });
     </script>
   </body>
</html>
```

这个基础模板将会被其他的模板扩展。在这个模板中，我们定义了以下区块：

* title：这个区块是给别的模板用来给每个页面添加定制的标题。
* content：这个是内容的主区块。所有扩展基础模板的模板都可以添加各自的内容到这个区块。
* domready：位于jQuery的`$document.ready()`方法里面。它允许我们执行代码当DOM完成加载的时候。

这个模板使用的CSS样式位于本章实例代码的*courses*应用下的*static/*目录下。你可以拷贝*static/*目录到你的项目的相同目录下来使用它们。

编辑*registration/login.html*模板并且添加以下代码：

```html
{% extends "base.html" %}

{% block title %}Log-in{% endblock %}

{% block content %}
     <h1>Log-in</h1>
     <div class="module">
       {% if form.errors %}
         <p>Your username and password didn't match. Please try again.</p>
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
     </div>
{% endblock %}
```

这是一个给Django的`login`视图用的标准登录模板。编辑*registration/logged_out.html*模板然后添加以下代码：

```shell
{% extends "base.html" %}
   
{% block title %}Logged out{% endblock %}

{% block content %}
     <h1>Logged out</h1>
     <div class="module">
       <p>You have been successfully logged out. You can <a href="{% url"login" %}">log-in again</a>.</p>
     </div>
{% endblock %}
```

这个模板将会在用户登出后展示。通过命令`python manage.py runserver`命令运行开发服务器然后在你的浏览器中打开 http://127.0.0.1:8000/accounts/login/ 。你会看到如下登录页面：

![django-10-2](http://ohqrvqrlb.bkt.clouddn.com/django-10-2.png)

##创建基于类的视图

我们将要构建一些视图用来创建，编辑，以及删除课程。为了这个目的我们将会使用基于类的视图。编辑*courses*应用的*views.py*文件并且添加如下代码：

```python
from django.views.generic.list import ListView
from .models import Course

class ManageCourseListView(ListView):
    model = Course
    template_name = 'courses/manage/course/list.html'
    
    def get_queryset(self):
        qs = super(ManageCourseListView, self).get_queryset()
        return qs.filter(owner=self.request.user)
```

以上就是`ManageCourseListView`视图。它从Django的通用`ListView`继承而来。我们重写了这个视图的`get_queryset()`方法来只对当前用户创建的课程进行检索。为了阻止用户对不是由他们创建的课程进行编辑，更新或者删除操作，我们还需要重写在创建，更新以及删除视图中的`get_queryse()`方法。当你需要去提供一个指定行为给多个基于类的视图，推荐你使用`mixins`。

##对基于类的视图使用mixins

mixins是一种特殊的用于一个类的多重继承。你可以使用它们来提供常见的离散功能，添加到其他的mixins，允许你去定义一个类的行为。有两种场景要使用mixins：

* 你想要提供多个可选的特性给一个类
* 你想要使用一个特定的特性在多个类上

你可以找到关于如何在基于类的视图上使用mixins的文档，通过访问 https://docs.djangoproject.com/en/1.8/topics/class-based-views/mixins/ 。

Django自带多个mixins用来提供额外的功能给你的基于类的视图。你可以找到所有的mixins在 https://docs.djangoproject.com/en/1.8/ref/class-based-views/mixins/ 。

我们将要创建一个mixin类来包含一个公用的行为并且将它给课程的视图使用。编辑*courses*应用的*views.py*文件，把它修改成如下所示：

```python
from django.core.urlresolvers import reverse_lazy
from django.views.generic.list import ListView
from django.views.generic.edit import CreateView, UpdateView, \
                                         DeleteView
from .models import Course

class OwnerMixin(object):
    def get_queryset(self):
        qs = super(OwnerMixin, self).get_queryset()
        return qs.filter(owner=self.request.user)

class OwnerEditMixin(object):
    def form_valid(self, form):
        form.instance.owner = self.request.user
        return super(OwnerEditMixin, self).form_valid(form)
   
class OwnerCourseMixin(OwnerMixin):
    model = Course
   
class OwnerCourseEditMixin(OwnerCourseMixin, OwnerEditMixin):
    fields = ['subject', 'title', 'slug', 'overview']
    success_url = reverse_lazy('manage_course_list')
    template_name = 'courses/manage/course/form.html'
    
class ManageCourseListView(OwnerCourseMixin, ListView):
    template_name = 'courses/manage/course/list.html'
    
class CourseCreateView(OwnerCourseEditMixin, CreateView):
    pass
    
class CourseUpdateView(OwnerCourseEditMixin, UpdateView):
    pass
    
class CourseDeleteView(OwnerCourseMixin, DeleteView):
    template_name = 'courses/manage/course/delete.html'
    success_url = reverse_lazy('manage_course_list')
```

在上述代码中，我们创建了`OwnerMixin`和`OwnerEditMixin`这两个mixin。我们将要使用这些mixins与Django提供的`ListView`，`CreateView`，`UpdateView`以及`DeleteView`视图结合。`Ownermixin`导入了以下方法。

* `get_queryset()`：这个方法被视图用来获取基础查询集。我们的mixin将会重写这个方法使用`owner`属性对对象进行过滤来检索属于当前用户的对象（request.user)。

`OwnerEditMixin`导入了以下方法：

* `form_valid()`：这个方法被视图用来使用Django的`ModelFormMixin` mixin，也就是说，带有表单的视图或模型表单的视图比如`Createview`和`UpdateView.form_valid()`当提交的表单是有效的时候就会被执行。这个方法默认的行为是保存实例（对于模型表单）以及重定向用户到`success_url`。我们重写了这个方法来自动设置当前的用户到本次会被保存的对象的`owner`属性中。为了做到前面所说的，我们设置自动分配一个拥有者给该对象，当该对象被保存的时候。

我们的`OwnerMixin`类能够被视图用来和任意模型进行交互使模型包含一个`owner`属性。

我们还定义了一个`OwnercourseMixin`类，该类继承`OwnerMixin`并且提供以下属性给子视图：

* `model`：这个模型给查询集使用。被所有视图使用。

我们定义一个`OwnerCourseEditMixin` mixin通过以下属性：

* `fields`：这些模型字段用来从`CreateView`和`UpdateView`视图中构建模型。
* `success_url`：被`CreateView`和`UpdateView`使用来在表单成功提交之后重定向用户。我们之后将会创建一个名为`manage_course_list`的URL来使用。

最后，我们创建以下视图，这些视图都是基于`OwnerCourseMixin`的子类：

* `MangeCourselISTvIEW`：排序用户创建的课程。它从`OwnerCourseMixin`和`ListView`继承而来。
* `CoursecreateView`：使用模型表单来创建一个新的`Course`对象。它使用定义在`OwnerCourseEditMixin`中的字段来构建一个表单模型并且也是`CreateView`的子类。
* `CourseUpdateView`：允许编辑一个现有的`Course`对象。它从`OwnerCourseMixin`和`UpdateView`继承而来。
* `CourseDeleteView`：从`OwnerCourseMixin`和通用的`DeleteView`继承而来。定义`success_url`在对象被删除的时候重定向用户。

##使用组和权限

我们已经创建了基础的视图来管理课程。但是目前所有的用户都可以使用这些视图。我们想要限制这些视图从而只有教师有权限去创建和管理课程。Django认证框架包含一个权限系统允许你去分配权限给用户和组。我们将要创建一个组给教师用户并且分配权限给他们可以创建，更新以及删除课程。

使用命令`python manage.py runserver`命令运行开发服务器并且在你的浏览器中打开 http://127.0.0.1:8000/admin/auth/group/add/ 来创建一个新的`Group`对象。添加的组名为*Instructors*,然后选择*courses*应用中的除了*Subject*模型的所有权限，如下所示：

![django-10-3](http://ohqrvqrlb.bkt.clouddn.com/django-10-3.png)

如你所见，有三种不同的权限给每个模型：**Can add**，**can change**以及**Can delete**。选择好给这个组的权限之后，点击**Save**按钮。

Django会自动给模型创建权限，但是你也可以创建定制的权限。你可以找到更多关于添加定制权限的文档，通过访问 https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#custom-permissions 。

打开 http://127.0.0.1:8000/admin/auth/user/add/ 然后创建一个新用户。编辑这个用户然后添加**Instructors**组给这个用户如下所示：

![django-10-4](http://ohqrvqrlb.bkt.clouddn.com/django-10-4.png)

用户会继承他们所在组的权限，但是你也可以在管理平台上添加单独的权限给一个指定的用户。当用户的*is_superuser*设置为*True*的时候会自动拥有所有的权限。

##限制使用基于类的视图

我们将要限制使用基于类的视图从而只有那些拥有合适权限的用户才能添加，修改，或者删除`Course`对象认证框架包含一个`permission_required`装饰器来限制对视图的使用。Django 1.9将会包含权限mixins给基于类的视图**（译者注：到目前为止，Django版本已经是1.10.6）**。然而，Django1.8还没有包含它们。因此，我们将要第三方模块提供的权限mixins，该第三方模块名为 django-braces**（译者注：。。。。。。下面我是不是可以不用翻译了。。。。。。）**。

##使用django-braces的mixins

Django-braces是一个第三方的模块，它包含一个通用mixins的采集给Django使用。这些mixins提供额外的特性给基于类的视图。你可以看到django-braces提供的所有mixins列表，通过访问 http://django-braces.readthedocs.org/en/latest/。

使用pip命令安装django-braces：

    pip install django-braces==1.8.1
    
我们将要使用以下两个django-braces提供的mixinx来限制视图的使用：

* *LoginRequiredMixin*：复制`login_required`装饰器的功能。
* *PermissionRequiredMixin*：准许拥有指定权限的用户使用该视图。请记住，超级用户自动拥有所有权限。

编辑*courses*应用的*views.py*文件，添加如下导入：

```python
from braces.views import LoginRequiredMixin,
                            PermissionRequiredMixin
```

像下面一样让`OwnerCourseEditMixin`继承`LoginRequiredMixin`：

```python
class OwnerCourseEditMixin(OwnerMixin, LoginRequiredMixin):
    model = Course
    fields = ['subject', 'title', 'slug', 'overview']
    success_url = reverse_lazy('manage_course_list')
```

之后，添加一个`permission_required`属性给创建，跟新，以及删除视图，如下所示：

```python
class CourseCreateView(PermissionRequiredMixin,
                       OwnerCourseEditMixin,
                       CreateView):
    permission_required = 'courses.add_course'
   
class CourseUpdateView(PermissionRequiredMixin,
                       OwnerCourseEditMixin,
                       UpdateView):
    template_name = 'courses/manage/course/form.html'
    permission_required = 'courses.change_course'
    
class CourseDeleteView(PermissionRequiredMixin, 
                       OwnerCourseMixin,
                       DeleteView):
    template_name = 'courses/manage/course/delete.html'
    success_url = reverse_lazy('manage_course_list')
    permission_required = 'courses.delete_course'
```

`PermissionRequiredMixin`会在用户使用视图的时候检查该用户是否有指定在`permission_required`属性中的权限。我们的视图现在只准许有适当权限的用户使用。

让我们给以上视图创建URLs。在*courses*应用目录中创建新的文件命名为*urls.py*。添加以下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^mine/$',
        views.ManageCourseListView.as_view(),
        name='manage_course_list'),
    url(r'^create/$',
        views.CourseCreateView.as_view(),
        name='course_create'),
    url(r'^(?P<pk>\d+)/edit/$',
        views.CourseUpdateView.as_view(),
        name='course_edit'),
    url(r'^(?P<pk>\d+)/delete/$',
        views.CourseDeleteView.as_view(),
        name='course_delete'),
]
```

以上的URL模式是给列表，创建，编辑以及删除课程试图使用的。编辑*educa*项目的主*urls.py*文件然后包含*courses*应用的URL模式，如下所示：

```python
urlpatterns = [
    url(r'^accounts/login/$', auth_views.login, name='login'),
    url(r'^accounts/logout/$', auth_views.logout, name='logout'),
    url(r'^admin/', include(admin.site.urls)),
    url(r'^course/', include('courses.urls')),
]
```

我们需要给这些视图创建模块。在*courses*应用中创建以下目录以及文件：

```shell
courses/
       manage/
           course/
               list.html
               form.html
               delete.html
```

编辑 *courses/manage/course/list.html*模板并且添加如下代码：

```html
{% extends "base.html" %}

{% block title %}My courses{% endblock %}

{% block content %}
     <h1>My courses</h1>
     <div class="module">
       {% for course in object_list %}
         <div class="course-info">
           <h3>{{ course.title }}</h3>
           <p>
             <a href="{% url "course_edit" course.id %}">Edit</a>
             <a href="{% url "course_delete" course.id %}">Delete</a>
           </p>
         </div>
       {% empty %}
         <p>You haven't created any courses yet.</p>
       {% endfor %}
       <p>
         <a href="{% url "course_create" %}" class="button">Create new course</a>
         </p>
    </div>
{% endblock %}
```

这是`ManageCourseListView`视图的模板。在这个模板中，我们通过当前用户来排列课程。我们给每个课程都包含了编辑或者删除链接，以及一个创建新课程的链接。

使用命令`python manage.py runserver`命令运行开发服务器。在你的浏览器中打开      http://127.0.0.1:8000/accounts/login/?next=/course/mine/ 然后使用*Instrctors*组中的一个用户进行登录。登录完成后，你会被重定向到 http://127.0.0.1:8000/course/mine/ 并且你会看到如下页面：

![django-10-5](http://ohqrvqrlb.bkt.clouddn.com/django-10-5.png)

这个页面将会展示所有当前用户创建的课程。

让我们创建一个给创建和更新课程视图使用的模板，该模板用来展示表单。编辑*courses/manage/course/form.html*模板并且输入以下代码：

```html
   {% extends "base.html" %}
   {% block title %}
     {% if object %}
       Edit course "{{ object.title }}"
     {% else %}
       Create a new course
     {% endif %}
   {% endblock %}
   {% block content %}
     <h1>
       {% if object %}
         Edit course "{{ object.title }}"
       {% else %}
         Create a new course
       {% endif %}
     </h1>
     <div class="module">
       <h2>Course info</h2>
       <form action="." method="post">
         {{ form.as_p }}
         {% csrf_token %}
         <p><input type="submit" value="Save course"></p>
       </form>
     </div>
   {% endblock %}  
```

这个*form.html*模板被`CoursecREATEvIEW`和`courseUpdateView`视图使用。在这个模板中，我们检查是否有个*object*变量在上下文环境中。如果*object*存在上下文环境中，我们就知道我们正在更新一个存在的课程，并且我们在页面标题中使用它。如果不存在，我们就要创建一个新的*Course*对象。

在你的浏览器中打开 http://127.0.0.1:8000/course/mine/ 然后点击**Create new course**按钮。你会看到如下页面：

![django-10-6](http://ohqrvqrlb.bkt.clouddn.com/django-10-6.png)

填写好表单内容然后点击**Save course**按钮。这个课程将会被保存并且你将会被重定向到课程列表页面。它看上去如下所示：

![django-10-7](http://ohqrvqrlb.bkt.clouddn.com/django-10-7.png)

之后，点击你刚才创建的课程的**Edit**链接。你将会再次看到表单，但是这一次你将编辑一个已经存在的*Course*对象而不是创建新课程。

最后，编辑*courses/manage/course/delete.html*模板然后添加以下代码：

```html
   {% extends "base.html" %}
   {% block title %}Delete course{% endblock %}
   {% block content %}
     <h1>Delete course "{{ object.title }}"</h1>
     <div class="module">
       <form action="" method="post">
         {% csrf_token %}
         <p>Are you sure you want to delete "{{ object }}"?</p>
         <input type="submit" class"button" value="Confirm">
       </form>
     </div>
   {% endblock %}
```

这个模板是给`CourseDeleteView`视图使用的。这个视图从Django提供的`DeleteView`视图继承而来，`DeleteView`视图期望用户确认删除一个对象。

打开你的浏览器，点击你的课程的**Delete**链接。你会看到如下确认页面：

![django-10-8](http://ohqrvqrlb.bkt.clouddn.com/django-10-8.png)

点击**CONFIRM**按钮。这个课程将会被删除并且你再次会被重定向到课程列表页面。

教师们现在可以创建，编辑，以及删除课程。下一步，我们需要提供他们一个内容管理系统来给课程添加模块以及内容。我们将从管理课程模块开始。

##使用formsets

Django自带一个抽象层用于在同一个页面中使用多个表单。这些表单的组合成为formsets。formsets能管理多个*Form*或者*ModelForm*表单实例。所有的表单都可以一次性提交并且formset会照顾到一些事情，例如，表单的初始化数据展示，限制表单能够提交的最大数字，以及所有表单的验证。

formsets包含一个`is_valid()`方法来一次性验证所有表单。你还可以提供初始数据给表单以及指定展示任意多的额外的空的表单。

你可以学习到更多关于formsets，通过访问 https://docs.djangoproject.com/en/1.8/topics/forms/modelforms/#model-formsets 。

##管理课程模块

由于课程会被分为可变数量的模块，因此在这里使用formets是有意义的。在*courses*应用目录下创建一个*forms.py*文件，然后添加以下代码：

```python
from django import forms
from django.forms.models import inlineformset_factory
from .models import Course, Module

ModuleFormSet = inlineformset_factory(Course,
                                         Module,
                                         fields=['title',
                                                 'description'],
                                         extra=2,
                                         can_delete=True)
```

以上就是`ModuleFormSet` formset。我们使用Django提供的`inlineformset_factory()`函数来构建它。内联formsets是在formsets之上的一个小抽象，用于方便被关联对象的操作。这个函数允许我们去给关联到一个*Course*对象的*Module*对象动态的构建一个模型formset。

我们使用以下参数去构建formset：

* fields：这个字段将会被formset中的每个表单包含。
* extra：允许我们设置在formset中显示的空的额外的表单数。
* can_delete：如果你将这个参数设置为*True*，Django将会包含一个布尔字段给所有的表单，该布尔字段将会渲染成一个复选框。允许你确定这个对象你真的要进行删除。

编辑*courses*应用的*views.py*文件并且添加如下代码：

```python
from django.shortcuts import redirect, get_object_or_404
from django.views.generic.base import TemplateResponseMixin, View
from .forms import ModuleFormSet

class CourseModuleUpdateView(TemplateResponseMixin, View):
    template_name = 'courses/manage/module/formset.html'
    course = None
    
    def get_formset(self, data=None):
        return ModuleFormSet(instance=self.course,data=data)

    def dispatch(self, request, pk):
        self.course = get_object_or_404(Course,
                                        id=pk,
                                        owner=request.user)
        return super(CourseModuleUpdateView,
                     self).dispatch(request, pk)
    
    def get(self, request, *args, **kwargs):
        formset = self.get_formset()
        return self.render_to_response({'course': self.course,
                                        'formset': formset})
    
    def post(self, request, *args, **kwargs):
        formset = self.get_formset(data=request.POST)
        if formset.is_valid():
            formset.save()
            return redirect('manage_course_list')
        return self.render_to_response({'course': self.course,
                                        'formset': formset})        
```

`CourseModuleUpdateView`视图控制formset给一个指定的课程添加，更新，以及删除模块。这个视图继承自以下的mixins和视图：

* TemplateResponseMixin：这个mixins负责渲染模板以及返回一个HTTP响应。它需要一个template_name属性，该属性指明要被渲染的模板，并提供`render_to_ response()`方法来传递上下文并渲染模板。

* View：Django提供的基础的基于类的视图。

在这个视图中，我们导入以下方法：

* get_formset()：我们定义这个方法去避免重复构建formset的代码。我们使用可选数据为给予的`Course`对象创建一个`ModuleFormSet`对象。

* dispatch()：这个方法由`View`类提供。它需要一个HTTP请求及其参数并尝试委托一个与使用的HTTP方法匹配的小写方法：GET请求被委派给`get()`方法和一个POST请求到`post()`。在这种方法中，我们使用`get_object_or_404()`快捷方式函数获取属于当前用户的给予id参数的Course对象。我们将这串代码包含在`dispatch()`方法中是因为我们需要检索所有GET和POST请求的课程。我们保存该对象到这个视图的`course`属性给使它能被别的方法使用。

* get()：给GET请求执行。我们构建一个空的`ModuleFormSet` formset并且使用`TemplateResponseMixin`提供的`render_to_response()`方法将它与当前的`Course`对象一起渲染到模板中。
* post()：给POST请求执行。在这个方法中，我们执行以下操作：

    * 1 我们使用提交的数据构建一个`ModuleFormSet`实例。
    * 2 我们执行formset的`is_valid()`方法来验证其中的所有表单。
    * 3 如果这个formset验证通过，我们通过调用`save()`方法来保存它。在这点上，任何的修改操作，例如增加，更新或者标记模块用来删除，都会应用到数据库中。之后，我们重定向用户到`manage_course_list` URL。如果这个formset没有通过验证，我们渲染模板展示所有内置的错误信息。

编辑*courses*应用的*urls.py*文件，添加以下URL模式：

```python
url(r'^(?P<pk>\d+)/module/$',
    views.CourseModuleUpdateView.as_view(),
    name='course_module_update'),
```

在*courses/manage/*模板目录中创建一个新的目录命名为*module*。创建一个*courses/manage/module/formset.html*模板并且添加以下代码：

```html
{% extends "base.html" %}

{% block title %}
     Edit "{{ course.title }}"
{% endblock %}

{% block content %}
     <h1>Edit "{{ course.title }}"</h1>
     <div class="module">
       <h2>Course modules</h2>
       <form action="" method="post">
         {{ formset }}
         {{ formset.management_form }}
         {% csrf_token %}
         <input type="submit" class="button" value="Save modules">
       </form>
     </div>
{% endblock %}
```

在这个模板中，我们创建一个`<form>`HTML元素，在其中我们包含我们的`formset`。我们还通过变量`{{ formset.management_form }}`包含给formset使用的管理表单。这个管理表单包含隐藏的字段去控制保单的初始化，总数，最小值和最大值。如你所见，创建一个formset非常容易。

编辑*courses/manage/course/list.html*模板并且在课程编辑和删除链接下方添加以下链接给`course_module_update`使用：

```html
<a href="{% url "course_edit" course.id %}">Edit</a>
<a href="{% url "course_delete" course.id %}">Delete</a>
<a href="{% url "course_module_update" course.id %}">Edit modules</a>
```

我们已经包含了用来编辑课程模板的链接。在你浏览器中打开 http://127.0.0.1:8000/course/mine/ 然后选择一个课程点击对应的**Edit modules**链接。你会看到一个如下的formset：

![django-10-9](http://ohqrvqrlb.bkt.clouddn.com/django-10-9.png)

这个formset包含所有在这个课程中存在的*Module*对象的表单。在这些表单之后，有两个空的额外的表单会被展示因为我们给`ModuleFormSet`设置`extra=2`。当你保存这个formset的时候，Django将会包含另外两个额外的字段来添加新的模块。

##添加内容给课程模块

现在，我们需要一个方法来添加内容给课程模块。我们有四种不同的内容类型：文本，视频，图片以及文件。我们可以考虑创建四个不同的视图去保存内容，给每个模型都对应上一个视图。然而，我们将采取更通用的方法，并创建一个处理创建或更新任何内容模型的对象的视图。

编辑*courses*应用的*views.py*文件并且添加如下代码：

```python
from django.forms.models import modelform_factory
from django.apps import apps
from .models import Module, Content

class ContentCreateUpdateView(TemplateResponseMixin, View):
    module = None
    model = None
    obj = None
    template_name = 'courses/manage/content/form.html'
    
    def get_model(self, model_name):
        if model_name in ['text', 'video', 'image', 'file']:
            return apps.get_model(app_label='courses',
                                     model_name=model_name)
        return None
    
    def get_form(self, model, *args, **kwargs):
        Form = modelform_factory(model, exclude=['owner',
                                                    'order',
                                                    'created',
                                                    'updated'])
        return Form(*args, **kwargs)
    
    def dispatch(self, request, module_id, model_name, id=None):
        self.module = get_object_or_404(Module,
                                        id=module_id,
                                    course__owner=request.user)
        self.model = self.get_model(model_name)
        if id:
            self.obj = get_object_or_404(self.model,
                                        id=id,
                                        owner=request.user)
        return super(ContentCreateUpdateView,
              self).dispatch(request, module_id, model_name, id)
```

以上是`ContentCreateUpdateView`视图的第一部分。这个视图允许我们去创建和更新不同模块的内容。这个视图定义了以下方法：

* get_model()：在这儿，我们会对被给予的模型名是否四种内容模型中的一种：text,video,image以及file.之后我们使用Django的`apps`模块去通过给予的模型名来获取实际的类。如果给予的模型名不是其中的一种，我们返回`None`。
* get_form()：我们使用表单框架的`modelform_factory()`函数来构建一个动态的表单。由于我们将要给*Text*，*Video*，*Image*以及*File*模型构建一个表单，我们使用`exclude`参数去指定要从表单中排除的公共字段，并允许自动包含所有其他属性。通过做到这点，我们不必去知道依赖的模型中锁包含的字段。
* dispatch()：它检索以下URL参数并且存储相符的模块，模型以及内容对象作为类的属性：

    * module_id：The id for the module that the content is/will be associated with**(译者注：求比较好的翻译)**。
    * model_name：要创建或更新的内容的模型名。
    * id：这是将要更新的对象的id。在创建新对象的时候它会是`None`。

添加以下`get()`和`post()`方法给`ContentCreateUpdateView`：

```python
def get(self, request, module_id, model_name, id=None):
    form = self.get_form(self.model, instance=self.obj)
    return self.render_to_response({'form': form,
                                       'object': self.obj})
                                       
def post(self, request, module_id, model_name, id=None):
    form = self.get_form(self.model,
                            instance=self.obj,
                            data=request.POST,
                            files=request.FILES)
    if form.is_valid():
        obj = form.save(commit=False)
        obj.owner = request.user
        obj.save()
        if not id:
            # new content
            Content.objects.create(module=self.module,
        return redirect('module_content_list', self.module.id)
    return self.render_to_response({'form': form,
                                       'object': self.obj})
```

以上方法如下所示：

* get()：当收到一个GET请求的时候会被执行。我们构建模型表单给`Text`，`Video`，`Image`,以及`File`实例使用当它们被保存的时候。除此以外，我们不会传递实例给创建新的对象，因为`self.obj`在没有id提供的时候是`None`。
* post()：当收到一个POST请求的时候会被执行。我们构建模型表单会传递所有提交的数据和文件给该表单。之后我们验证该表单。如果这个表单验证通过，我们创建一个新的对象并且在保存该对象到数据库之前分配`request.user`作为该对象的拥有者。我们会检查`id`参数，如果没有提供`id`，我们就知道当前用户正在创建一个新的对象而不是更新一个已经存在的对象。如果这是一个新的对象，我们创建一个`Content`对象给给予的模块并且关联新的内容给该模块。

编辑*courses*应用的*urls.py*文件禀帖添加以下URL模式：

```python
url(r'^module/(?P<module_id>\d+)/content/(?P<model_name>\w+)/create/$',
    views.ContentCreateUpdateView.as_view(),
    name='module_content_create'),
url(r'^module/(?P<module_id>\d+)/content/(?P<model_name>\w+)/(?P<id>\d+)/$',
    views.ContentCreateUpdateView.as_view(),
    name='module_content_update'),
```

以上新的URL模式如下：

* module_content_create：用来创建新的文本，视频，图片或者文件对象并且给一个模块添加这些对象。它包含`module_id`和`model_name`参数。前者允许连接新的内容对象给给予的模块。后者指定构建表单使用的内容模型。
* module_content_update：用来更新一个已有的文本，视频，图片或者文件对象。它包含`module_id`和`model_name`参数，以及一个`id`参数来辨明那个需要被更新的内容。

在*courses/manage/*模板目录下创建新的目录命名为*content*。创建模板*courses/manage/content/form.html*并且添加以下代码：

```html
   {% extends "base.html" %}
   
   {% block title %}
     {% if object %}
       Edit content "{{ object.title }}"
     {% else %}
       Add a new content
     {% endif %}
   {% endblock %}
   
   {% block content %}
     <h1>
       {% if object %}
         Edit content "{{ object.title }}"
       {% else %}
         Add a new content
       {% endif %}
     </h1>
     <div class="module">
       <h2>Course info</h2>
       <form action="" method="post" enctype="multipart/form-data">
         {{ form.as_p }}
         {% csrf_token %}
         <p><input type="submit" value="Save content"></p>
       </form>
     </div>
   {% endblock %}
```

这个模板是给`ContentCreateUpdateView`视图使用的。在这个模板中，我们会检查是否有一个`object`变量在上下文环境中。如果`object`存在上下文环境中，我们知道我们正在更新一个已经存在的对象。如果没有，我们在创建一个新的对象。

我们在`<form>`HTML元素中包含`enctype="multipart/form-data"`，因为这个表单包含一个文件上传用来给`Field`和`Image`内容模型使用。

运行开发服务器。给存在的课程创建一个模块并且在你的浏览器中打开 http://127.0.0.1:8000/course/module/6/content/image/create/ 。如果有必要，在ULR中修改模块id。你将会看到以下表单用来创建新的`Image`对象：

![django-10-10](http://ohqrvqrlb.bkt.clouddn.com/django-10-10.png)

先不要提交表单。如果你想要尝试，它将会是失败的，因为我们还没有定义`module_content_list`的URL。我们一会儿就要去创建它。

我们还需要一个视图去删除内容。编辑*courses*应用的*views.py*文件，添加以下代码：

```python
class ContentDeleteView(View):
    def post(self, request, id):
        content = get_object_or_404(Content,
                            id=id,
                            module__course__owner=request.user)
        module = content.module
        content.item.delete()
        content.delete()
        return redirect('module_content_list', module.id)
```

`ContentDeleteView`通过给予的id检索`content`对象，它删除关联的*Text*，*Video*，*Image*以及*File*对象，并且在最后，它会删除`Content`对象并且重定向用户到`module_content_list` URL去排列其他模块的内容。

编辑*courses*应用的*urls.py*文件并且添加以下URL模式：

```python
url(r'^content/(?P<id>\d+)/delete/$',
    views.ContentDeleteView.as_view(),
    name='module_content_delete'),
```

现在，教师们可以方便的创建，更新以及删除内容。

##管理模块和内容

我们已经构建了用来创建，编辑以及删除课程模块和内容的视图。现在，我们需要一个视图去给一个课程显示所有的模块并且给一个指定的模块排列所有的内容。

编辑*courses*应用的*views.py*文件并且添加以下代码：

```python
class ModuleContentListView(TemplateResponseMixin, View):
    template_name = 'courses/manage/module/content_list.html'
    
    def get(self, request, module_id):
        module = get_object_or_404(Module,
                                      id=module_id,
                                      course__owner=request.user)
                                      
        return self.render_to_response({'module': module})
```

以上就是`ModuleContentListView`视图。这个视图通过给予的id拿到`Module`对象该对象是属于当前的用户并且通过给予的模块渲染一个模板。

编辑*courses*应用的*urls.py*文件，添加以下URL模式：

```python
url(r'^module/(?P<module_id>\d+)/$',
    views.ModuleContentListView.as_view(),
    name='module_content_list'),
```

在*templates/courses/manage/module/*目录下创建新的模板命名为*content_list.html*，添加以下代码：

```html
{% extends "base.html" %}
{% block title %}
     Module {{ module.order|add:1 }}: {{ module.title }}
{% endblock %}
{% block content %}
{% with course=module.course %}
  <h1>Course "{{ course.title }}"</h1>
  <div class="contents">
    <h3>Modules</h3>
    <ul id="modules">
      {% for m in course.modules.all %}
        <li data-id="{{ m.id }}" {% if m == module %}
        class="selected"{% endif %}>
          <a href="{% url "module_content_list" m.id %}">
            <span>
              Module <span class="order">{{ m.order|add:1 }}</span>
            </span>
            <br>
            {{ m.title }}
          </a> 
        </li>
      {% empty %}
        <li>No modules yet.</li>
      {% endfor %}
    </ul>
    <p><a href="{% url "course_module_update" course.id %}">Edit modules</a>
    </p>
  </div>
  <div class="module">
    <h2>Module {{ module.order|add:1 }}: {{ module.title }}</h2>
    <h3>Module contents:</h3>
    <div id="module-contents">
      {% for content in module.contents.all %}
        <div data-id="{{ content.id }}">
          {% with item=content.item %}
            <p>{{ item }}</p>
            <a href="#">Edit</a>
            <form action="{% url "module_content_delete" content.id %}" method="post">
              <input type="submit" value="Delete">
              {% csrf_token %}
            </form>
          {% endwith %}
        </div>
      {% empty %}
        <p>This module has no contents yet.</p>
      {% endfor %}
      </div>
       <hr>
       <h3>Add new content:</h3>
       <ul class="content-types">
         <li><a href="{% url "module_content_create" module.id "text" %}">Text</a></li>
         <li><a href="{% url "module_content_create" module.id "image" %}">Image</a></li>
         <li><a href="{% url "module_content_create" module.id "video" %}">Video</a></li>
         <li><a href="{% url "module_content_create" module.id "file" %}">File</a></li>
      </ul> 
    </div>
{% endwith %}
{% endblock %}   
```

这个模板展示一个课程所有的模块以及被选中的模块的内容。我们迭代课程模块并将它们展示在侧边栏。我们还迭代模块的内容并且通过`content.item`去获取关联的*Text*，*Video*，*Image*以及*File*对象。我们还包含可以创建新的文本，视频，图片以及文件内容的链接。

我们想要知道每个*item*对象是哪种类型（文本，视频，图片或者文件）的对象。我们需要模型名用来构建URL去编辑对象。除此以外，我们还在模板中展示各式各样的不同的*item*，基于*item*的内容类型。我们可以通过访问对象的`_meta`属性来从模型的`Meta`类中获取一个对象的模型。尽管如此，Django不允许访问开头是下划线的变量或者属性在模板中为了编辑检索私有属性或者调用到私有方法。我们可以通过编写一个定制模板过滤器来解决这个问题。

在*courses*应用目录下创建以下文件结构：

```shell
templatetags/
    __init__.py
    course.py
```

编辑*course.py*模块，添加以下代码：

```python
from django import template

register = template.Library()

@register.filter
def model_name(obj):
    try:
        return obj._meta.model_name
    except AttributeError:
        return None
```

以上就是`model_name`模板过滤器。我们可以在模板中通过`object|model_name`应用它来给一个对象获取模型的名字。

编辑*templates/courses/manage/module/content_list.html*模板，在`{% extends %}`模板标签下添加以下内容：

    {% load course %}

这样将会加载*course*模板标签。之后，将以下内容：

```html
<p>{{ item }}</p>
<a href="#">Edit</a>
```

替换成：

```html
<p>{{ item }} ({{ item|model_name }})</p>
<a href="{% url "module_content_update" module.id item|model_name item.id %}">Edit</a>
```

现在，我们在模板中展示item模型并且使用模型名曲构建编辑对象的链接。编辑*courses/manage/course/list.html*模板，添加一个链接给`module_content_list` URL，如下所示：

```html
<a href="{% url "course_module_update" course.id %}">Edit modules</a>
{% if course.modules.count > 0 %}
 <a href="{% url "module_content_list" course.modules.first.id %}">Manage contents</a>
{% endif %}
```

这个新链接允许用户去访问课程的第一个模块的内容，如果有好多内容的话。

打开 http://127.0.0.1:8000/course/mine/ 然后点击一个包含最新模块的课程的**Manage contents**链接。你会看到如下页面：

![django-10-11](http://ohqrvqrlb.bkt.clouddn.com/django-10-11.png)

当你点击左方侧边栏的一个模块上，它的内容会在主区域显示。这个模板还包含用来给展示的模块添加一个新的文本，视频，图片或者文件内容。给这个模块添加一堆不同的内容然后看下结果。这个内容将会出现在**Module contents**之后，如下所示：

![django-10-12](http://ohqrvqrlb.bkt.clouddn.com/django-10-12.png)

##重新整理模块和内容

我们需要提供一个简单的放来去重新排序课程模板和它们的内容。我们将要使用一个JavaScript drag-n-drop 控件去让我们的用户通过拖拽课程的模块来对课程模块进行重新排序。当用户完成拖拽一个模块，我们将会执行一个异步请求（AJAX）去存储新的模块顺序。

我们需要一个视图，该视图通过编译在JSON中的模块的id来检索新的对象。编辑*courses*应用的*views.py*文件，添加以下代码：

```python
from braces.views import CsrfExemptMixin, JsonRequestResponseMixin

class ModuleOrderView(CsrfExemptMixin,
                         JsonRequestResponseMixin,
                         View):
    
    def post(self, request):
        for id, order in self.request_json.items():
            Module.objects.filter(id=id,
                course__owner=request.user).update(order=order)
        return self.render_json_response({'saved': 'OK'})
```

以上是`ModuleOrderView`。我们使用以下django-braces的mixins：

* csrfExemptMixin：用来避免在POST请求中检查一个CSRF token。
* JsonRequestResponseMixin：将请求的数据分析为JSON并且将相应也序列化成JSON并且返回一个`application/json`内容类型的HTTP响应。

我们可以构建一个类似的视图去排序一个模块的内容。添加以下代码到*views.py*中：

```python
class ContentOrderView(CsrfExemptMixin,
                          JsonRequestResponseMixin,
                          View):
    def post(self, request):
        for id, order in self.request_json.items():
            Content.objects.filter(id=id,
                          module__course__owner=request.user) \
                          .update(order=order)
        return self.render_json_response({'saved': 'OK'})
```

现在，编辑*courses*应用的*urls.py*文件，添加以下URL模式：

```python
url(r'^module/order/$',
       views.ModuleOrderView.as_view(),
       name='module_order'),
url(r'^content/order/$',
       views.ContentOrderView.as_view(),
       name='content_order'),
```

最后，我们在模板中导入drag-n-drop功能。我们将要使用jQuery UI库来使用这个功能。jQuery UI基于jQuery构建并且它提供了一组界面交互，效果和小部件。我们将要使用它的*sortable*元素。首先，我们需要在基础模板中加载jQuery UI。打开*courses*应用下的*templates/*目录下的*base.html*文件，在加载jQuery的下方添加jQuery UI脚本，如下所示：

```html
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/jqueryui/1.11.4/jquery-ui.min.js"></script>
```

**（译者注：要用以上地址，记得翻墙。。。。。或者自己直接下载）**

我们在jQuery框架下加载jQuery UI。现在，编辑*courses/manage/module/content_list.html*模板添加以下代码在模板的底部：

```html
{% block domready %}
   $('#modules').sortable({
       stop: function(event, ui) {
           modules_order = {};
           $('#modules').children().each(function(){
               // update the order field
               $(this).find('.order').text($(this).index() + 1);
               // associate the module's id with its order
               modules_order[$(this).data('id')] = $(this).index();
               });
               $.ajax({
                type: 'POST',
                url: '{% url "module_order" %}',
                contentType: 'application/json; charset=utf-8',
                dataType: 'json',
                data: JSON.stringify(modules_order)
                });
    }
});

$('#module-contents').sortable({
    stop: function(event, ui) {
        contents_order = {};
        $('#module-contents').children().each(function(){
            // associate the module's id with its order
            contents_order[$(this).data('id')] = $(this).index();
           });
        $.ajax({
               type: 'POST',
               url: '{% url "content_order" %}',
               contentType: 'application/json; charset=utf-8',
               dataType: 'json',
               data: JSON.stringify(contents_order),
        }); 
      }
   });
{% endblock %}  
```

这个JavaScripy代码在`{% block domready %}`区块中，因此它会被包含在我们之前定义在*base.html*模板中的jQuery的`$(document).ready()`事件中。这将保证我们的JavaScripy代码会在页面每次加载的时候都会被执行一次。我们给列在侧边栏的模块定义了一个*sortable*元素并且给模块内容列也定义了一个不同的。这两者有着相似的方式。在以上代码中，我们执行以下任务：

* 1 首先，我们给*modules* HTML元素定义了一个*sortable*元素。请记住，我们使用`#moudles`，因为jQuery给选择器使用CSS符号。
* 2 我们给*stop*事件指定一个函数。这个时间会在用户每次储存一个元素的时候被触发。
* 3 我们创建一个空的*modules_orders*目录。给这个目录的键将会是模块的id，并且给每个模块的值都会被分配次序。
* 4 我们迭代`#module`子元素。我们给每个模块重新计算展示次序并且拿到每个模块的*data-id*属性，该属性包含了模块的id。我们给*modules_order*目录添加id作为一个键并且模型的新的索引作为值。
* 5 我们运行一个AJAX POST请求给*content_order* URL，在请求中包含*modules_orders*的序列化的JSON数据。相应的`ModuleOrderView`会负责更新模块的顺序。

*sortable*元素排列内容非常类似与上者的方法。回到你的浏览器然后重载页面。现在你将可以点击并且拖动模块和内容，去重新排序它们如下所示：

![django-10-13](http://ohqrvqrlb.bkt.clouddn.com/django-10-13.png)

很好！现在你可以重新排序课程模块和模块内容了。

##总结

在这章中，你学习了如何创建一个多功能的内容管理系统。你使用了模型继承以及创建了一个定制模型字段。你还通过基于类的视图和mixins工作。你创建了formsets以及一个系统去管理不同类型的内容。

在下一章，你将会创建一个学生注册系统。你还将熏染不同类型的内容，并且你还会学习如何使用Django的缓存框架。

##译者总结

四个字：本章真长。

