# 第六章

中介模型（intermediate model）
查询集（QuerySets）
活动流（activity stream）

## 跟踪用户动作

在上一章中，你在你的项目中实现了AJAX视图（views），通过使用jQuery并创建了一个JavaScript书签在你的平台中分享别的网站的内容。

在本章中，你会学习如何创建一个粉丝系统以及创建一个用户活动流。你会发现Django信号（signals）的工作方式以及在你的项目中集成Redis快速 I/O 仓库用来存储 item 视图（views）。

本章将会覆盖以下几点：

* 通过一个中介模型创建多对对的关系
* 创建 AJAX 视图（views）
* 创建一个活动流应用
* 给模型（modes）添加通用关系
* 取回对象的最优查询集
* 使用信号（signals）给非规范化的计数
* 存储 item 视图（views）到 Redis 中

### 创建一个粉丝系统

我们将要在我们的项目中创建一个粉丝系统。我们的用户在平台中能够彼此关注并且跟踪其他用户的分享。这个关系在用户中的是多对多的关系，一个用户能够关注多个用户并且能被多个用户关注。

### 通过一个中介模型（intermediary model）创建多对对的关系

在上一章中，你创建了多对对关系通过在其中一个有关联的模型（model）上添加了一个*ManyToManyField*然后让Django为这个关系创建了数据库表。这种方式支持大部分的场景，但是有时候你需要为这种关系创建一个中介模型。创建一个中介模型是非常有必要的当你想要为当前关系存储额外的信息，例如当前关系创建的时间点或者一个描述当前关系类型的字段。

我们会创建一个中介模型用来在用户之间构建关系。有两个原因可以解释为什么我们要用一个中介模型：

* 我们使用Django提供的*user*模型（model）并且我们想要避免修改它。
* 我们想要存储关系建立的时间

编辑你的*account*应用中的*models.py*文件添加如下代码：

```python
from django.contrib.auth.models import User   class Contact(models.Model):       user_from = models.ForeignKey(User,                                     related_name='rel_from_set')       user_to = models.ForeignKey(User,                                   related_name='rel_to_set')       created = models.DateTimeField(auto_now_add=True,                                      db_index=True)       class Meta:           ordering = ('-created',)       def __str__(self):           return '{} follows {}'.format(self.user_from,self.user_to)
```

这个*Contact*模型我们将会给用户关系使用。它包含以下字段：

* user_form：一个*ForeignKey*指向创建关系的用户
* user_to：一个*ForeignKey*指向被关注的用户
* created：一个`auto_now_add=True`的*DateTimeField*字段用来存储关系创建时的时间

在*ForeignKey*字段上会自动生成一个数据库索引。我们使用`db_index=True`来创建一个数据库索引给*created*字段。这会提升查询执行的效率当通过这个字段对查询集进行排序的时候。

使用 ORM ，我们可以创建一个关系给一个用户 *user1* 关注另一个用户 *user2*，如下所示：

```python
user1 = User.objects.get(id=1)user2 = User.objects.get(id=2)Contact.objects.create(user_from=user1, user_to=user2)
```

关系管理器 *rel_form_set* 和 *rel_to_set* 会返回一个查询集给*Contace*模型（model）。为了
从*User*模型（model）中存取最终的关系侧，*Contace*模型（model）会期望*User*包含一个*ManyToManyField*，如下所示（译者注：以下代码是作者假设的，实际上*User*不会包含以下代码）：

```python
following = models.ManyToManyField('self',                                   through=Contact,                                   related_name='followers',                                   symmetrical=False)
```

在这个例子中，我们告诉Django去使用我们定制的中介模型来创建关系通过给*ManyToManyField*添加`through=Contact`。这是一个从*User*模型到本身的多对对关系：我们在*ManyToMnyfIELD*字段中引用 `'self'`来创建一个关系给相同的模型（model）。

> 当你在多对多关系中需要额外的字段，创建一个定制的模型（model)，一个关系侧就是一个*ForeignKey*。添加一个 *ManyToManyField* 在其中一个有关联的模型（models）中然后通过在*through*参数中包含该中介模型指示Django去使用你的定制中介模型。

