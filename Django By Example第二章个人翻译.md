（译者注：翻译完第一章后，发现翻译第二章的速度上升了不少，难道这就是传说中的经验值提升了？）

#第二章
##使用高级特性来优化你的博客

在上一章中，你创建了一个基础的博客应用。现在你将要改造它成为一个功能更加齐全的博客，利用一些高级的特性例如通过email来分享帖子，添加评论，给帖子打上tag，检索出相似的帖子。在本章中，你将会学习以下几点：

* 使用Django发送email
* 在views中创建并操作表单
* 通过models创建表单
* 构建复杂的QuerySets

###通过email分享帖子
首先，我们会允许用户通过发送邮件来分享他们的帖子。首先让我们花费一小会时间来想下你该如何使用views，urls和templates来创建这个功能根据你在上一章中学到的知识。现在，核对一下你需要哪几点才能允许你的用户通过邮箱来发送帖子。你需要做到以下几点：

* 创建一个表单给用户来填写他们的姓名，email，收件方以及评论，评论不是必选的。
* 在views.py文件中创建一个view来操作发布的数据和发送email
* 在博客应用的urls.py中为新的view添加一个URL pattern
* 创建一个模板来展示这个表单

###使用Django创建表单
让我们开始创建一个用来分享帖子的表单。Django有一个内置的表单框架允许你通过简单的方式来创建表单。这个表单框架允许你定义你的表单字段，指定他们必须展示的方式，以及指定他们如何验证输入的数据。Django表单框架还提供一个灵活的方式来渲染表单以及操作数据。

Django应用了两个基础类来创建表单：

* Form: 允许你创建一个标准表单
* ModelForm: 允许你创建一个表单可用于创建或者更新model的实例

首先，创建一个forms.py文件在你博客应用的目录下，输入以下代码：

	from django import forms
	class EmailPostForm(forms.Form):
		name = forms.CharField(max_length=25)
		email = forms.EmailField()
		to = forms.EmailField()
		comments = forms.CharField(required=False,
									widget=forms.Textarea)

这是你的第一个Django表单。看下代码：我们已经创建了一个继承基础Form类的表单。我们使用不同的字段类型以使Django来验证字段。

> 表单可以存在你的Django项目的任何地方，但按照惯例将它们放在每一个应用下面的forms.py文件中

name字段是一个CharField。这种类型的字段等同于`<input type=“text”>`HTML元素。每个字段都有默认的控件来确定它在HTML中的展示。通过改变控件的属性可以重写默认的控件。在comment字段中，我们使用Textarea控件来使它展示成一个`<textarea></textarea>`HTML元素来代替默认的`<input>`元素。

字段的验证也依赖于字段的类型。举个例子，email和to字段是*EmailField*,它们需要一个有效的地址，否则字段验证不通过将会返回forms.ValidationError异常导致表单提交失败。其他的参数将进入表单验证：我们指定name字段最多只能输入25个字符，通过设置`required=False`表明comments字段不是必填项。目前我们在表单中使用的这些字段类型只是Django支持的表单字段的一部分。要查看更多可利用的表单字段，你可以访问：https://docs.djangoproject.com/en/1.8/ref/forms/fields/

