书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**2016年12月13日发布（3天完成第二章的翻译，但没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月17日校对完成（不是精校，希望大家多指出需要修改的地方）**

（译者注：翻译完第一章后，发现翻译第二章的速度上升了不少，难道这就是传说中的经验值提升了？）

#第二章
##使用高级特性来优化你的blog

在上一章中，你创建了一个基础的blog应用。现在你将要改造它成为一个功能更加齐全的blog通过一些高级特性例如通过email来分享帖子，添加评论，标记帖子，检索出相似的帖子。在本章中，你将会学习以下几点：

* 通过Django发送email
* 在视图（views）中创建并操作表单
* 通过模型（models）创建表单
* 构建复杂的查询集（QuerySets)

###使用email分享帖子

首先，我们将允许用户使用email发送帖子来分享帖子。让我们花费一小会时间来想下你该如何使用**视图（views）**，**URLs**和**模板（templates）**来创建这个功能根据你在上一章中学到的知识。现在，核对一下你需要哪几点才能允许你的用户通过邮箱来发送帖子。你需要做到以下几点：

* 创建一个表单给用户来填写他们的姓名，email，收件方以及非必须的评论。
* 在*views.py*文件中创建一个视图（view）来操作发布的数据和发送email
* 在blog应用的*urls.py*中为新的视图（view）添加一个URL模式
* 创建一个模板（template）来展示这个表单

###使用Django创建表单

让我们开始创建一个表单用来分享帖子。Django有一个内置的表单框架允许你通过简单的方式来创建表单。这个表单框架允许你定义你的表单字段，指定这些字段必须展示的方式，以及指定这些字段如何验证输入的数据。Django表单框架还提供一个灵活的方式来渲染表单以及操作数据。

Django从两个基础类来创建表单：

* Form: 允许你创建一个标准表单
* ModelForm: 允许你创建一个表单来创建或者更新模型（model）的实例

首先，创建一个*forms.py*文件在你blog应用的目录下，输入以下代码：

```python
from django import forms

class EmailPostForm(forms.Form):
    name = forms.CharField(max_length=25)
    email = forms.EmailField()
    to = forms.EmailField()
    comments = forms.CharField(required=False,
									     widget=forms.Textarea)
```

这是你的第一个Django表单。看下代码：我们已经创建了一个表单，该表单继承基础*Form*类。因此我们可以使用不同的字段类型以使Django来验证字段。

> 表单可以存在你的Django项目的任何地方，但按照惯例将它们放在每一个应用下面的*forms.py*文件中

*name*字段是一个*CharField*。这种类型的字段等同于`<input type=“text”>`HTML元素。每种字段类型都有默认的控件来确定它在HTML中的展示形式。默认的控件能够被覆盖通过使用*widget*属性。在*comments*字段中，我们使用*Textarea*控件来使它展示成一个`<textarea>`HTML元素来代替默认的`<input>`元素。

字段的验证也依赖于字段的类型。举个例子，*email*和*to*字段是*EmailField*,这两个字段都需要一个有效的email地址，否则字段验证将会抛出一个*forms.ValidationError*异常导致表单验证不通过。其他的参数也会考虑到被表单验证：我们指定*name*字段最多只能输入25个字符，以及通过设置`required=False`表明*comments*字段不是必填项。以上这些都考虑到了字段验证。目前我们在表单中使用的这些字段类型只是Django支持的表单字段的一部分。要查看更多可利用的表单字段，你可以访问：https://docs.djangoproject.com/en/1.8/ref/forms/fields/

###在视图（views）中操作表单

你需要创建一个新的视图（view）当表单成功提交后进行表单操作和发送email。编辑blog应用下的*views.py*文件，添加以下代码：

```python
from .forms import EmailPostForm

def post_share(request, post_id):
    # retrieve post by id
    post = get_object_or_404(Post, id=post_id, status='published')
    
    if request.method == 'POST':
        # Form was submitted
        form = EmailPostForm(request.POST)
        if form.is_vlid():
            # Form fields passed validation
            cd = form.cleaned_data
            # ... send email
    else:
        form = EmailPostform()
    return render(request, 'blog/post/share.html', {'post': post,
										                       'form: form})
```

以上视图（view）完成了以下工作：