如果*User*模型（model）是我们应用的一部分，我们可以添加以上的字段给模型（model）（译者注：所以说，上面的代码是作者假设存在）。但实际上，我们无法直接修改*User*类，因为它是属于*django.contrib.auth*应用的。我们将要做些轻微的改动，给*User*模型动态的添加这个字段。编辑*account*应用中的*model.py*文件，添加如下代码：

```python
# Add following field to User dynamicallyUser.add_to_class('following',                   models.ManyToManyField('self',                                          through=Contact,                                          related_name='followers',                                          symmetrical=False))
```

在以上代码中，我们使用Django模型（models）的`add_to_class()`方法给*User*模型（model）添加*monkey-patch*(译者注：猴子补丁 *Monkey patch* 就是在运行时对已有的代码进行修改，而不需要修改原始代码）。你需要意识到，我们不推荐使用`add_to_class()`为模型（models）添加字段。我们在这个场景中利用这种方法是因为以下的原因：

* 我们可以非常简单的取回关系对象使用Django ORM的`user.followers.all()`以及`user.following.all()`。我们使用中介（intermediary） *Contact* 模型（model）可以避免复杂的查询例如使用到额外的数据库操作*joins*，如果在我们的定制*Profile*模型（model）中定义过了关系。
* 这个多对多关系的表将会被创建通过使用*Contact*模型（model）。因此，动态的添加*ManyToManyField*将不会对Django *User* 模型（model）的数据库进行任意改变。
* 我们避免了创建一个定义的用户模型（model），保持了所有Django内置*User*的特性。

请记住，在大部分的场景中，在我们之前创建的*Profile*模型（model）添加字段是更好的方法，可以替代在*User*模型（model）上打上*monkey-patch*。Django还允许你使用定制的用户模型（models）。如果你想要使用你的定制用户模型（model），可以访问 https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#specifying-a-custom-user-model 获得更多信息。

你能看到上述代码中的关系包含了`symmetrical=Flase`来定义一个非对称（non-symmetric）关系。这表示如果我关注了你，你不会自动的关注我。

> 当你使用了一个中介模型给多对多关系，一些关系管理器的方法将不可用，例如：`add()`，`create()`以及`remove()`。你需要创建或删除中介模型的实例来代替。

运行如下命令来生成*account*应用的初始迁移：

    python manage.py makemigrations account
    
你会看到如下输出：

```python
Migrations for 'account':     0002_contact.py:       - Create model Contact
```

现在继续运行以下命令来同步应用到数据库中：

    python manage.py migrate account
    
你会看到如下内容包含在输出中：

    Applying account.0002_contact... OK
    
*Contact*模型（model）现在已经被同步进了数据库，我们可以在用户之间创建关系。但是，我们的网站还没有提供一个方法来浏览用户或查看详细的用户profile。让我们为*User*模型构建列表和详情视图（views）。

### 为用户profiles创建列表和详情视图（views）

打开*account*应用中的*views.py*文件添加如下代码：

```python
from django.shortcuts import get_object_or_404from django.contrib.auth.models import User   @login_required   def user_list(request):       users = User.objects.filter(is_active=True)       return render(request,                     'account/user/list.html',                     {'section': 'people',                      'users': users})   @login_required   def user_detail(request, username):       user = get_object_or_404(User,                                username=username,                                is_active=True)       return render(request,                     'account/user/detail.html',                     {'section': 'people',                      'user': user})
```

以上是*User*对象的简单列表和详情视图（views）。`user_list`视图（view）获得了所有的可用用户。Django *User* 模型（model）包含了一个标志（flag）`is_active`来指示用户账户是否可用。我们通过`is_active=True`来过滤查询只返回可用的用户。这个视图（vies）返回了所有结果，但是你可以改善它通过添加页码，这个方法我们在*image_list*视图（view）中使用过。

*user_detail*视图（view）使用`get_object_or_404()`快捷方法来返回所有可用的用户通过传入的用户名。当使用传入的用户名无法找到可用的用户这个视图（view）会返回一个HTTP 404响应。

编辑*account*应用的*urls.py*文件，为以上两个视图（views）添加URL模式，如下所示：

```python
urlpatterns = [       # ...       url(r'^users/$', views.user_list, name='user_list'),       url(r'^users/(?P<username>[-\w]+)/$',           views.user_detail,           name='user_detail'),]
```

我们会使用 `user_detail` URL模式来给用户生成规范的URL。你之前就在模型（model）中定义了一个`get_absolute_url()`方法来为每个对象返回规范的URL。另外一种方式为一个模型（model）指定一个URL是为你的项目添加*ABSOLUTE_URL_OVERRIDES*设置。

编辑项目中的*setting.py*文件，添加如下代码：

```python
ABSOLUTE_URL_OVERRIDES = {    'auth.user': lambda u: reverse_lazy('user_detail',                                        args=[u.username])}
```

Django会为所有出现在*ABSOLUTE_URL_OVERRIDES*设置中的模型（models）动态添加一个`get_absolute_url()`方法。这个方法会给设置中指定的模型返回规范的URL。我们给传入的用户返回*user_detail* URL。现在你可以在一个User实例上使用`get_absolute_url()`来取回他自身的规范URL。打开Python shell输入命令`python manage.py shell`运行以下代码来进行测试：

```
>>> from django.contrib.auth.models import User>>> user = User.objects.latest('id')>>> str(user.get_absolute_url())'/account/users/ellington/'
```

返回的URL如同期望的一样。我们需要为我们刚才创建的视图（views）创建模板（templates）。在*account*应用下的*templates/account/目录下添加以下目录和文件：

```
/user/
    detail.html
    list.html
```

编辑*account/user/list.html*模板（template）给它添加如下代码：

```html
{% extends "base.html" %}{% load thumbnail %}{% block title %}People{% endblock %}{% block content %}    <h1>People</h1>    <div id="people-list">       {% for user in users %}         <div class="user">            <a href="{{ user.get_absolute_url }}">             {% thumbnail user.profile.photo "180x180" crop="100%" as im %}               <img src="{{ im.url }}">             {% endthumbnail %}           </a>           <div class="info">             <a href="{{ user.get_absolute_url }}" class="title">               {{ user.get_full_name }}             </a> 
           </div>         </div>       {% endfor %}    </div>{% endblock %}
```

这个模板（template）允许我们在网站中排列所有可用的用户。我们对给予的用户进行迭代并且使用`{% thumbnail %}模板（template）标签（tag）来生成profile图片缩微图。

打开项目中的*base.html*模板（template），在以下菜单项的*href*属性中包含*user_list*URL：

```html
<li {% if section == "people" %}class="selected"{% endif %}>
    <a href="{% url "user_list" %}">People</a>
</li>
```

通过命令`python manage.py runserver`启动开发服务器然后在浏览器打开 http://127.0.0.1:8000/account/users/ 。你会看到如下所示的用户列：

![django-6-1](http://ohqrvqrlb.bkt.clouddn.com/django-6-1.png)

（译者注：图灵，特斯拉，爱因斯坦，都是大牛啊）

编辑*account*应用下的*account/user/detail.html*模板，添加如下代码：

```html
{% extends "base.html" %}{% load thumbnail %}{% block title %}{{ user.get_full_name }}{% endblock %}{% block content %}    <h1>{{ user.get_full_name }}</h1>    <div class="profile-info">    {% thumbnail user.profile.photo "180x180" crop="100%" as im %}        <img src="{{ im.url }}" class="user-detail">    {% endthumbnail %}    </div>    {% with total_followers=user.followers.count %}    <span class="count">        <span class="total">{{ total_followers }}</span>        follower{{ total_followers|pluralize }}    </span>    <a href="#" data-id="{{ user.id }}" data-action="{% if request.user in user.followers.all %}un{% endif %}follow" class="followbutton">        {% if request.user not in user.followers.all %}
            Follow
        {% else %}
            Unfollow
        {% endif %}
    </a>
    <div id="image-list" class="imget-container">
        {% include "images/image/list_ajax.html" with images = user.images_create.all %}
    </div>
    {% endwith %}
{% endblock %}
```

在详情模板（template）中我们展示用户profile并且我们使用`{% thumbnail %}`模板（template）标签（tag）来显示profile图片。我们显示粉丝的总数以及一个链接可以 *follow/unfollow* 该用户。我们会隐藏关注链接当用户在查看他们自己的profile，防止用户自己关注自己。我们会执行一个AJAX请求来 *follow/unfollow* 一个指定用户。我们给 `<a>` HTML元素添加*data-id*和*data-action*属性包含用户ID以及当该链接被点击的时候会执行的初始操作，*follow/unfollow* ，这个操作依赖当前页面的展示的用户是否已经被正在浏览的用户所关注。我们展示当前页面用户的图片书签通过*list_ajax.html*模板。

再次打开你的浏览器，点击一个拥有图片书签的用户链接，你会看到一个profile详情如下所示：

![django-6-2](http://ohqrvqrlb.bkt.clouddn.com/django-6-2.png)

### 创建一个AJAX视图（view）来关注用户

我们将会创建一个简单的视图（view）使用AJAX来 *follow/unfollow* 用户。编辑*account*应用中的*views.py*文件添加如下代码：

```python
from django.http import JsonResponsefrom django.views.decorators.http import require_POSTfrom common.decorators import ajax_requiredfrom .models import Contact@ajax_required@require_POST@login_requireddef user_follow(request):    user_id = request.POST.get('id')    action = request.POST.get('action')    if user_id and action:        try:            user = User.objects.get(id=user_id)            if action == 'follow':                Contact.objects.get_or_create(                    user_from=request.user,                    user_to=user)            else:                Contact.objects.filter(user_from=request.user,                                        user_to=user).delete()            return JsonResponse({'status':'ok'})        except User.DoesNotExist:            return JsonResponse({'status':'ko'})    return JsonResponse({'status':'ko'})
```

*user_follow*视图（view）有点类似与我们之前创建的*image_like*视图（view）。因为我们使用了一个定制中介模型给用户的多对多关系，所以*ManyToManyField*管理器默认的`add()`和`remove()`方法将不可用。我们使用中介*Contact*模型（model）来创建或删除用户关系。

在*account*应用中的*urls.py*文件中导入你刚才创建的视图（view）然后为它添加URL模式：

```python
    url(r'^users/follow/$', views.user_follow, name='user_follow'),
```

请确保你放置的这个URL模式的位置在*user_detail*URL模式之前。否则，任何对 */users/follow/* 的请求都会被*user_detail*模式给正则匹配然后执行。请记住，每一次的HTTP请求Django都会对每一条存在的URL模式进行匹配直到第一条匹配成功才会停止继续匹配。

编辑*account*应用下的*user/detail.html*模板添加如下代码：

```javascript
{% block domready %}     $('a.follow').click(function(e){       e.preventDefault();       $.post('{% url "user_follow" %}',         {           id: $(this).data('id'),           action: $(this).data('action')         },         function(data){           if (data['status'] == 'ok') {             var previous_action = $('a.follow').data('action');
                          // toggle data-action             $('a.follow').data('action',               previous_action == 'follow' ? 'unfollow' : 'follow');             // toggle link text             $('a.follow').text(               previous_action == 'follow' ? 'Unfollow' : 'Follow');
                            // update total followers             var previous_followers = parseInt(               $('span.count .total').text());             $('span.count .total').text(previous_action == 'follow' ? previous_followers + 1 : previous_followers - 1);          }
        }      });    });{% endblock %}
```

这段JavaScript代码执行AJAX请求来关注或不关注一个指定用户并且触发 *follow/unfollow* 链接。我们使用jQuery去执行AJAX请求的同时会设置 *follow/unfollow* 两种链接的*data-aciton*属性以及HTML`<a>`元素的文本基于它上一次的值。当AJAX操作执行完成，我们还会对显示在页面中的粉丝总数进行更新。打开一个存在的用户的详情页面，然后点击**Follow**链接尝试下我们刚才构建的功能是否正常。

### 创建一个通用的活动流应用

许多社交网站会给他们的用户显示一个活动流，这样他们可以跟踪其他用户在平台中的操作。一个活动流是一个用户或一个用户组最近活动的列表。举个例子，*Facebook*的*News Feed*就是一个活动流。用户X给Y图片打上了书签或者用户X关注了用户Y也是例子操作。我们将会构建一个活动流应用这样每个用户都能看到他关注的用户最近进行的交互。为了做到上述功能，我们需要一个模型（modes）来保存用户在网站上的操作执行，还需要一个简单的方法来添加操作给feed。

运行以下命令在你的项目中创建一个新的应用命名为*actions*：

```python
django-admin startapp actions
```

在你的项目中的*settings.py*文件中的*INSTALLED_APPS*设置中添加*'actions'*，这样可以让Django知道这个新的应用是可用状态：

```python
INSTALLED_APPS = (
    # ...
    'actions',    
)
```

编辑*account*应用下的*models.py*文件添加如下代码：

```python
from django.db import modelsfrom django.contrib.auth.models import User
class Action(models.Model):    user = models.ForeignKey(User,                            related_name='actions',                            db_index=True)    verb = models.CharField(max_length=255)    created = models.DateTimeField(auto_now_add=True,                                    db_index=True)
                                        class Meta:        ordering = ('-created',)
```

这个*Action*模型（model）将会用来记录用户的活动。模型（model）中的字段解释如下：

* user：执行该操作的用户。这个一个指向Django *User*模型（model）的 *ForeignKey*。
* verb：这是用户执行操作的动作描述。
* created：这个时间日期会在动作执行的时候创建。我们使用`auto_now_add=True`来动态设置它为当前的时间当这个对象第一次被保存在数据库中。

通过这个基础模型（model），我们只能够存储操作例如用户X做了哪些事情。我们需要一个额外的*ForeignKey*字段为了保存操作会涉及到的一个*target*（目标）对象，例如用户X给图片Y打上了暑期那或者用户X现在关注了用户Y。就像你之前知道的，一个普通的*ForeignKey*只能指向一个其他的模型（model）。但是，我们需要一个方法，可以让操作的*target*（目标）对象是任何一个已经存在的模型（model）的实例。这个场景就由Django内容类型框架来上演。

### 使用内容类型框架

Django包含了一个内容类型框架位于*django.contrib.contenttypes*。这个应用可以跟踪你的项目中所有的模型（models）以及提供一个通用接口来与你的模型（models）进行交互。

当你使用*startproject*命令创建一个新的项目的时候这个*contenttypes*应用就被默认包含在*INSTALLED_APPS*设置中。它被其他的*contrib*包使用，例如认证(authentication）框架以及admin应用。

*contenttypes*应用包含一个*ContentType*模型（model）。这个模型（model）的实例代表了你的应用中真实存在的模型（models），并且新的*ContentTYpe*实例会动态的创建当新的模型（models）安装在你的项目中。*ContentType*模型（model）有以下字段：

* app_label：模型（model）属于的应用名，它会自动从模型（model）*Meta*选项中的*app_label*属性获取到。举个例子：我们的*Image*模型（model）属于*images*应用
* model：模型（model）类的名字
* name：模型的可读名，它会自动从模型（model）*Meta*选项中的*verbose_name*获取到。

让我们看一下我们如何实例化*ContentType*对象。打开Python终端使用`python manage.py shell`命令。你可以获取一个指定模型（model）对应的*ContentType*对象通过执行一个带有*app_label*和*model*属性的查询，例如：

```python
>>> from django.contrib.contenttypes.models import ContentType>>> image_type = ContentType.objects.get(app_label='images',model='image')>>> image_type<ContentType: image>
```

你还能反过来获取到模型（model）类从一个*ContentType*对象中通过调用它的`model_class()`方法：

```python
>>> from images.models import Image>>> ContentType.objects.get_for_model(Image)<ContentType: image>
```

以上就是内容类型的一些例子。Django提供了更多的方法来使用他们进行工作。你可以访问 https://docs.djangoproject.com/en/1.8/ref/contrib/contenttypes/ 找到关于内容类型框架的官方文档。

### 添加通用的关系给你的模型（models）

在通用关系中*ContentType*对象扮演指向模型（model）的角色被关联所使用。你需要3个字段在模型（model）中组织一个通用关系：






