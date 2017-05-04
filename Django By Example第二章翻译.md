书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**2016年12月13日发布（3天完成第二章的翻译，但没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月17日校对完成（不是精校，希望大家多指出需要修改的地方）**

**2017年3月6日精校完成（感谢大牛 @kukoo 的精校！）**

**2017年3月21日再度精校（感谢大牛 @妈妈不在家 的精校！初版我已经不敢再看！）**

（译者注：翻译完第一章后，发现翻译第二章的速度上升了不少，难道这就是传说中的经验值提升了？）

#第二章
##用高级特性来增强你的blog

 在上一章中，你创建了一个基础的博客应用。现在你将利用一些高级的特性例如通过email来分享帖子，添加评论，给帖子打上tag，检索出相似的帖子等将它改造成为一个功能更加齐全的博客。在本章中，你将会学习以下几点：

* 通过Django发送email
* 在视图（views）中创建并操作表单
* 通过模型（models）创建表单
* 集成第三方应用
* 构建复杂的查询集（QuerySets)

###通过email分享帖子

首先，我们会允许用户通过发送邮件来分享他们的帖子。让我们花费一小会时间来想下，根据在上一章中学到的知识，你该如何使用views，urls和templates来创建这个功能。现在，核对一下你需要哪些才能允许你的用户通过邮件来发送帖子。你需要做到以下几点：

* 给用户创建一个表单来填写他们的姓名，email，收件人以及评论，评论不是必选项。
* 在*views.py*文件中创建一个视图（view）来操作发布的数据和发送email
* 在blog应用的*urls.py*中为新的视图（view）添加一个URL模式
* 创建一个模板（template）来展示这个表单

###使用Django创建表单

让我们开始创建一个表单来分享帖子。Django有一个内置的表单框架允许你通过简单的方式来创建表单。这个表单框架允许你定义你的表单字段，指定这些字段必须展示的方式，以及指定这些字段如何验证输入的数据。Django表单框架还提供了一种灵活的方式来渲染表单以及操作数据。

Django提供了两个可以创建表单的基本类：

* Form: 允许你创建一个标准表单
* ModelForm: 允许你创建一个可用于创建或者更新model实例的表单

首先，在你blog应用的目录下创建一个*forms.py*文件，输入以下代码：

```python
from django import forms

class EmailPostForm(forms.Form):
    name = forms.CharField(max_length=25)
    email = forms.EmailField()
    to = forms.EmailField()
    comments = forms.CharField(required=False,
									     widget=forms.Textarea)
```

这是你的第一个Django表单。看下代码：我们已经创建了一个继承了基础*Form*类的表单。我们使用不同的字段类型以使Django有依据的来验证字段。

> 表单可以存在你的Django项目的任何地方，但按照惯例将它们放在每一个应用下面的*forms.py*文件中

*name*字段是一个*CharField*。这种类型的字段被渲染成`<input type=“text”>`HTML元素。每种字段类型都有默认的控件来确定它在HTML中的展示形式。通过改变控件的属性可以重写默认的控件。在comment字段中，我们使用`<textarea></textarea>`HTML元素而不是使用默认的`<input>`元素来显示它。

字段验证取决于字段类型。例如，*email*和*to*字段是*EmailField*,这两个字段都需要一个有效的email地址，否则字段验证将会抛出一个*forms.ValidationError*异常导致表单验证不通过。在表单验证的时候其他的参数也会被考虑进来：我们将name字段定义为一个最大长度为25的字符串；通过设置`required=False`让comments的字段可选。所有这些也会被考虑到字段验证中去。目前我们在表单中使用的这些字段类型只是Django支持的表单字段的一部分。要查看更多可利用的表单字段，你可以访问：https://docs.djangoproject.com/en/1.8/ref/forms/fields/

###在视图（views）中操作表单

当表单成功提交后你必须创建一个新的视图（views）来操作表单和发送email。编辑blog应用下的*views.py*文件，添加以下代码：

```python
from .forms import EmailPostForm

def post_share(request, post_id):
    # retrieve post by id
    post = get_object_or_404(Post, id=post_id, status='published')
    
    if request.method == 'POST':
        # Form was submitted
        form = EmailPostForm(request.POST)
        if form.is_valid():
            # Form fields passed validation
            cd = form.cleaned_data
            # ... send email
    else:
        form = EmailPostform()
    return render(request, 'blog/post/share.html', {'post': post,
										                       'form: form})
```

该视图（view）完成了以下工作：

* 我们定义了*post_share*视图，参数为*request*对象和*post_id*。
* 我们使用*get_object_or_404*快捷方法通过ID获取对应的帖子，并且确保获取的帖子有一个*published*状态。
* 我们使用同一个视图（view）来展示初始表单和处理提交后的数据。我们会区别被提交的表单和不基于这次请求方法的表单。我们将使用*POST*来提交表单。如果我们得到一个*GET*请求，一个空的表单必须显示，而如果我们得到一个*POST*请求，则表单需要提交和处理。因此，我们使用`request.method == 'POST'`来区分这两种场景。