* 我们定义了*post_share*视图，将*request*对象和*post_id*作为该视图参数。
* 我们使用*get_object_or_404*快捷方法通过ID获取对应的帖子，并且确保获取的帖子有一个*published*状态。
* 我们使用同一个视图（view）来展示初始表单和处理提交后的数据。我们会辨别如果表单进行了提交，该提交是否基于指定的请求方法。我们使用*POST*来提交表单。我们假设当我们遇到一个*GET*请求，就需要展示一个空的表单，当我们遇到一个*POST*请求，我们就需要处理表单提交上来的数据。因此，我们使用`request.method == 'POST'`来区分这两种场景。

下面是展示和操作表单的进程：

* 1.当视图（view）加载初期遇到一个*GET*请求，我们会创建一个新的表单实例，该实例会被用来展示一个空的表单在模板（template）中：
    
        form = EmailPostForm()
    
* 2.当用户填写好了表单并通过*POST*提交表单。那么，我们会创建一个表单实例来使用提交的数据，这些数据被包含在`request.POST`中：
        
        if request.method == 'POST':
           # Form was submitted
           form = EmailPostForm(request.POST)
           
* 3.在以上步骤之后，我们使用表单的*is_valid()*方法来验证提交的数据。这个方法会验证表单引进的数据，如果所有的字段都是有效数据，将会返回*True*。一旦有任何一个字段是无效的数据，*is_valid()*就会返回*False*。你可以通过访问 `form.errors`来查看所有验证错误的列表。
* 4.如果该表单无效，我们会再一次渲染表单在模板（template）中通过提交的数据。我们将会显示验证错误在模板（template）中。
* 5.如果表单有效，我们获取验证通过的数据通过读取`form.cleaned_data`。这个属性是一个字典包含表单字段和它们的值。

> 如果你的表单数据没有通过验证，*cleaned_data*只会包含验证通过的数据

现在，你需要学习如何使用Django来发送email。

##使用Django发送email

使用Django发送email非常简单。首先，你需要有一个本地的SMTP服务或者定义一个外部SMTP服务的配置，在你的项目中的*settings.py*文件中添加以下设置：

* EMAIL_HOST: SMTP服务地址。默认本地。
* EMAIL_POSR: SMATP服务端口，默认25。
* EMAIL_HOST_USER: SMTP服务的用户名。
* EMAIL_HOST_PASSWORD: SMTP服务的密码。
* EMAIL_USE_TLS: 是否使用TLS加密协议。
* EMAIL_USE_SSL: 是否使用SSL加密协议。

如果你没有本地SMTP服务，你可以使用你的email服务供应商提供的SMTP服务。下面提供了一个简单的例子展示如何通过使用Google账户的Gmail服务来发送email：

```python
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = 'your_account@gmail.com'
EMAIL_HOST_PASSWORD = 'your_password'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
```
    
打开Python shell运行以下代码来发送一封email：

```shell
>>> from django.core.mail import send_mail
>>> send_mail('Django mail', 'This e-mail was sent with Django.','your_account@gmail.com', ['your_account@gmail.com'], fail_silently=False)
```
    