###在views中操作表单
你必须创建一个新的view当表单成功提交后进行操作和发送email。编辑博客应用下的views.py文件，添加以下代码：

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
			form = emailPostform()
		return render(request, 'blog/post/share.html', {'post': post,
												'form: form})

该view完成了以下工作：

* 我们定义了post_share视图，参数为request对象和post_id。
* 我们使用*get_object_or_404*方法通过ID获取对应的帖子，并且确保取回的帖子一定是已发布的状态。
* 我们使用同一个view来展示初始表单以及处理提交后的数据。
* 我们会区别被提交的表单和不基于这次请求方法的表单。我们通过使用*POST*来提交表单。我们假设我们会遇到*GET*请求，这时就需要展示一个空的表单，而我们遇到*POST*请求，我们就需要处理表单提交上来的数据。因此，我们使用`request.method == 'POST'`来区分这两种场景。

下面是展示和操作表单的进程：

* 1.当view加载后遇到一个*GET*请求，我们会创建一个新的表单实例在模板中展示一个空的表单：
    
        form = EmailPostForm()
    
* 2.当用户填写了表单并通过*POST*方式提交，我们会 创建一个表单实例来使用提交的数据，这些数据被包含在`request.POST`中：
        
        if request.method == 'POST':
           # Form was submitted
           form = EmailPostForm(request.POST)
           
* 3.在以上步骤之后，我们通过表单的*is_valid()*方法来验证提交的数据。这个方法会验证表单引进的数据，如果所有的字段都是有效数据，将会返回*True*。一旦有任何一个字段包含无效的数据，*is_valid()*将会返回*False*。你可以通过访问 `form.errors`查看所有验证错误的列表。
* 4.如果表单数据验证没有通过，我们会再次使用提交的数据在模板中渲染表单。我们会在模板中显示验证错误提示。
* 5.如果表单数据验证通过，我们通过访问`form.cleaned_data`获取验证过的数据。这个属性是一个表单字段和值的字典。

> 如果表单中的数据没有通过验证，*cleaned_data*只会包含验证过的数据

现在，你需要学习如何通过Django来发送email。

##使用Django发送email
通过Django发送email非常简单。首先，你需要有一个本地的SMTP服务或者获取到一个外部SMTP服务的配置，接下来在你的项目中的setting.py文件中设置如下内容：

* EMAIL_HOST: SMTP服务地址。默认本地。
* EMAIL_POSR: SMATP服务端口，默认25。
* EMAIL_HOST_USER: SMTP服务的用户名。
* EMAIL_HOST_PASSWORD: SMTP服务的密码。
* EMAIL_USE_TLS: 是否使用TLS加密协议。
* EMAIL_USE_SSL: 是否使用SSL加密协议。

如果你没有本地SMTP服务，你可以使用你的email提供的SMTP服务。下面提供了一个简单的例子展示如何通过使用Google账户的Gmail服务来发送email：

    EMAIL_HOST = 'smtp.gmail.com'
    EMAIL_HOST_USER = 'your_account@gmail.com'
    EMAIL_HOST_PASSWORD = 'your_password'
    EMAIL_PORT = 587
    EMAIL_USE_TLS = True
    
打开Python shell运行以下代码来发送一封email：

    >>> from django.core.mail import send_mail
    >>> send_mail('Django mail', 'This e-mail was sent with Django.',
    'your_account@gmail.com', ['your_account@gmail.com'], fail_
    silently=False)
    
*send_mail()*方法需要这些参数：邮件主题，内容，发送人以及一个收件人的列表。通过设置可选参数`fail_silently=False`，我们告诉这个方法如果email没有发送成功那么需要抛出一个异常。如果你看到输出是*1*，证明你的email发送成功了。如果你通过Gmail来发送邮件使用之前的配置，你可能需要去设置一下低安全级别的应用权限在https://www.google.com/settings/security/lesssecureapps 。(译者注：自我练习时老老实实用QQ邮箱吧）

现在，我们要将以上代码添加到我们的view中。在博客应用下的views.py文件中编辑*post_share* view如下所示：

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
    subject = '{} ({}) recommends you reading "{}"'.
    format(cd['name'], cd['email'], post.title)
    message = 'Read "{}" at {}\n\n{}\'s comments: {}'.
    format(post.title, post_url, cd['name'], cd['comments'])
    send_mail(subject, message, 'admin@myblog.com',
    [cd['to']])
    sent = True
    else:
    form = EmailPostForm()
    return render(request, 'blog/post/share.html', {'post': post,
    'form': form,
    'sent': sent})

请注意，我们声明了一个*sent*变量并且赋予它*True*当帖子被成功发送。当表单成功提交的时候，我们会使用这个变量用来在template中显示一条成功提示。因为我们需要在email中包含帖子的超链接，所以我们通过使用`post.get_absolute_url()`方法来取到帖子的绝对路径。我们将这个绝对路径作为`request.build_absolute_uri()`的输入值来构建一个完整的HTTP链接。我们通过使用验证过的表单数据来构建email的主题和消息内容并最终发送email给表单中的*to*字段中包含的所有email地址。

现在你的view已经完成了，别忘记为它去添加一个新的URL pattern。打开你的博客应用下的*urls.py*文件添加*post_share*的URL pattern如下所示：

    urlpatterns = [
    # ...
    url(r'^(?P<post_id>\d+)/share/$', views.post_share,
    name='post_share'),
    ]

##在templates中渲染表单
在通过创建表单，编写view以及添加URL patter后，我们就只剩下为这个view添加tempalte了。在`blog/templates/blog/post/`目录下创建一个新的文件并命名为*share.html*。在html文件中添加如下代码：

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

这个tempalte专门用来显示一个表单或email成功发送后展示一条成功提示信息。就像你所看见的，我们创建的HTML表单元素里面声明了提交方式采用的是POST方法：

    <form action="." method="post">    
    
接下来我们要包含真实的表单实例。我们告诉Django使用HTML的`<p>`元素来渲染它的字段通过使用`as_p`方法。我们也可以使用无序列表来渲染表单通过使用`as_ul`或者展示成一个HTML表格通过使用`as_table`。如果我们想要渲染每一个字段，我们可以通过迭代字段例如下方的例子：

    {% for field in form %}
    <div>
    {{ field.errors }}
    {{ field.label_tag }} {{ field }}
    </div>
    {% endfor %}
    
`{% csrf_token %}`tempalte标签引进了一个隐藏的字段包含一个自动生成的标记来避开*Cross-Site request forgery(CSRF)*攻击。这些攻击由一个恶意的站点或程序组成并执行一个不需要的操作给你站点中的用户。你可以找到更多的信息通过访问https://en.wikipedia.org/wiki/Cross-site_request_forgery 。
上述的标签生成的隐藏字段就像下面一样：

    <input type='hidden' name='csrfmiddlewaretoken' value='26JjKo2lcEtYkGoV9z4XmJIEHLXN5LDR' />
    
> 默认情况下，Django在所有的POST请求中都会检查CSRF标签。请记住要在所有使用POST方法提交的表单中包含*csrf_token*标签。（译者注：当然你也可以关闭这个检查，注释掉app_list中的csrf应用即可）

编辑你的*blog/post/detail.html* template添加如下的链接来分享帖子的URL在`{{ post.body|linebreaks }}`变量后面：

    <p>
      <a href="{% url "blog:post_share" post.id %}">
        Share this post
      </a>
    </p>
    
请记住，我们通过使用Django提供的`{% url %}`template标签来生成动态的URL。我们使用叫做*blog*的命名空间和叫做*post_share*的URL命名以及传递一个帖子ID为参数来构建绝对URL。

现在，通过命令`python manage.py runserver`命令来启动开发服务，在浏览器中打开 http://127.0.0.1:8000/blog/ 。点击任意一个帖子标题进入详情页面。在帖子内容的下方，你会看到我们刚刚添加的链接，如下所示：
（图片）

点击**Share this post**你会看到一个包含通过email分享这个帖子的表单。看上去如下所示：
（图片）

为这个表单提供的CSS样式被包含在示例代码中的 *static/css/blog.css*文件中。当你点击**Send e-mail**按钮，这个表单会提交并验证。如果所有的字段都通过了验证，你会得到一条成功信息如下所示：
（图片）

如果你输入了错误的数据，你会看到表单被再次渲染，并展示出验证错误信息，如下所示：
（图片）

##创建一个评论系统
现在我们准备为博客创建一个评论系统，这样用户可以在帖子上进行评论。你需要做到以下几点来创建一个评论系统：

* 创建一个modle保存评论
* 创建一表单用来提交平路你和验证输入的数据
* 添加一个view来处理表单和保存新的评论到数据库中
* 编辑帖子的详情template来展示评论列表以及用来添加新评论的表单

首先，让我们创建一个model来存储评论。打开你博客应用下的models.py文件添加如下代码：

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

以上就是我们的Comment model。它包含了一个外键用来关联一个单独的帖子。在Comment model中定义多对单的关系是因为每一条评论只能在一个帖子下生成，而每一个帖子又可能包含多个评论。*related_name*属性允许我们命名这个属性这样我们就可以使用这个关系从有关联的对象来读取这儿。定义好这个之后，我们可以从一条评论来取到对应的帖子通过使用 `comment.post`和取回一个帖子所有的评论通过使用`post.comments.all()`。如果你没有定义*related_name*属性，Django会使用这个model的命名加上*_set*（例如：comment_set）来命名关联对象读取这儿的manager。

你可以学习更多关于多对单的关系通过访问https://docs.djangoproject.com/en/1.8/topics/db/examples/many_to_one/

我们有包含一个*active*布尔字段用来手动使不好的评论无效不展示。我们使用*created*字段使评论默认根据创建时间来进行排序。

你刚创建的这个新的*Comment* model并没有同步到数据库中。运行以下命令通过新的model生成一个新的数据迁移：

    python manage.py makemigrations blog

你会看到如下输出：

    Migrations for 'blog':
        0002_comment.py:
            - Create model Comment

Django在博客应用下的*migrations/*目录下生成了一个*0002_comment.py*文件。现在你需要创建一个关联数据库模式并且应用这些改变到数据库中。运行以下命令来应用已经存在的数据迁移：

    python manage.py migrate

你会获取以下输出：

    Applying blog.0002_comment... OK
    
我们杠杆创建的数据迁移已经被执行，现在一张*blog_comment*表已经存在数据库中。

现在，我们可以添加我们新的model到管理站点中通过简单的接口来管理评论。打开博客应用下的*admin.py*文件，添加如下内容：

    from .models import Post, Comment
    class CommentAdmin(admin.ModelAdmin):
    list_display = ('name', 'email', 'post', 'created', 'active')
    list_filter = ('active', 'created', 'updated')
    search_fields = ('name', 'email', 'body')
    admin.site.register(Comment, CommentAdmin)
    
通过运行命令`python manage.py runserver`来启动开发服务病在浏览器中打开 http://127.0.0.1:8000/admin/ 。 你会看到新的model在**Blog**区域中出现，如下所示：
（图片）

我们的model已经被注册到管理站点我们可以使用简单的接口来管理评论实例。

##通过models创建表单
我们仍然需要创建一个表单可以让我们的用户在博客帖子下进行评论。请记住，Django有两个用来创建表单的基础类：Form和ModelForm。你已经使用过前者可以让用户通过email来分享帖子。在下面的例子中，你将需要使用ModelForm因为你必须从你的Comment model中创建一个动态的表单。编辑博客应用下的*forms.py*，添加如下代码：

    from .models import Comment
    class CommentForm(forms.ModelForm):
    class Meta:
    model = Comment
    fields = ('name', 'email', 'body')
    
从model中创建表单，我们只需要在这个表单的*Meta*类中声明使用哪个model来构建表单。Django将会解析model并为我们动态的创建表单。每一种model字段类型都有对应的默认表单字段类型。默认的，Django创建的表单包含model中包含的每个字段。当然，你可以明确的告诉框架你想在你的表单中包含哪些字段通过使用*fields*列，或者定义哪些字段你不需要的通过使用*exclude*列。对于我们的*CommentForm*,我们在表单中只需要*name*,*email*,和*body*字段，因为我们只需要用到这3个字段让我们的用户来填写。

##在views中操作ModelForms
我们会使用帖子的详情view来实例化表单，能更简单的处理它。编辑*models.py*文件，导入*Comment* modle和*CommentForm*表单，并且修改*post_detail* view如下所示：

    from .models import Post, Comment    from .forms import EmailPostForm, CommentForm    def post_detail(request, year, month, day, post):        post = get_object_or_404(Post, slug=post,                                      status='published',                                      publish__year=year,                                      publish__month=month,                                      publish__day=day)
        # List of active comments for this post        comments = post.comments.filter(active=True)
                if request.method == 'POST':            # A comment was posted            comment_form = CommentForm(data=request.POST)            if comment_form.is_valid():                # Create Comment object but don't save to database yet                new_comment = comment_form.save(commit=False)                # Assign the current post to the comment                new_comment.post = post                # Save the comment to the database                new_comment.save()        else:            comment_form = CommentForm()        return render(request,              'blog/post/detail.html',              {'post': post,
              'comments': comments, 
              'comment_form': comment_form})

让我们来回顾下我们刚才对对我的view添加了哪些操作。我们使用*post_detail* view来显示帖子和对应的评论。我们添加了一个QuerySet来显示这个帖子所有有效的评论：
    
    comments = post.comments.filter(active=True)
    
我们从*post*对象开始构建这个QuerySet。使用在*Comment* model中通过*related_name属性定义的关系啦返回所有有关系的对象给*comments*。

我们还在这个view中让我们的用户添加一条新的评论。如果调用这个view通过GET请求，就我们创建了一个表单实例通过使用`comment_fomr = commentForm()`。如果请求是一个已经完成验证的POST，我们实例化表单来使用提交的数据并且验证数据通过使用*is_valid()*方法。如果这个表单是无效的，我们会在template中渲染验证错误的信息。如果表单通过验证，我们会做以下的操作：

* 1. 我们创建一个新的*Comment*对象通过调用这个表单的*save()*方法，如下所示：
   
       new_comment = comment_form.save(commit=False)
       
   *saven()*方法创建了一个model的实例用来保存表单的数据到数据库中。如果你调用这个方法通过设置`comment=False`，你创建的model实例不会即时保存到数据中。这是非常方便的当你想在最终保存之前修改这个model对象，我们接下来将做这一步骤。*save()*方法是给*ModelForm*用的，但不是给表单实例们用的，因为它们没有关联上任何model。

* 2.我们分配一个帖子给我们刚才创建的评论：
    
        new_comment.post = post
    
    通过以上动作，我们指定新的评论是属于分配的帖子。

* 3.我们保存新的评论到数据库，如下所示：
    
        new_comment.save()
        
我们的view已经准备好显示和处理新的评论了。

##在帖子详情template中添加评论
我们为帖子创建了一个管理评论的功能。现在我们需要修改我们的*post_detail.html* template来适应这个功能，通过做到以下步骤：

* 显示这个帖子的评论总数
* 显示评论的列
* 显示一个表单给用户来添加新的评论

首先，我们来添加评论的总数。打开*blog_detail.html* template在*content*区块中添加如下代码：

    {% with comments.count as total_comments %}     <h2>       {{ total_comments }} comment{{ total_comments|pluralize }}     </h2>    {% endwith %}
    
我们在template中使用Django ORM执行`comments.count()` QuerySet。注意，在Django template语言中调用方法时不使用圆括号。`{% with %}` tag允许我们分配一个值给新的变量，这个变量可以一直使用直到遇到`{% endwith %}`。

> `{% whti %}` template tag是非常有用的可以避开直接使用数据库或花费大量的时间在处理复杂的方法。

我们使用*pluralize* template filter在单词*comment*的后面展示*total_comments*的复数形式。Template filters使输入的变量值经过应用然后返回计算后的值。我们将会更多的讨论tempalte filters在*第三章 扩展你的博客应用*中。

这*pluralize* template filter 会在值的末尾显示一个"s"如果值不为 1。在之前的文本将会渲染成类似： 0 comments, 1 comment 或者 N comments。Django内置大量的template tags 和 filters来帮助你通过各种方法展示各类信息。

现在，让我们加入评论列。在template中之前的代码后面加入以下内容：
    
    {% for comment in comments %}     <div class="comment">       <p class="info">         Comment {{ forloop.counter }} by {{ comment.name }}         {{ comment.created }}    </p>       {{ comment.body|linebreaks }}     </div>       {% empty %}     <p>There are no comments yet.</p>    {% endfor %}
    
我们使用`{% for %}` template tag来循环所有的评论。我们会显示一个默认的信息如果*comments*列为空，告诉我们的用户这篇帖子还没有任何评论。我们通过使用 `{{ forloop.counter }}`变量来枚举评论，该变量包含在每次迭代的循环计数中。之后我们显示发送评论的用户名，日期，和评论的内容。

最后，你需要渲染表单或者显示一条成功的信息来代替表单当表单提交成功后。在之前的代码后面添加如下内容：

    {% if new_comment %}    <h2>Your comment has been added.</h2>    {% else %}    <h2>Add a new comment</h2>    <form action="." method="post">       {{ comment_form.as_p }}       {% csrf_token %}       <p><input type="submit" value="Add comment"></p>    </form>    {% endif %}
    
这段代码非常简洁明了：如果存在*new_comment*对象，我们会展示一条成功信息因为成功创建了一条新评论。否则，我们通过一个段落`<p>`元素渲染表单中每一个字段，并且表明这个POST请求包含有CSRF标记。在浏览器中打开 http://127.0.0.1:8000/blog/ 然后点击任意一篇帖子的标题进入它的详情页面。你会看到如下页面展示：
（图片）

使用表单添加几条评论。这些评论会在你的帖子下面根据时间排序来展示，类似下图：
（图片）

在你的浏览器中打开http://127.0.0.1:8000/admin/blog/comment/ 。你会看到管理页面中存在你创建的评论列表。随便点击其中的一行来进行编辑，取消选择**Active**复选框，然后点击**Save**按钮。你会再次被重定向到评论列页面，刚才编辑的评论**Active**列将会显示一个不激活的图表。类似下图：
（图片）

##增加标签功能
在应用了我们的评论系统之后，我们准备创建一种方法来为我们的帖子打标签。我们准备在我们的项目中集成第三方的Django标签应用来使用。*django-taggit*是一个可重用的应用，它首先提供给你一个*Tag* model和一个manager用来方便的给每个model添加标签。你可以阅读到源码，通过访问 https://github.com/alex/django-taggit 。

首先，你需要通过pip安装django-taggit，运行以下命令：
    
    pip install django-taggit==0.17.1
    
之后打开*mysite*项目下的*settings.py*文件，在*INSTALLED_APPS*设置中设置如下：
    
    INSTALLED_APPS = (    # ...    'blog',    'taggit',    )
    
打开你的博客应用下的*model.py*文件添加django-taggit提供的*TaggableManager* manager给*Post* model，代码如下：
    
    from taggit.managers import TaggableManager    class Post(models.Model):        # ...        tags = TaggableManager()
        
这个tags manager允许你给*Post*对象添加，获取以及移除标签。

运行以下命令为你的model的改变创建一个数据迁移：

    python manage.py makemigrations blog

你会看到如下输出：
    
    Migrations for 'blog':     0003_post_tags.py:       - Add field tags to post

现在，运行以下代码在数据库中生成django-taggit model对应的表以及同步你的model的改变：

    python manage.py migrate
    
你会看到以下输出：
    
    Applying taggit.0001_initial... OK    Applying taggit.0002_auto_20150616_2121... OK    Applying blog.0003_post_tags... OK
    
你的数据库现在已经可以使用*django-taggit* model。打开终端运行命令` python manage.py shell`来学习如何使用*tags* manager。首先，我们通过ID匹配来取回我们的其中一篇帖子：
    
    >>> from blog.models import Post    >>> post = Post.objects.get(id=1)

之后为它添加一些tags并且取回tags来检查是否添加成功：

    >>> post.tags.add('music', 'jazz', 'django')    >>> post.tags.all()    [<Tag: jazz>, <Tag: django>, <Tag: music>]
    
最后，移除一个tag并且再次检查tags列：

    >>> post.tags.remove('django')    >>> post.tags.all()    [<Tag: jazz>, <Tag: music>]
    
非常简单，对吧？运行命令`python manage.py runserver`启动开发服务，在浏览器中打开  http://127.0.0.1:8000/admin/taggit/tag/ 。你会看到管理页面包含了*taggit*应用的*Tag*对象列：
（图片）

转到 http://127.0.0.1:8000/admin/blog/post/ 并点击一篇帖子进行编辑。你会看到帖子中包含了一个新的**Tags**字段如下所示，你可以非常容易的编辑它：
（图片）

现在，我们准备编辑我们的博客来显示这些tags。打开*blog/post/list.html* template在博客的标题下方添加如下代码：

    <p class="tags">Tags: {{ post.tags.all|join:", " }}</p>

*join* template filter的功能类似python的`join()`方法，将给予的元素串联在一起成为一个字符串。在浏览器中打开 http://127.0.0.1:8000/blog/ 。 你会看到tags在每一个帖子的标题下面：
（图片）

现在，让我们来编辑我们的*post_list* view让用户可以列出打上了特定tag的所有帖子。打开博客应用下的*views.py*文件，导入*Tag* model从*django-taggit*中，然后修改*post_list* view如下所示：

    from taggit.models import Tag    def post_list(request, tag_slug=None): object_list = Post.published.all() tag = None        if tag_slug:            tag = get_object_or_404(Tag, slug=tag_slug) 
            object_list =   object_list.filter(tags__in=[tag]) # ...