下面是展示和操作表单的过程：

* 1.通过GET请求视图（view）被初始加载后，我们创建一个新的表单实例，用来在模板（template）中显示一个空的表单：
    
        form = EmailPostForm()
    
* 2.当用户填写好了表单并通过*POST*提交表单。之后，我们会用保存在`request.POST`中提交的数据创建一个表单实例。
        
        if request.method == 'POST':
           # Form was submitted
           form = EmailPostForm(request.POST)
           
* 3.在以上步骤之后，我们使用表单的*is_valid()*方法来验证提交的数据。这个方法会验证表单引进的数据，如果所有的字段都是有效数据，将会返回*True*。一旦有任何一个字段是无效的数据，*is_valid()*就会返回*False*。你可以通过访问 `form.errors`来查看所有验证错误的列表。
* 4如果表单数据验证没有通过，我们会再次使用提交的数据在模板（template）中渲染表单。我们会在模板（template）中显示验证错误的提示。
* 5.如果表单数据验证通过，我们通过访问`form.cleaned_data`获取验证过的数据。这个属性是一个表单字段和值的字典。

> 如果你的表单数据没有通过验证，*cleaned_data*只会包含验证通过的字段

现在，你需要学习如何使用Django来发送email,把所有的事情结合起来。

##使用Django发送email

使用Django发送email非常简单。首先，你需要有一个本地的SMTP服务或者通过在你项目的settings.py文件中添加以下设置去定义一个外部SMTP服务器的配置：

* EMAIL_HOST: SMTP服务地址。默认本地。
* EMAIL_POSR: SMATP服务端口，默认25。
* EMAIL_HOST_USER: SMTP服务的用户名。
* EMAIL_HOST_PASSWORD: SMTP服务的密码。
* EMAIL_USE_TLS: 是否使用TLS加密连接。
* EMAIL_USE_SSL: 是否使用隐式的SSL加密连接。

如果你没有本地SMTP服务，你可以使用你的email服务供应商提供的SMTP服务。下面提供了一个简单的例子展示如何通过使用Google账户的Gmail服务来发送email：

```python
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = 'your_account@gmail.com'
EMAIL_HOST_PASSWORD = 'your_password'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
```
    
运行命令python manage.py shell来打开Python shell，发送一封email如下所示：

```shell
>>> from django.core.mail import send_mail
>>> send_mail('Django mail', 'This e-mail was sent with Django.','your_account@gmail.com', ['your_account@gmail.com'], fail_silently=False)
```
    
*send_mail()*方法需要这些参数：邮件主题，内容，发送人以及一个收件人的列表。通过设置可选参数`fail_silently=False`，我们告诉这个方法如果email没有发送成功那么需要抛出一个异常。如果你看到输出是*1*，证明你的email发送成功了。如果你使用之前的配置用Gmail来发送邮件，你可能需要去 https://www.google.com/settings/security/lesssecureapps 去开通一下低安全级别应用的权限。(译者注：练习时老老实实用QQ邮箱吧）

现在，我们要将以上代码添加到我们的视图（view）中。在blog应用下的*views.py*文件中编辑*post_share*视图（view）如下所示：

```python
from django.core.mail import send_mail

def post_share(request, post_id):
    # Retrieve post by id
    post = get_object_or_404(Post, id=post_id, status='published')
    sent = False
    if request.method == 'POST':
        # Form was submitted
        form = EmailPostForm(request.POST)
        if form.is_valid():
            # Form fields passed validation
            cd = form.cleaned_data
            post_url = request.build_absolute_url(
                                    post.get_absolute_url())
            subject = '{} ({}) recommends you reading "{}"'.format(cd['name'], cd['email'], post.title)
            message = 'Read "{}" at {}\n\n{}\'s comments: {}'.format(post.title, post_url, cd['name'], cd['comments'])
            send_mail(subject, message, 'admin@myblog.com',[cd['to']])
            sent = True
    else:
        form = EmailPostForm()
        
    return render(request, 'blog/post/share.html', {'post': post,
                                                    'form': form,
                                                    'sent': sent})
```

请注意，我们声明了一个*sent*变量并且当帖子被成功发送时赋予它*True*。当表单成功提交的时候，我们之后将在模板（template）中使用这个变量显示一条成功提示。由于我们需要在email中包含帖子的超链接，所以我们通过使用`post.get_absolute_url()`方法来获取到帖子的绝对路径。我们将这个绝对路径作为`request.build_absolute_uri()`的输入值来构建一个完整的包含了HTTP schema和主机名的url。我们通过使用验证过的表单数据来构建email的主题和消息内容并最终给表单to字段中包含的所有email地址发送email。