*send_mail()*方法需要这些参数：邮件主题，内容，发送人以及一个收件人的列表。通过设置可选参数`fail_silently=False`，我们告诉这个方法如果email没有发送成功那么需要抛出一个异常。如果你看到输出是*1*，证明你的email发送成功了。如果你通过Gmail来发送邮件使用之前的配置，你可能需要去 https://www.google.com/settings/security/lesssecureapps 设置一下低安全级别的应用权限。(译者注：练习时老老实实用QQ邮箱吧）

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
            post_url = request.build_absolute_uri(
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

请注意，我们声明了一个*sent*变量并且当帖子被成功发送时赋予它*True*。当表单成功提交的时候，我们会使用这个变量在模板（template）中显示一条成功提示。因为我们需要在email中包含帖子的超链接，所以我们通过使用`post.get_absolute_url()`方法来获取到帖子的绝对路径。我们将这个绝对路径作为`request.build_absolute_uri()`的输入值来构建一个完整的HTTP链接。我们通过使用验证过的表单数据来构建email的主题和消息内容并最终发送email给表单中的*to*字段中包含的所有email地址。

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

这个模板（tempalte）专门用来显示一个表单或当email成功发送后展示一条成功提示信息。如你所见，我们创建的HTML表单元素里面表明了它必须通过POST方法提交：

    <form action="." method="post">

接下来我们要包含真实的表单实例。我们告诉Django使用HTML的`<p>`元素通过使用`as_p`方法来渲染它的字段。我们也可以通过使用`as_ul`采用无序列表来渲染表单或者使用`as_table`展示成一个HTML表格。如果我们想要渲染每一个字段，我们可以通过迭代字段例如下方的例子：

```html
{% for field in form %}
  <div>
    {{ field.errors }}
    {{ field.label_tag }} {{ field }}
  </div>
{% endfor %}
```
    
`{% csrf_token %}`模板（tempalte）标签（tag）引进了一个隐藏的字段带有一个自动生成的标记（token）来避免*Cross-Site request forgery(CSRF)*攻击。这些攻击由一个恶意的站点或程序组成并会执行一个不需要的操作给你站点中的用户。你可以找到更多的信息通过访问 https://en.wikipedia.org/wiki/Cross-site_request_forgery 。

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
    
请记住，我们通过使用Django提供的`{% url %}`模板（template）标签（tag）来生成动态的URL。我们使用叫做*blog*的命名空间和叫做*post_share*的URL，以及我们传递的帖子ID为参数来构建绝对的URL。

现在，通过命令`python manage.py runserver`命令来启动开发服务器，在浏览器中打开 http://127.0.0.1:8000/blog/ 。点击任意一个帖子标题查看详情页面。在帖子内容的下方，你会看到我们刚刚添加的链接，如下所示：
![django-2-1](http://ohqrvqrlb.bkt.clouddn.com/django-2-1.png)

点击**Share this post**你会看到一个页面，该页面包含通过email分享这个帖子的表单。看上去如下所示：
![django-2-2](http://ohqrvqrlb.bkt.clouddn.com/django-2-2.png)

为这个表单提供的CSS样式被包含在示例代码中的 *static/css/blog.css*文件中。当你点击**Send e-mail**按钮，这个表单会提交并验证。如果所有的字段都通过了验证，你会得到一条成功信息如下所示：
![django-2-3](http://ohqrvqrlb.bkt.clouddn.com/django-2-3.png)

如果你输入了错误的数据，你会看到表单被再次渲染，包含所有的验证错误信息，如下所示：
![django-2-4](http://ohqrvqrlb.bkt.clouddn.com/django-2-4.png)

##创建一个评论系统

现在我们要为blog创建一个评论系统，这样用户可以在帖子上进行评论。创建一个评论系统，你需要做到以下几点：

* 创建一个模型（model）用来保存评论
* 创建一个表单用来提交评论并且验证输入的数据
* 添加一个视图（view）来处理表单和保存新的评论到数据库中
* 编辑帖子详情模板（template）来展示评论列以及用来添加新评论的表单

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

以上就是我们的*Comment*模型（model）。它包含了一个外键用来关联一个单独的帖子。在*Comment*模型（model）中定义多对单（many-to-one）的关系是因为每一条评论只能在一个帖子下生成，而每一个帖子又可能包含多个评论。*related_name*属性允许我们命名这个属性这样我们就可以使用这个关系从有关联的对象来读取这儿。定义好这个之后，我们可以通过使用 `comment.post`从一条评论来取到对应的帖子或者通过使用`post.comments.all()`取回一个帖子所有的评论。如果你没有定义*related_name*属性，Django会使用这个模型（model）的命名加上*_set*（例如：comment_set）来命名关联对象读取这儿的管理器（manager）。

访问https://docs.djangoproject.com/en/1.8/topics/db/examples/many_to_one/, 你可以学习更多关于多对单的关系。

我们包含了一个*active*布尔字段，该字段可以被我们用来手动停用无效的评论。我们使用*created*字段来排序评论，默认根据创建时间来进行排序。

你刚创建的这个新的*Comment*模型（model）并没有同步到数据库中。运行以下命令通过新的模型（model）生成一个新的数据迁移：

    python manage.py makemigrations blog

你会看到如下输出：

```shell
Migrations for 'blog':
  0002_comment.py:
    - Create model Comment
```

Django会在blog应用下的*migrations/*目录中生成了一个*0002_comment.py*文件。现在你需要创建一个有关联的数据库模式并且应用这些改变到数据库中。运行以下命令来应用已经存在的数据迁移：

    python manage.py migrate

你会获取以下输出：

    Applying blog.0002_comment... OK
    
我们刚刚创建的数据迁移已经被执行，并且一张*blog_comment*表已经存在数据库中。


现在，我们可以添加我们新的模型（model）到管理站点中并通过简单的接口来管理评论。打开blog应用下的*admin.py*文件，添加如下内容：

```python
from .models import Post, Comment

class CommentAdmin(admin.ModelAdmin):
    list_display = ('name', 'email', 'post', 'created', 'active')
    list_filter = ('active', 'created', 'updated')
    search_fields = ('name', 'email', 'body')
admin.site.register(Comment, CommentAdmin)
```
    
通过运行命令`python manage.py runserver`来启动开发服务器然后在浏览器中打开 http://127.0.0.1:8000/admin/ 。 你会看到新的模型（model）被包含**Blog**区域，如下所示：
![django-2-5](http://ohqrvqrlb.bkt.clouddn.com/django-2-5.png)

我们的模型（model）现在已经被注册到了管理站点种，这样我们就可以使用简单的接口来管理评论实例。

##从模型（models）创建表单

我们仍然需要构建一个表单让我们的用户在blog帖子上进行评论。请记住，Django有两个基础类可以构建表单：*Form*和*ModelForm*。你已经使用过前者可以让用户使用email来分享帖子。在下面的例子中，你需要使用*ModelForm*因为你必须动态的构建表单根据你的*Comment*模型（model）。编辑blog应用下的*forms.py*，添加如下代码：

```python
from .models import Comment

class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ('name', 'email', 'body')
```
    
根据模型（model）创建表单，我们只需要在这个表单的*Meta*类里表明哪个模型（model）被用来构建表单。Django会对这个模型（model）内审，并为我们动态的构建表单。每一种模型（model）字段类型都有对应的默认表单字段类型。默认的，Django构建一个表单字段给模型（model）中包含的每个字段。当然，你可以通过使用*fields*列明确的告诉框架你想在表单中包含哪些字段，或者通过使用*exclude*列定义哪些字段你不需要的。对于我们的*CommentForm*,我们在表单中只需要*name*,*email*,和*body*字段，因为我们只需要用到这3个字段让我们的用户来填写。

##在视图（views）中操作*ModelForms*

我们会使用帖子详情视图（view）来实例化表单，并且处理它为了保持它的简单。编辑*views.py*文件**(注: 原文此处有错，应为 `views.py`)**，导入*Comment*模型（model）和*CommentForm*表单，并且修改*post_detail*视图（view）如下所示：

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
                  'comment_form': comment_form})
```

让我们来回顾一下我们刚才对视图（view）添加了哪些操作。我们使用*post_detail*视图（view）来显示帖子和该帖子的评论。我们添加了一个查询集（QuerySet）来获取这个帖子所有有效的评论：
    
    comments = post.comments.filter(active=True)
    
我们从*post*对象开始构建这个查询集（QuerySet）。通过使用在*Comment* 模型（model）中定义的*related_name属性关系来返回所有有关系的评论对象赋给*comments*。

我们还在这个视图（view）中让我们的用户添加一条新的评论。如果调用这个视图（view）通过GET请求，我们就通过使用`comment_fomr = commentForm()`创建了一个表单实例。如果请求是一个POST，我们使用提交的数据来实例化表单并且通过使用*is_valid()*方法来验证数据。如果这个表单是无效的，我们会在模板（template）中渲染验证错误的信息。如果表单通过验证，我们会做以下的操作：

* 1.我们通过调用这个表单的*save()*方法创建一个新的*Comment*对象，如下所示：

`new_comment = comment_form.save(commit=False)` 

*saven()*方法创建了一个模型（model）的实例，这样这个表单被联系起来并且保存表单的数据到数据库中。如果你调用这个方法时设置`comment=False`，你创建的模型（model）实例不会即时保存到数据库中。当你想在最终保存之前修改这个model对象会非常方便，我们接下来将做这一步骤。*save()*方法是给*ModelForm*用的，而不是给*Form*实例用的，因为*Form*实例没有联系上任何模型（model）。

* 2.我们分配当前的帖子给我们刚才创建的评论：
    
        new_comment.post = post
    
    通过以上动作，我们指定新的评论是属于这篇给予的帖子。

* 3.最后我们保存新的评论到数据库，如下所示：
    
        new_comment.save()
        
我们的视图（view）已经准备好显示和处理新的评论了。

##添加评论到帖子详情模板（template）

我们已经创建了功能来管理一篇帖子的评论。现在我们需要修改我们的*post_detail.html*模板（template）来适应这个功能，通过做到以下步骤：

* 显示这篇帖子的评论总数
* 显示评论列
* 显示一个表单给用户来添加新的评论

首先，我们要添加评论的总数。打开*blog_detail.html*模板（template）在*content*区块中添加如下代码：

```html
{% with comments.count as total_comments %}
  <h2>
    {{ total_comments }} comment{{ total_comments|pluralize }}
  </h2>
{% endwith %}
```    

我们使用Django ORM在模板（template）中执行这个查询集（QuerySet）`comments.count()`。注意，Django模板（template）语言中不使用圆括号来调用方法。`{% with %}` 标签（tag）允许我们分配一个值给新的变量，这个变量可以一直存在直到遇到`{% endwith %}`标签（tag）。

> `{% with %}`模板（template）标签（tag）是非常有用的，可以避开直接使用数据库或花费大量的时间在处理复杂的方法上。

我们使用*pluralize*模板（template）过滤器（filter）在单词*comment*的后面展示*total_comments*的复数形式。模板（Template）过滤器（filters）将它们应用的变量值作为输入然后返回计算后的值。我们将会在*第三章 扩展你的博客应用*中讨论更多的模板过滤器（tempalte filters）。

*pluralize*模板（template）过滤器（filter）如果值不为 1，会在值的末尾显示一个"s"。在之前的文本将会渲染成类似：*0 comments*, *1 comment* 或者 *N comments*。Django内置大量的模板（template）标签（tags）和过滤器（filters）来帮助你通过各种方法展示各类信息。

现在，让我们包含评论列。添加以下行到模板（template）中在之前的代码后面：
    
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
    
我们使用`{% for %}`模板（template）标签（tag）来循环所有的评论。如果*comments*列为空我们会显示一个默认的信息，告诉我们的用户这篇帖子还没有任何评论。我们通过使用 `{{ forloop.counter }}`变量来枚举评论，该变量包含了每次迭代的循环数。之后我们显示发送评论的用户名，日期，和评论的内容。

最后，当表单提交成功后,你需要渲染表单或者显示一条成功的信息。在之前的代码后面添加如下内容：

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
    
这段代码非常简洁明了：如果*new_comment*对象存在，我们会展示一条成功信息因为成功创建了一条新评论。否则，我们通过一个段落`<p>`元素渲染表单中的每一个字段，并且包含必须CSRF标记给POST请求。在浏览器中打开 http://127.0.0.1:8000/blog/ 然后点击任意一篇帖子的标题查看它的详情页面。你会看到如下页面展示：

![django-2-6](http://ohqrvqrlb.bkt.clouddn.com/django-2-6.png)

使用该表单添加数条评论。这些评论会在你的帖子下面根据时间排序来展示，类似下图：
![django-2-7](http://ohqrvqrlb.bkt.clouddn.com/django-2-7.png)

在你的浏览器中打开 http://127.0.0.1:8000/admin/blog/comment/ 。你会看到管理页面中存在你创建的评论列。随便点击其中的一条来进行编辑，取消选择**Active**复选框，然后点击**Save**按钮。你会再次被重定向到评论列页面，刚才编辑的评论**Active**列将会显示一个不激活的图表。类似下图：
![django-2-8](http://ohqrvqrlb.bkt.clouddn.com/django-2-8.png)

##增加标签（tagging）功能

在实现了我们的评论系统之后，我们要创建一个方法来给我们的帖子添加标签。我们会在我们的项目中集成第三方的Django标签应用来使用。*django-taggit*是一个可重用的应用，它会提供给你一个*Tag*模型（model）和一个管理器（manager）来方便的给任何模型（model）添加标签。访问 https://github.com/alex/django-taggit 你可以阅读到源码。

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
    
打开你的blog应用下的*model.py*文件添加*django-taggit*提供的*TaggableManager*管理器（manager）给*Post*模型（model），使用如下代码：

```python    
from taggit.managers import TaggableManager
class Post(models.Model):
    # ...
    tags = TaggableManager()
```
        
这个*tags*管理器（manager）允许你添加，获取以及移除标签从*Post*对象上。

运行以下命令为你的模型（model）改变创建一个数据库迁移：

    python manage.py makemigrations blog

你会看到如下输出：

```shell    
Migrations for 'blog':
  0003_post_tags.py:
    - Add field tags to post
```

现在，运行以下命令来创建必需的数据库表给*django-taggit*模型（model）并且同步你的模型（model）改变：

    python manage.py migrate

你会看到以下输出：

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
    
最后，移除一个标签并且再次检查标签列：

```shell
>>> post.tags.remove('django')
>>> post.tags.all()
[<Tag: jazz>, <Tag: music>]
```
    
非常简单，对吧？运行命令`python manage.py runserver`启动开发服务器，在浏览器中打开 http://127.0.0.1:8000/admin/taggit/tag/ 。你会看到管理页面包含了*taggit*应用的*Tag*对象列：

![django-2-9](http://ohqrvqrlb.bkt.clouddn.com/django-2-9.png)

转到 http://127.0.0.1:8000/admin/blog/post/ 并点击一篇帖子进行编辑。你会看到帖子中包含了一个新的**Tags**字段如下所示，你可以非常容易的编辑它：
![django-2-10](http://ohqrvqrlb.bkt.clouddn.com/django-2-10.png)

现在，我们准备编辑我们的blog帖子来显示这些标签。打开*blog/post/list.html* 模板（template）在帖子标题下方添加如下HTML代码：

    <p class="tags">Tags: {{ post.tags.all|join:", " }}</p>

*join*模板（template）过滤器（filter）的功能类似Python的`join()`方法，连接元素通过给予的字符串。在浏览器中打开 http://127.0.0.1:8000/blog/ 。 你会看到标签在每一篇帖子的标题下面：
![django-2-11](http://ohqrvqrlb.bkt.clouddn.com/django-2-11.png)

现在，让我们来编辑我们的*post_list*视图（view）让用户可以列出打上了特定标签的所有帖子。打开blog应用下的*views.py*文件，导入*Tag*模型（model）从*django-taggit*中，然后修改*post_list*视图（view）随意的过滤帖子通过标签，如下所示：

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
* 2.视图（view）的内部，我们构建了初始的查询集（QuerySet），取回所有发布状态的帖子，并且如果有一个给予的标签slug，我们获取*Tag*对象通过给予的slug使用`get_object_or_404()`快捷方法。
* 3.之后我们过滤帖子列通过给予的标签中包含的标签。因为有一个多对多（many-to-many）的关系，我们必须过滤在给予的列中包含的标签，该列在我们的场景中只包含一个元素。

要记住查询集（QuerySets)是惰性的。这个查询集（QuerySets)只有当我们在模板（template）中循环渲染帖子列表才会被执行。

最后，修改视图（view）最底部的*render()*函数来传递*tag*变量给模板（template）。这个视图（view）完成后如下所示：

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
    
再添加以下代码，使用附加的URL参数用来通过标签排序帖子：

```python
url(r'^tag/(?P<tag_slug>[-\w]+)/$',views.post_list,
    name='post_list_by_tag'),
```
    
如你所见，两个模式都指向了相同的视图（view），但是我们可以给它们不同的命名。第一个模式会调用*post_list*视图（view）并且不带上任何可选参数。第二个模式会调用这个视图（view）带上*tag_slug*参数。

因为我们要使用*post_list*视图（view）,编辑*blog/post/list.html* 模板（template）修改pagination使用*posts*对象，如下所示：

    {% include "pagination.html" with page=posts %}

在`{% for %}`循环上方添加如下代码：

```html
{% if tag %}
  <h2>Posts tagged with "{{ tag.name }}"</h2>
{% endif %}
```
    
如果用户正在访问blog，他会看到所有帖子列。如果他指定一个标签来过滤所有的帖子，他就会看到以上的信息。现在，修改标签的显示方式，如下所示：

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
    
现在，我们循环一篇帖子的所有标签显示一个自定义的链接给URL来过滤帖子通过循环到的标签。我们构建URL通过使用` {% url "blog:post_list_by_tag"
tag.slug %}`，使用URL的命名以及标签slug作为参数。我们使用逗号分隔这些标签。

在浏览器中打开 http://127.0.0.1:8000/blog/ 然后点击任意的标签链接，你会看到通过该标签过滤过的帖子列，如下所示：

![django-2-12](http://ohqrvqrlb.bkt.clouddn.com/django-2-12.png)

##获取类似的帖子

如今，我们已经可以给我们的blog帖子加上标签，我们可以通过它们做更多有意思的事情。通过使用标签，我们能够很好的分类我们的blog帖子。拥有类似主题的帖子一般会有几个标签。我们会创建一个功能通过帖子共享的标签数量来显示类似的帖子。通过这个方法，当一个用户阅读一个帖子，我们可以建议他们去读其他有关联的帖子。

为了给一个指定的帖子获取类似的帖子，我们需要做到以下几点：

* 返回当前帖子的所有标签。
* 获取所有的帖子，这些帖子带有当前帖子带有的任意标签。
* 在返回的帖子列中排除当前的帖子避开再次阅读到相同的帖子。
* 通过和当前帖子共享的标签数量来排序所有的返回结果。
* 假设有两个或多个帖子拥有相同数量的标签，推荐最近的帖子。
* 限制我们想要推荐的帖子数量。

这些步骤可以转换成一个复杂的查询集（QuerySet），该查询集（QuerySet）我们需要包含在我们的*post_detail*视图（view）中。打开blog应用中的*view.py*文件，在顶部添加如下导入：

    from django.db.models import Count

这是Django ORM的*Count*聚合函数。这个函数允许我们处理聚合计算。然后在*post_detail*视图（view）的*render()*功能之前添加如下代码：

```python
# List of similar posts
post_tags_ids = post.tags.values_list('id', flat=True)
similar_posts = Post.published.filter(tags__in=post_tags_ids)\
                               .exclude(id=post.id)
similar_posts = similar_posts.annotate(same_tags=Count('tags'))\
                            .order_by('-same_tags','-publish')[:4]
```
    

以上代码的解释如下：

* 1.我们取回了一个包含当前帖子的所有标签的ID的Python列。*values_list()* 查询集（QuerySet）返回元祖包含给予的字段值。我们通过使用`flat=True`处理它获取一个简单的列表类似`[1,2,3,...]`。
* 2.我们获取所有包含这些标签的帖子排除了当前的帖子。
* 3.我们使用*Count*聚合函数来生成一个计算字段*same_tags*，该字段包含共享的标签数量通过查询所有的标签。
* 4.我们通过共享的标签数量来排序（降序）结果并且通过*publish*字段来挑选拥有相同共享标签数量的帖子中最近的一篇帖子。我们对返回的结果进行切片只保留最前面的4篇帖子。

在*render()*函数中给上下文字典增加*similar_posts*对象，如下所示：

    return render(request,
                    'blog/post/detail.html',
                    {'post': post,
                    'comments': comments,
                    'comment_form': comment_form,
                    'similar_posts': similar_posts})


现在，编辑*blog/post/detail.html*模板（template）在帖子评论列前添加如下代码：

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
    
通过我们在帖子列表模板（tempalte）中使用的同样方法，你也可以在你的帖子详情模板（template）中添加标签的列来推荐类似的帖子。现在，你的帖子详情页面看上去如下所示：
![django-2-13](http://ohqrvqrlb.bkt.clouddn.com/django-2-13.png)

你已经成功的为你的用户推荐了类似的帖子。*django-taggit*还内置了一个`similar_objects()` 管理器（manager）使你可以用来通过共享的标签返回所有对象。你可以通过访问 http://django-taggit.readthedocs.org/en/latest/api.html 看到所有django-taggit管理器。

##总结
在本章中，你学习了如何使用Django的表单和模型（model）表单。你创建了一个通过email用来分享你的站点内容的系统，还创建了一个评论系统给你的blog。你为你的帖子增加了标签的功能，集成了一个可复用的应用，同时，你还构建了一个复杂的查询集（QuerySets)用来返回类似的对象。

在下一章中，你会学习到如何创建自定义的模板（temaplate）标签（tags）和过滤器（filters）。你还会构建一个自定义的站点地图和RSS，然后集成一个高级的搜索引擎在你的应用中。


##译者总结：
第二章翻译的速度超出之前的预期（**渣翻**当然速度快- -|||），在翻译成中文的过程中，我也终于想通了之前看这章时产生的很多困惑点，因为需要根据上下文以及代码执行情况来翻译成最接近原意的中文。接下来要开始翻译第三章，第三章最后集成搜索引擎的部分我非常排斥，因为这部分照书的操作下来竟然是失败的。。。再看吧，也许通过翻译我能最终执行成功。不过在翻译第三章之前，我要花点时间编辑下前两章的翻译，修改一些语句不通的地方，以及加入更多的我自己学习过程中的理解，就这样。


书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**2016年12月13日发布（3天完成第二章的翻译，但没有进行校对，有很多错别字以及模糊不清的语句，请大家见谅）**

**2017年2月17日校对完成（不是精校，希望大家多指出需要修改的地方）**