现在你的视图（view）已经完成了，别忘记为它去添加一个新的URL模式。打开你的blog应用下的*urls.py*文件添加*post_share*的URL模式如下所示：

```python
urlpatterns = [
# ...
url(r'^(?P<post_id>\d+)/share/$', views.post_share,
    name='post_share'),
]
```

##在模板（templates）中渲染表单

在通过创建表单，编写视图（view）以及添加URL模式后，我们就只剩下为这个视图（view）添加模板（tempalte）了。在`blog/templates/blog/post/`目录下创建一个新的文件并命名为*share.html*。在该文件中添加如下代码：

```html
{% extends "blog/base.html" %}

{% block title %}Share a post{% endblock %}

{% block content %}
  {% if sent %}
    <h1>E-mail successfully sent</h1>
    <p>
      "{{ post.title }}" was successfully sent to {{ cd.to }}.
    </p>
  {% else %}
    <h1>Share "{{ post.title }}" by e-mail</h1>
    <form action="." method="post">
      {{ form.as_p }}
      {% csrf_token %}
      <input type="submit" value="Send e-mail">
    </form>
  {% endif %}
{% endblock %}
```

这个模板（tempalte）专门用来显示一个表单或一条成功提示信息。如你所见，我们创建的HTML表单元素里面表明了它必须通过POST方法提交：

    <form action="." method="post">

接下来我们要包含真实的表单实例。我们告诉Django用`as_p`方法利用HTML的`<p>`元素来渲染它的字段。我们也可以使用`as_ul`利用无序列表来渲染表单或者使用`as_table`利用HTML表格来渲染。如果我们想要逐一渲染每一个字段，我们可以迭代字段。例如下方的例子：

```html
{% for field in form %}
  <div>
    {{ field.errors }}
    {{ field.label_tag }} {{ field }}
  </div>
{% endfor %}
```
    
`{% csrf_token %}`模板（tempalte）标签（tag）引进了可以避开*Cross-Site request forgery(CSRF)*攻击的自动生成的令牌，这是一个隐藏的字段。这些攻击由恶意的站点或者可以在你的站点中为用户执行恶意行为的程序组成。通过访问 https://en.wikipedia.org/wiki/Cross-site_request_forgery你可以找到更多的信息 。

上述的标签（tag）生成的隐藏字段就像下面一样：

```html
<input type='hidden' name='csrfmiddlewaretoken' value='26JjKo2lcEtYkGoV9z4XmJIEHLXN5LDR' />
``` 
    
> 默认情况下，Django在所有的POST请求中都会检查CSRF标记（token）。请记住要在所有使用POST方法提交的表单中包含*csrf_token*标签。**（译者注：当然你也可以关闭这个检查，注释掉app_list中的csrf应用即可，我就是这么做的，因为我懒）**


编辑你的*blog/post/detail.html*模板（template），在`{{ post.body|linebreaks }}`变量后面添加如下的链接来分享帖子的URL：

```html
<p>
  <a href="{% url "blog:post_share" post.id %}">
    Share this post
  </a>
</p>
```
    
请记住，我们通过使用Django提供的`{% url %}`模板（template）标签（tag）来动态的生成URL。我们以*blog*为命名空间，以*post_share*为URL，同时传递帖子ID作为参数来构建绝对的URL。

现在，通过`python manage.py runserver`命令来启动开发服务器，在浏览器中打开 http://127.0.0.1:8000/blog/ 。点击任意一个帖子标题查看详情页面。在帖子内容的下方，你会看到我们刚刚添加的链接，如下所示：
![django-2-1](http://ohqrvqrlb.bkt.clouddn.com/django-2-1.png)

点击**Share this post**,你会看到包含通过email分享帖子的表单的页面。看上去如下所示：
![django-2-2](http://ohqrvqrlb.bkt.clouddn.com/django-2-2.png)

这个表单的CSS样式被包含在示例代码中的 static/css/blog.css文件中。当你点击**Send e-mail**按钮，这个表单会提交并验证。如果所有的字段都通过了验证，你会得到一条成功信息如下所示：
![django-2-3](http://ohqrvqrlb.bkt.clouddn.com/django-2-3.png)

如果你输入了无效数据，你会看到表单被再次渲染，并展示出验证错误信息，如下所示：
![django-2-4](http://ohqrvqrlb.bkt.clouddn.com/django-2-4.png)

##创建一个评论系统

现在我们准备为blog创建一个评论系统，这样用户可以在帖子上进行评论。需要做到以下几点来创建一个评论系统：

* 创建一个模型（model）用来保存评论
* 创建一个表单用来提交评论并且验证输入的数据
* 添加一个视图（view）来处理表单和保存新的评论到数据库中
* 编辑帖子详情模板（template）来展示评论列表以及用来添加新评论的表单

首先，让我们创建一个模型（model）来存储评论。打开你的blog应用下的*models.py*文件添加如下代码：

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, related_name='comments')
    name = models.CharField(max_length=80)
    email = models.EmailField()
    body = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    active = models.BooleanField(default=True)
    
    class Meta:
        ordering = ('created',)
        
    def __str__(self):
        return 'Comment by {} on {}'.format(self.name, self.post)
```

以上就是我们的*Comment*模型（model）。它包含了一个外键将一个单独的帖子和评论关联起来。在*Comment*模型（model）中定义多对一（many-to-one）的关系是因为每一条评论只能在一个帖子下生成，而每一个帖子又可能包含多个评论。*related_name*属性允许我们给这个属性命名，这样我们就可以利用这个关系从相关联的对象反向定位到这个对象。定义好这个之后，我们通过使用 `comment.post`就可以从一条评论来取到对应的帖子，以及通过使用`post.comments.all()`来取回一个帖子所有的评论。如果你没有定义*related_name*属性，Django会使用这个模型（model）的名称加上*_set*（在这里是：comment_set）来命名从相关联的对象反向定位到这个对象的manager。

访问https://docs.djangoproject.com/en/1.8/topics/db/examples/many_to_one/, 你可以学习更多关于多对一的关系。

我们用一个*active*布尔字段用来手动禁用那些不合适的评论。默认情况下，我们根据*created*字段，对评论按时间顺序进行排序。

你刚创建的这个新的*Comment*模型（model）并没有同步到数据库中。运行以下命令生成一个新的反映了新模型（model）创建的数据迁移：

    python manage.py makemigrations blog

你会看到如下输出：

```shell
Migrations for 'blog':
  0002_comment.py:
    - Create model Comment
```

Django在blog应用下的*migrations/*目录中生成了一个*0002_comment.py*文件。现在你需要创建一个关联数据库模式并且将这些改变应用到数据库中。运行以下命令来执行已经存在的数据迁移：

    python manage.py migrate

你会获取以下输出：

    Applying blog.0002_comment... OK
    
我们刚刚创建的数据迁移已经被执行，现在一张*blog_comment*表已经存在数据库中。


现在，我们可以添加我们新的模型（model）到管理站点中并通过简单的接口来管理评论。打开blog应用下的*admin.py*文件，添加comment model的导入,添加如下内容：

```python
from .models import Post, Comment

class CommentAdmin(admin.ModelAdmin):
    list_display = ('name', 'email', 'post', 'created', 'active')
    list_filter = ('active', 'created', 'updated')
    search_fields = ('name', 'email', 'body')
admin.site.register(Comment, CommentAdmin)
```
    
运行命令`python manage.py runserver`来启动开发服务器然后在浏览器中打开 http://127.0.0.1:8000/admin/ 。你会看到新的模型（model）在**Blog**区域中出现，如下所示：
![django-2-5](http://ohqrvqrlb.bkt.clouddn.com/django-2-5.png)

我们的模型（model）现在已经被注册到了管理站点，这样我们就可以使用简单的接口来管理评论实例。

##通过模型（models）创建表单

我们仍然需要构建一个表单让我们的用户在blog帖子下进行评论。请记住，Django有两个用来创建表单的基础类：*Form*和*ModelForm*。你先前已经使用过第一个让用户通过email来分享帖子。在当前的例子中，你将需要使用*ModelForm*，因为你必须通过你的*Comment*模型（model）动态的创建表单。编辑blog应用下的*forms.py*，添加如下代码：

```python
from .models import Comment

class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ('name', 'email', 'body')
```
    
根据模型（model）创建表单，我们只需要在这个表单的*Meta*类里表明使用哪个模型（model）来构建表单。Django将会解析model并为我们动态的创建表单。每一种模型（model）字段类型都有对应的默认表单字段类型。表单验证时会考虑到我们定义模型（model）字段的方式。Django为模型（model）中包含的每个字段都创建了表单字段。然而，使用*fields* 列表你可以明确的告诉框架你想在你的表单中包含哪些字段，或者使用*exclude* 列表定义你想排除在外的那些字段。对于我们的*CommentForm*来说,我们在表单中只需要*name*,*email*,和*body*字段，因为我们只需要用到这3个字段让我们的用户来填写。

##在视图（views）中操作*ModelForms*

为了能更简单的处理它，我们会使用帖子的详情视图（view）来实例化表单。编辑*views.py*文件**(注: 原文此处有错，应为 `views.py`)**，导入*Comment*模型（model）和*CommentForm*表单，并且修改*post_detail*视图（view）如下所示：

```python
from .models import Post, Comment
from .forms import EmailPostForm, CommentForm

def post_detail(request, year, month, day, post):
    post = get_object_or_404(Post, slug=post,
                                   status='published',
                                   publish__year=year,
                                   publish__month=month,
                                   publish__day=day)
    # List of active comments for this post
    comments = post.comments.filter(active=True)
    new_comment = None
        
    if request.method == 'POST':
        # A comment was posted
        comment_form = CommentForm(data=request.POST)
        if comment_form.is_valid():
            # Create Comment object but don't save to database yet
            new_comment = comment_form.save(commit=False)
            # Assign the current post to the comment
            new_comment.post = post
            # Save the comment to the database
            new_comment.save()
    else:
        comment_form = CommentForm()
    return render(request,
                  'blog/post/detail.html',
                  {'post': post,
                  'comments': comments, 
                  'new_comment': new_comment,
                  'comment_form': comment_form})
```

让我们来回顾一下我们刚才对视图（view）添加了哪些操作。我们使用*post_detail*视图（view）来显示帖子和该帖子的评论。我们添加了一个查询集（QuerySet）来获取这个帖子所有有效的评论：
    
    comments = post.comments.filter(active=True)
    
我们从*post*对象开始构建这个查询集（QuerySet）。我们使用关联对象的manager，这个manager是我们在*Comment* 模型（model）中使用*related_name*关系属性为*comments*定义的。
我们还在这个视图（view）中让我们的用户添加一条新的评论。因此，如果这个视图（view）是通过GET请求被加载的，那么我们用`comment_fomr = commentForm()`来创建一个表单实例。如果是通过POST请求，我们使用提交的数据并且用*is_valid()*方法验证这些数据去实例化表单。如果这个表单是无效的，我们会用验证错误信息渲染模板（template）。如果表单通过验证，我们会做以下的操作：

* 1.我们通过调用这个表单的*save()*方法创建一个新的*Comment*对象，如下所示：

`new_comment = comment_form.save(commit=False)` 

*Save()*方法创建了一个表单链接的model的实例，并将它保存到数据库中。如果你调用这个方法时设置`comment=False`，你创建的模型（model）实例不会即时保存到数据库中。当你想在最终保存之前修改这个model对象会非常方便，我们接下来将做这一步骤。*save()*方法是给*ModelForm*用的，而不是给*Form*实例用的，因为*Form*实例没有关联上任何模型（model）。

* 2.我们为我们刚创建的评论分配一个帖子：
    
        new_comment.post = post
    
    通过以上动作，我们指定新的评论是属于这篇给定的帖子。

* 3.最后，我们用下面的代码将新的评论保存到数据库中：
    
        new_comment.save()
        
我们的视图（view）已经准备好显示和处理新的评论了。

##在帖子详情模板（template）中添加评论

我们为帖子创建了一个管理评论的功能。现在我们需要修改我们的*post_detail.html*模板（template）来适应这个功能，通过做到以下步骤：

* 显示这篇帖子的评论总数
* 显示评论的列表
* 显示一个表单给用户来添加新的评论

首先，我们来添加评论的总数。打开*views_detail.html* **（译者注：根据官网最新更正修改，原文是blog_detail.html）**模板（template）在*content*区块中添加如下代码：

```html
{% with comments.count as total_comments %}
  <h2>
    {{ total_comments }} comment{{ total_comments|pluralize }}
  </h2>
{% endwith %}
```    

在模板（template）中我们使用Django ORM执行`comments.count()` 查询集（QuerySet）。注意，Django模板（template）语言中不使用圆括号来调用方法。`{% with %}` 标签（tag）允许我们分配一个值给新的变量，这个变量可以一直使用直到遇到`{% endwith %}`标签（tag）。

> `{% with %}`模板（template）标签（tag）是非常有用的，可以避免直接操作数据库或避免多次调用花费较多的方法。

根据*total_comments*的值，我们使用*pluralize *模板（template）过滤器（filter）为单词*comment*显示复数后缀。模板（Template）过滤器（filters）获取到他们输入的变量值，返回计算后的值。我们将会在*第三章 扩展你的博客应用*中讨论更多的模板过滤器（tempalte filters）。

*pluralize*模板（template）过滤器（filter）在值不为1时，会在值的末尾显示一个"s"。之前的文本将会被渲染成类似：*0 comments*, *1 comment* 或者 *N comments*。Django内置大量的模板（template）标签（tags）和过滤器（filters）来帮助你以你想要的方式来显示信息。

现在，让我们加入评论列表。在模板（template）中之前的代码后面加入以下内容：
    
```html
{% for comment in comments %}
  <div class="comment">
    <p class="info">
      Comment {{ forloop.counter }} by {{ comment.name }}
      {{ comment.created }}
    </p>
    {{ comment.body|linebreaks }}
  </div>
{% empty %}
  <p>There are no comments yet.</p>
{% endfor %}
```
    
我们使用`{% for %}`模板（template）标签（tag）来循环所有的评论。如果*comments*列为空我们会显示一个默认的信息，告诉我们的用户这篇帖子还没有任何评论。我们使用 `{{ forloop.counter }}`变量来枚举所有的评论，在每次迭代中该变量都包含循环计数。之后我们显示发送评论的用户名，日期，和评论的内容。

最后，当表单提交成功后,你需要渲染表单或者显示一条成功的信息来代替之前的内容。在之前的代码后面添加如下内容：

```html
{% if new_comment %}
  <h2>Your comment has been added.</h2>
{% else %}
  <h2>Add a new comment</h2>
  <form action="." method="post">
    {{ comment_form.as_p }}
    {% csrf_token %}
    <p><input type="submit" value="Add comment"></p>
  </form>
{% endif %}
```
    
这段代码非常简洁明了：如果*new_comment*对象存在，我们会展示一条成功信息因为成功创建了一条新评论。否则，我们用段落`<p>`元素渲染表单中每一个字段，并且包含*POST*请求需要的*CSRF*令牌。在浏览器中打开 http://127.0.0.1:8000/blog/ 然后点击任意一篇帖子的标题查看它的详情页面。你会看到如下页面展示：

![django-2-6](http://ohqrvqrlb.bkt.clouddn.com/django-2-6.png)

使用该表单添加数条评论。这些评论会在你的帖子下面根据时间排序来展示，类似下图：
![django-2-7](http://ohqrvqrlb.bkt.clouddn.com/django-2-7.png)

在你的浏览器中打开 http://127.0.0.1:8000/admin/blog/comment/ 。你会在管理页面中看到你创建的评论列表。点击其中一个进行编辑，取消选择**Active**复选框，然后点击**Save**按钮。你会再次被重定向到评论列表页面，刚才编辑的评论**Save**列将会显示成一个没有激活的图标。类似下图的第一个评论：
![django-2-8](http://ohqrvqrlb.bkt.clouddn.com/django-2-8.png)

##增加标签（tagging）功能

在实现了我们的评论系统之后，我们准备创创建一个方法来给我们的帖子添加标签。我们将通过在我们的项目中集成第三方的Django标签应用来完成这个功能。*django-taggit*是一个可复用的应用，它会提供给你一个*Tag*模型（model）和一个管理器（manager）来方便的给任何模型（model）添加标签。你可以在 https://github.com/alex/django-taggit 看到它的源码。

首先，你需要通过pip安装django-taggit，运行以下命令：
    
    pip install django-taggit==0.17.1**（译者注：根据@孤独狂饮 验证，直接 `pip install django-taggit` 安装最新版即可，原作者提供的版本过旧会有问题，感谢@孤独狂饮）**
    
之后打开*mysite*项目下的*settings.py*文件，在*INSTALLED_APPS*设置中设置如下：

```python    
INSTALLED_APPS = (
    # ...
    'blog',
    'taggit',
)
```
    
打开你的blog应用下的*model.py*文件，给*Post*模型（model）添加*django-taggit*提供的*TaggableManager*管理器（manager），使用如下代码：

```python    
from taggit.managers import TaggableManager
class Post(models.Model):
    # ...
    tags = TaggableManager()
```
        
这个*tags*管理器（manager）允许你给*Post*对象添加，获取以及移除标签。

运行以下命令为你的模型（model）改变创建一个数据库迁移：

    python manage.py makemigrations blog

你会看到如下输出：

```shell    
Migrations for 'blog':
  0003_post_tags.py:
    - Add field tags to post
```
现在，运行以下代码在数据库中生成*django-taggit*模型（model）对应的表以及同步你的模型（model）的改变：

    python manage.py migrate

你会看到以下输出,：

```shell
Applying taggit.0001_initial... OK
Applying taggit.0002_auto_20150616_2121... OK
Applying blog.0003_post_tags... OK
```
    
你的数据库现在已经可以使用*django-taggit*模型（model）。打开终端运行命令` python manage.py shell`来学习如何使用*tags*管理器（manager）。首先，我们取回我们的其中一篇帖子（该帖子的ID为1）：
  
```shell  
>>> from blog.models import Post
>>> post = Post.objects.get(id=1)
```

之后为它添加一些标签并且取回它的标签来检查标签是否添加成功：

```shell
>>> post.tags.add('music', 'jazz', 'django')
>>> post.tags.all()
[<Tag: jazz>, <Tag: django>, <Tag: music>]
```
    
最后，移除一个标签并且再次检查标签列表：

```shell
>>> post.tags.remove('django')
>>> post.tags.all()
[<Tag: jazz>, <Tag: music>]
```
    
非常简单，对吧？运行命令`python manage.py runserver`启动开发服务器，在浏览器中打开 http://127.0.0.1:8000/admin/taggit/tag/ 。你会看到管理页面包含了*taggit*应用的*Tag*对象列表：

![django-2-9](http://ohqrvqrlb.bkt.clouddn.com/django-2-9.png)

转到 http://127.0.0.1:8000/admin/blog/post/ 并点击一篇帖子进行编辑。你会看到帖子中包含了一个新的**Tags**字段如下所示，你可以非常容易的编辑它：
![django-2-10](http://ohqrvqrlb.bkt.clouddn.com/django-2-10.png)

现在，我们准备编辑我们的blog帖子来显示这些标签。打开*blog/post/list.html* 模板（template）在帖子标题下方添加如下HTML代码：

    <p class="tags">Tags: {{ post.tags.all|join:", " }}</p>

*join*模板（template）过滤器（filter）的功能类似python字符串的join()方法，将给定的字符串连接起来。在浏览器中打开 http://127.0.0.1:8000/blog/ 。 你会看到每一个帖子的标题下面的标签列表：

![django-2-11](http://ohqrvqrlb.bkt.clouddn.com/django-2-11.png)

现在，让我们来编辑我们的*post_list*视图（view）让用户可以列出打上了特定标签的所有帖子。打开blog应用下的*views.py*文件，从*django-taggit*中导入*Tag*模型（model），然后修改*post_list*视图（view）让它可以通过标签选择性的过滤，如下所示：

```python
from taggit.models import Tag

def post_list(request, tag_slug=None): 
    object_list = Post.published.all() 
    tag = None
    
    if tag_slug:
        tag = get_object_or_404(Tag, slug=tag_slug) 
        object_list =   object_list.filter(tags__in=[tag]) 
        # ...
```
            
这个视图（view）做了以下工作：

* 1.视图（view）带有一个可选的*tag_slug*参数，默认是一个*None*值。这个参数会带进URL中。
* 2.视图（view）的内部，我们构建了初始的查询集（QuerySet），取回所有发布状态的帖子，假如给予一个标签 slug，我们通过`get_object_or_404()`用给定的slug来获取标签对象。
* 3.之后我们过滤所有帖子只留下包含给定标签的帖子。因为有一个多对多（many-to-many）的关系，我们必须通过给定的标签列表来过滤，在我们的例子中标签列表只包含一个元素。

要记住查询集（QuerySets)是惰性的。这个查询集（QuerySets)只有当我们在模板（template）中循环渲染帖子列表时才会被执行。

最后，修改视图（view）最底部的*render()*函数来，传递*tag*变量给模板（template）。这个视图（view）完成后如下所示：

```python
def post_list(request, tag_slug=None):
   object_list = Post.published.all()
   tag = None
   
   if tag_slug:
       tag = get_object_or_404(Tag, slug=tag_slug)
       object_list = object_list.filter(tags__in=[tag])
       
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
   return render(request, 'blog/post/list.html', {'page': page,
                                                  'posts': posts,
                                                  'tag': tag})
```
        
打开blog应用下的*url.py*文件，注释基于类的*PostListView* URL模式，然后取消*post_list*视图（view）的注释，如下所示：

```python
url(r'^$', views.post_list, name='post_list'),
# url(r'^$', views.PostListView.as_view(), name='post_list'),
```
    
添加下面额外的URL pattern到通过标签过滤过的帖子列表中：

```python
url(r'^tag/(?P<tag_slug>[-\w]+)/$',views.post_list,
    name='post_list_by_tag'),
```
    
如你所见，两个模式都指向了相同的视图（view），但是我们可以给它们不同的命名。第一个模式会调用*post_list*视图（view）并且不带上任何可选参数。然而第二个模式会调用这个视图（view）带上*tag_slug*参数。

因为我们要使用*post_list*视图（view）,编辑*blog/post/list.html* 模板（template），使用*posts*对象修改pagination，如下所示：

    {% include "pagination.html" with page=posts %}

在`{% for %}`循环上方添加如下代码：

```html
{% if tag %}
  <h2>Posts tagged with "{{ tag.name }}"</h2>
{% endif %}
```
    
如果用户正在访问blog，他会看到所有帖子列表。如果他指定一个标签来过滤所有的帖子，他就会看到以上的信息。现在，修改标签的显示方式，如下所示：

```html
<p class="tags">
  Tags:
  {% for tag in post.tags.all %}
    <a href="{% url "blog:post_list_by_tag" tag.slug %}">
      {{ tag.name }}
    </a>
    {% if not forloop.last %}, {% endif %}
  {% endfor %}
</p>
```
现在，我们循环一个帖子的所有标签，通过某一标签来显示一个自定义的链接URL。我们通过`{% url "blog:post_list_by_tag" tag.slug %}`，用URL的名称以及标签 slug作为参数来构建URL。我们使用逗号分隔这些标签。

在浏览器中打开 http://127.0.0.1:8000/blog/ 然后点击任意的标签链接，你会看到通过该标签过滤过的帖子列表，如下所示：

![django-2-12](http://ohqrvqrlb.bkt.clouddn.com/django-2-12.png)

##检索类似的帖子

如今，我们已经可以给我们的blog帖子加上标签，我们可以通过它们做更多有意思的事情。通过使用标签，我们能够很好的分类我们的blog帖子。拥有类似主题的帖子一般会有几个共同的标签。我们准备创建一个功能：通过帖子共享的标签数量来显示类似的帖子。这样的话，当一个用户阅读一个帖子，我们可以建议他们去读其他有关联的帖子。

为了通过一个特定的帖子检索到类似的帖子，我们需要做到以下几点：

* 返回当前帖子的所有标签。
* 返回所有带有这些标签的帖子。
* 在返回的帖子列表中排除当前的帖子，避免推荐相同的帖子。
* 通过和当前帖子共享的标签数量来排序所有的返回结果。
* 假设有两个或多个帖子拥有相同数量的标签，推荐最近的帖子。
* 限制我们想要推荐的帖子数量。

这些步骤可以转换成一个复杂的查询集（QuerySet），该查询集（QuerySet）我们需要包含在我们的*post_detail*视图（view）中。打开blog应用中的*view.py*文件，在顶部添加如下导入：

    from django.db.models import Count

这是Django ORM的*Count*聚合函数。这个函数允许我们处理聚合计算。然后在*post_detail*视图（view）的*render()*函数之前添加如下代码：

```python
# List of similar posts
post_tags_ids = post.tags.values_list('id', flat=True)
similar_posts = Post.published.filter(tags__in=post_tags_ids)\
                               .exclude(id=post.id)
similar_posts = similar_posts.annotate(same_tags=Count('tags'))\
                            .order_by('-same_tags','-publish')[:4]
```
    

以上代码的解释如下：

* 1.我们取回了一个包含当前帖子所有标签的ID的Python列表。*values_list()* 查询集（QuerySet）返回包含给定的字段值的元祖。我们传给元祖`flat=True`来获取一个简单的列表类似`[1,2,3,...]`。
* 2.我们获取所有包含这些标签的帖子排除了当前的帖子。
* 3.我们使用*Count*聚合函数来生成一个计算字段*same_tags*，该字段包含与查询到的所有 标签共享的标签数量。
* 4.我们通过共享的标签数量来排序（降序）结果并且通过*publish*字段来挑选拥有相同共享标签数量的帖子中的最近的一篇帖子。我们对返回的结果进行切片只保留最前面的4篇帖子。

在*render()*函数中给上下文字典增加*similar_posts*对象，如下所示：

    return render(request,
                    'blog/post/detail.html',
                    {'post': post,
                    'comments': comments,
                    'comment_form': comment_form,
                    'similar_posts': similar_posts})


现在，编辑*blog/post/detail.html*模板（template）在帖子评论列表前添加如下代码：

```html
<h2>Similar posts</h2>
  {% for post in similar_posts %}
    <p>
      <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
    </p>
  {% empty %}
    There are no similar posts yet.
  {% endfor %}
```
    
推荐你在你的帖子详情模板中添加标签列表，就像我们在帖子列表模板所做的一样。现在，你的帖子详情页面看上去如下所示：
![django-2-13](http://ohqrvqrlb.bkt.clouddn.com/django-2-13.png)

你已经成功的为你的用户推荐了类似的帖子。*django-taggit*还内置了一个`similar_objects()` 管理器（manager）使你可以通过共享的标签返回所有对象。你可以通过访问 http://django-taggit.readthedocs.org/en/latest/api.html 看到所有django-taggit管理器。

##总结
在本章中，你学习了如何使用Django的表单和模型（model）表单。你创建了一个通过email分享你的站点内容的系统，还为你的博客创建了一个评论系统。通过集成一个可复用的应用，你为你的帖子增加了打标签的功能。同时，你还构建了一个复杂的查询集（QuerySets)用来返回类似的对象。

在下一章中，你会学习到如何创建自定义的模板（temaplate）标签（tags）和过滤器（filters）。你还会为你的博客应用构建一个自定义的站点地图，集成一个高级的搜索引擎。

书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**2016年12月13日发布（3天完成第二章的翻译，但没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月17日校对完成（不是精校，希望大家多指出需要修改的地方）**

**2017年3月6日精校完成（感谢感谢大牛 @kukoo 的精校！）**

**2017年3月21日再度精校（感谢大牛 @妈妈不在家 的精校！初版我已经不敢再看！）**




