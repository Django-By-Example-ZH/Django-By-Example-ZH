书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

（译者注：无他，祝大家年会都中奖！）
# 第六章

## 跟踪用户动作

在上一章中，你在你的项目中实现了AJAX视图（views），通过使用jQuery并创建了一个JavaScript书签在你的平台中分享别的网站的内容。

在本章中，你会学习如何创建一个粉丝系统以及创建一个用户活动流（activity stream）。你会学习到Django信号（signals）的工作方式以及在你的项目中集成Redis快速 I/O 仓库用来存储视图（views）项。

本章将会覆盖以下几点：

* 通过一个中介模型（intermediate model）创建多对对的关系
* 创建 AJAX 视图（views）
* 创建一个活动流（activity stream）应用
* 给模型（modes）添加通用关系
* 取回对象的最优查询集（QuerySets）
* 使用信号（signals）给非规范化的计数
* 存储视图（views）项到 Redis 中

### 创建一个粉丝系统

我们将要在我们的项目中创建一个粉丝系统。我们的用户在平台中能够彼此关注并且跟踪其他用户的分享。这个关系在用户中的是多对多的关系，一个用户能够关注多个用户并且能被多个用户关注。

### 通过一个中介模型（intermediate model）（intermediary model）创建多对对的关系

在上一章中，你创建了多对对关系通过在其中一个有关联的模型（model）上添加了一个*ManyToManyField*然后让Django为这个关系创建了数据库表。这种方式支持大部分的场景，但是有时候你需要为这种关系创建一个中介模型（intermediate model）。创建一个中介模型（intermediate model）是非常有必要的当你想要为当前关系存储额外的信息，例如当前关系创建的时间点或者一个描述当前关系类型的字段。

我们会创建一个中介模型（intermediate model）用来在用户之间构建关系。有两个原因可以解释为什么我们要用一个中介模型（intermediate model）：

* 我们使用Django提供的*user*模型（model）并且我们想要避免修改它。
* 我们想要存储关系建立的时间

编辑你的*account*应用中的*models.py*文件添加如下代码：

```python
from django.contrib.auth.models import User
   class Contact(models.Model):
       user_from = models.ForeignKey(User,
                                     related_name='rel_from_set')
       user_to = models.ForeignKey(User,
                                   related_name='rel_to_set')
       created = models.DateTimeField(auto_now_add=True,
                                      db_index=True)
       class Meta:
           ordering = ('-created',)
       def __str__(self):
           return '{} follows {}'.format(self.user_from,
self.user_to)
```

这个*Contact*模型我们将会给用户关系使用。它包含以下字段：

* user_form：一个*ForeignKey*指向创建关系的用户
* user_to：一个*ForeignKey*指向被关注的用户
* created：一个`auto_now_add=True`的*DateTimeField*字段用来存储关系创建时的时间

在*ForeignKey*字段上会自动生成一个数据库索引。我们使用`db_index=True`来创建一个数据库索引给*created*字段。这会提升查询执行的效率当通过这个字段对查询集（QuerySets）进行排序的时候。

使用 ORM ，我们可以创建一个关系给一个用户 *user1* 关注另一个用户 *user2*，如下所示：

```python
user1 = User.objects.get(id=1)
user2 = User.objects.get(id=2)
Contact.objects.create(user_from=user1, user_to=user2)
```

关系管理器 *rel_form_set* 和 *rel_to_set* 会返回一个查询集（QuerySets）给*Contace*模型（model）。为了
从*User*模型（model）中存取最终的关系侧，*Contace*模型（model）会期望*User*包含一个*ManyToManyField*，如下所示（译者注：以下代码是作者假设的，实际上*User*不会包含以下代码）：

```python
following = models.ManyToManyField('self',
                                   through=Contact,
                                   related_name='followers',
                                   symmetrical=False)
```

在这个例子中，我们告诉Django去使用我们定制的中介模型（intermediate model）来创建关系通过给*ManyToManyField*添加`through=Contact`。这是一个从*User*模型到本身的多对对关系：我们在*ManyToMnyfIELD*字段中引用 `'self'`来创建一个关系给相同的模型（model）。

> 当你在多对多关系中需要额外的字段，创建一个定制的模型（model)，一个关系侧就是一个*ForeignKey*。添加一个 *ManyToManyField* 在其中一个有关联的模型（models）中然后通过在*through*参数中包含该中介模型（intermediate model）指示Django去使用你的定制中介模型（intermediate model）。

如果*User*模型（model）是我们应用的一部分，我们可以添加以上的字段给模型（model）（译者注：所以说，上面的代码是作者假设存在）。但实际上，我们无法直接修改*User*类，因为它是属于*django.contrib.auth*应用的。我们将要做些轻微的改动，给*User*模型动态的添加这个字段。编辑*account*应用中的*model.py*文件，添加如下代码：

```python
# Add following field to User dynamically
User.add_to_class('following',
                   models.ManyToManyField('self',
                                          through=Contact,
                                          related_name='followers',
                                          symmetrical=False))
```

在以上代码中，我们使用Django模型（models）的`add_to_class()`方法给*User*模型（model）添加*monkey-patch*(译者注：猴子补丁 *Monkey patch* 就是在运行时对已有的代码进行修改，而不需要修改原始代码）。你需要意识到，我们不推荐使用`add_to_class()`为模型（models）添加字段。我们在这个场景中利用这种方法是因为以下的原因：

* 我们可以非常简单的取回关系对象使用Django ORM的`user.followers.all()`以及`user.following.all()`。我们使用中介（intermediary） *Contact* 模型（model）可以避免复杂的查询例如使用到额外的数据库操作*joins*，如果在我们的定制*Profile*模型（model）中定义过了关系。
* 这个多对多关系的表将会被创建通过使用*Contact*模型（model）。因此，动态的添加*ManyToManyField*将不会对Django *User* 模型（model）的数据库进行任意改变。
* 我们避免了创建一个定义的用户模型（model），保持了所有Django内置*User*的特性。

请记住，在大部分的场景中，在我们之前创建的*Profile*模型（model）添加字段是更好的方法，可以替代在*User*模型（model）上打上*monkey-patch*。Django还允许你使用定制的用户模型（models）。如果你想要使用你的定制用户模型（model），可以访问 https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#specifying-a-custom-user-model 获得更多信息。

你能看到上述代码中的关系包含了`symmetrical=Flase`来定义一个非对称（non-symmetric）关系。这表示如果我关注了你，你不会自动的关注我。

> 当你使用了一个中介模型（intermediate model）给多对多关系，一些关系管理器的方法将不可用，例如：`add()`，`create()`以及`remove()`。你需要创建或删除中介模型（intermediate model）的实例来代替。

运行如下命令来生成*account*应用的初始迁移：

    python manage.py makemigrations account
    
你会看到如下输出：

```python
Migrations for 'account':
     0002_contact.py:
       - Create model Contact
```

现在继续运行以下命令来同步应用到数据库中：

    python manage.py migrate account
    
你会看到如下内容包含在输出中：

    Applying account.0002_contact... OK
    
*Contact*模型（model）现在已经被同步进了数据库，我们可以在用户之间创建关系。但是，我们的网站还没有提供一个方法来浏览用户或查看详细的用户profile。让我们为*User*模型构建列表和详情视图（views）。

### 为用户profiles创建列表和详情视图（views）

打开*account*应用中的*views.py*文件添加如下代码：

```python
from django.shortcuts import get_object_or_404
from django.contrib.auth.models import User
   @login_required
   def user_list(request):
       users = User.objects.filter(is_active=True)
       return render(request,
                     'account/user/list.html',
                     {'section': 'people',
                      'users': users})
   @login_required
   def user_detail(request, username):
       user = get_object_or_404(User,
                                username=username,
                                is_active=True)
       return render(request,
                     'account/user/detail.html',
                     {'section': 'people',
                      'user': user})
```

以上是*User*对象的简单列表和详情视图（views）。`user_list`视图（view）获得了所有的可用用户。Django *User* 模型（model）包含了一个标志（flag）`is_active`来指示用户账户是否可用。我们通过`is_active=True`来过滤查询只返回可用的用户。这个视图（vies）返回了所有结果，但是你可以改善它通过添加页码，这个方法我们在*image_list*视图（view）中使用过。

*user_detail*视图（view）使用`get_object_or_404()`快捷方法来返回所有可用的用户通过传入的用户名。当使用传入的用户名无法找到可用的用户这个视图（view）会返回一个HTTP 404响应。

编辑*account*应用的*urls.py*文件，为以上两个视图（views）添加URL模式，如下所示：

```python
urlpatterns = [
       # ...
       url(r'^users/$', views.user_list, name='user_list'),
       url(r'^users/(?P<username>[-\w]+)/$',
           views.user_detail,
           name='user_detail'),
]
```

我们会使用 `user_detail` URL模式来给用户生成规范的URL。你之前就在模型（model）中定义了一个`get_absolute_url()`方法来为每个对象返回规范的URL。另外一种方式为一个模型（model）指定一个URL是为你的项目添加*ABSOLUTE_URL_OVERRIDES*设置。

编辑项目中的*setting.py*文件，添加如下代码：

```python
ABSOLUTE_URL_OVERRIDES = {
    'auth.user': lambda u: reverse_lazy('user_detail',
                                        args=[u.username])
}
```

Django会为所有出现在*ABSOLUTE_URL_OVERRIDES*设置中的模型（models）动态添加一个`get_absolute_url()`方法。这个方法会给设置中指定的模型返回规范的URL。我们给传入的用户返回*user_detail* URL。现在你可以在一个User实例上使用`get_absolute_url()`来取回他自身的规范URL。打开Python shell输入命令`python manage.py shell`运行以下代码来进行测试：

```
>>> from django.contrib.auth.models import User
>>> user = User.objects.latest('id')
>>> str(user.get_absolute_url())
'/account/users/ellington/'
```

返回的URL如同期望的一样。我们需要为我们刚才创建的视图（views）创建模板（templates）。在*account*应用下的*templates/account/目录下添加以下目录和文件：

```
/user/
    detail.html
    list.html
```

编辑*account/user/list.html*模板（template）给它添加如下代码：

```html
{% extends "base.html" %}
{% load thumbnail %}
{% block title %}People{% endblock %}
{% block content %}
    <h1>People</h1>
    <div id="people-list">
       {% for user in users %}
         <div class="user">
            <a href="{{ user.get_absolute_url }}">
             {% thumbnail user.profile.photo "180x180" crop="100%" as im %}
               ![]({{ im.url }})
             {% endthumbnail %}
           </a>
           <div class="info">
             <a href="{{ user.get_absolute_url }}" class="title">
               {{ user.get_full_name }}
             </a> 
           </div>
         </div>
       {% endfor %}
    </div>
{% endblock %}
```

这个模板（template）允许我们在网站中排列所有可用的用户。我们对给予的用户进行迭代并且使用`{% thumbnail %}模板（template）标签（tag）来生成profile图片缩微图。

打开项目中的*base.html*模板（template），在以下菜单项的*href*属性中包含*user_list*URL：

```html
<li {% if section == "people" %}class="selected"{% endif %}>
    <a href="{% url "user_list" %}">People</a>
</li>
```

通过命令`python manage.py runserver`启动开发服务器然后在浏览器打开 http://127.0.0.1:8000/account/users/ 。你会看到如下所示的用户列：

![django-6-1](http://upload-images.jianshu.io/upload_images/3966530-d2b101ca88a607d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（译者注：图灵，特斯拉，爱因斯坦，都是大牛啊）

编辑*account*应用下的*account/user/detail.html*模板，添加如下代码：

```html

{% extends "base.html" %}
{% load thumbnail %}
{% block title %}{{ user.get_full_name }}{% endblock %}
{% block content %}
    <h1>{{ user.get_full_name }}</h1>
    <div class="profile-info">
    {% thumbnail user.profile.photo "180x180" crop="100%" as im %}
        ![]({{ im.url }})
    {% endthumbnail %}
    </div>
    {% with total_followers=user.followers.count %}
    <span class="count">
        <span class="total">{{ total_followers }}</span>
        follower{{ total_followers|pluralize }}
    </span>
    <a href="#" data-id="{{ user.id }}" data-action="{% if request.user in user.followers.all %}un{% endif %}follow" class="followbutton">
        {% if request.user not in user.followers.all %}
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

![django-6-2](http://upload-images.jianshu.io/upload_images/3966530-6dffd599f25f35aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 创建一个AJAX视图（view）来关注用户

我们将会创建一个简单的视图（view）使用AJAX来 *follow/unfollow* 用户。编辑*account*应用中的*views.py*文件添加如下代码：

```python
from django.http import JsonResponse
from django.views.decorators.http import require_POST
from common.decorators import ajax_required
from .models import Contact
@ajax_required
@require_POST
@login_required
def user_follow(request):
    user_id = request.POST.get('id')
    action = request.POST.get('action')
    if user_id and action:
        try:
            user = User.objects.get(id=user_id)
            if action == 'follow':
                Contact.objects.get_or_create(
                    user_from=request.user,
                    user_to=user)
            else:
                Contact.objects.filter(user_from=request.user,
                                        user_to=user).delete()
            return JsonResponse({'status':'ok'})
        except User.DoesNotExist:
            return JsonResponse({'status':'ko'})
    return JsonResponse({'status':'ko'})
```

*user_follow*视图（view）有点类似与我们之前创建的*image_like*视图（view）。因为我们使用了一个定制中介模型（intermediate model）给用户的多对多关系，所以*ManyToManyField*管理器默认的`add()`和`remove()`方法将不可用。我们使用中介*Contact*模型（model）来创建或删除用户关系。

在*account*应用中的*urls.py*文件中导入你刚才创建的视图（view）然后为它添加URL模式：

```python
    url(r'^users/follow/$', views.user_follow, name='user_follow'),
```

请确保你放置的这个URL模式的位置在*user_detail*URL模式之前。否则，任何对 */users/follow/* 的请求都会被*user_detail*模式给正则匹配然后执行。请记住，每一次的HTTP请求Django都会对每一条存在的URL模式进行匹配直到第一条匹配成功才会停止继续匹配。

编辑*account*应用下的*user/detail.html*模板添加如下代码：

```javascript
{% block domready %}
     $('a.follow').click(function(e){
       e.preventDefault();
       $.post('{% url "user_follow" %}',
         {
           id: $(this).data('id'),
           action: $(this).data('action')
         },
         function(data){
           if (data['status'] == 'ok') {
             var previous_action = $('a.follow').data('action');
             
             // toggle data-action
             $('a.follow').data('action',
               previous_action == 'follow' ? 'unfollow' : 'follow');
             // toggle link text
             $('a.follow').text(
               previous_action == 'follow' ? 'Unfollow' : 'Follow');
               
             // update total followers
             var previous_followers = parseInt(
               $('span.count .total').text());
             $('span.count .total').text(previous_action == 'follow' ? previous_followers + 1 : previous_followers - 1);
          }
        }
      });
    });
{% endblock %}
```

这段JavaScript代码执行AJAX请求来关注或不关注一个指定用户并且触发 *follow/unfollow* 链接。我们使用jQuery去执行AJAX请求的同时会设置 *follow/unfollow* 两种链接的*data-aciton*属性以及HTML`<a>`元素的文本基于它上一次的值。当AJAX操作执行完成，我们还会对显示在页面中的粉丝总数进行更新。打开一个存在的用户的详情页面，然后点击**Follow**链接尝试下我们刚才构建的功能是否正常。

### 创建一个通用的活动流（activity stream）应用

许多社交网站会给他们的用户显示一个活动流（activity stream），这样他们可以跟踪其他用户在平台中的操作。一个活动流（activity stream）是一个用户或一个用户组最近活动的列表。举个例子，*Facebook*的*News Feed*就是一个活动流（activity stream）。用户X给Y图片打上了书签或者用户X关注了用户Y也是例子操作。我们将会构建一个活动流（activity stream）应用这样每个用户都能看到他关注的用户最近进行的交互。为了做到上述功能，我们需要一个模型（modes）来保存用户在网站上的操作执行，还需要一个简单的方法来添加操作给feed。

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

编辑*actions*应用下的*models.py*文件添加如下代码：

```python
from django.db import models
from django.contrib.auth.models import User

class Action(models.Model):
    user = models.ForeignKey(User,
                            related_name='actions',
                            db_index=True)
    verb = models.CharField(max_length=255)
    created = models.DateTimeField(auto_now_add=True,
                                    db_index=True)
                                    
    class Meta:
        ordering = ('-created',)
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
>>> from django.contrib.contenttypes.models import ContentType
>>> image_type = ContentType.objects.get(app_label='images',model='image')
>>> image_type
<ContentType: image>
```

你还能反过来获取到模型（model）类从一个*ContentType*对象中通过调用它的`model_class()`方法：

```python
>>> from images.models import Image
>>> ContentType.objects.get_for_model(Image)
<ContentType: image>
```

以上就是内容类型的一些例子。Django提供了更多的方法来使用他们进行工作。你可以访问 https://docs.djangoproject.com/en/1.8/ref/contrib/contenttypes/ 找到关于内容类型框架的官方文档。

### 添加通用的关系给你的模型（models）

在通用关系中*ContentType*对象扮演指向模型（model）的角色被关联所使用。你需要3个字段在模型（model）中组织一个通用关系：

* 一个*ForeignKey*字段*ContentType*。这个字段会告诉我们给这个关联的模型（model）。
* 一个字段用来存储被关联对象的primary key。这个字段通常是一个*PositiveIntegerField*用来匹配Django自动的primary key字段。
* 一个字段用来定义和管理通用关系通过使用前面的两个字段。内容类型框架提供一个*GenericForeignKey*字段来完成这个目标。

编辑*actions*应用的*models.py*文件，添加如下代码：

```python
from django.db import models
from django.contrib.auth.models import User
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey
class Action(models.Model):
    user = models.ForeignKey(User,
                             related_name='actions',
                             db_index=True)
    verb = models.CharField(max_length=255)
    target_ct = models.ForeignKey(ContentType,
                                  blank=True,
                                  null=True,
                                  related_name='target_obj')
    target_id = models.PositiveIntegerField(null=True,
                                            blank=True,
                                            db_index=True)
    target = GenericForeignKey('target_ct', 'target_id')
    created = models.DateTimeField(auto_now_add=True,
                                   db_index=True)
    class Meta:
        ordering = ('-created',)
```

我们给*Action*模型添加了以下字段：

* target_ct：一个*ForeignKey*字段指向*ContentType*模型（model）。
* target_id：一个*PositiveIntegerField*用来存储被关联对象的primary key。
* target：一个*GenericForeignKey*字段指向被关联的对象基于前面两个字段的组合之上。

Django没有创建任何字段在数据库中给*GenericForeignKey*字段。只有*target_ct*和*target_id*两个字段被映射到数据库字段。两个字段都有`blank=True`和`null=True`属性所以一个*target*（目标）对象不是必须的当保存*Action*对象的时候。

> 你可以让你的应用更加灵活通过使用通用关系替代外键当它对拥有一个通用关系有意义。

运行以下命令来创建初始迁移为这个应用：

    python manage.py makemigrations actions

你会看到如下输出：

```python
    Migrations for 'actions':
        0001_initial.py:
            - Create model Action
```

接着，运行下一条命令来同步应用到数据库中：

    python manage.py migrate

这条命令的输出表明新的迁移已经被应用：

    Applying actions.0001_initial... OK

让我们在管理站点中添加*Action*模型（model）。编辑*actions*应用的*admin.py*文件，添加如下代码：

```python
from django.contrib import admin
from .models import Action

class ActionAdmin(admin.ModelAdmin):
    list_display = ('user', 'verb', 'target', 'created')
    list_filter = ('created',)
    search_fields = ('verb',)
    
admin.site.register(Action, ActionAdmin)
```

你已经将*Action*模型（model）注册到了管理站点中。运行命令`python manage.py runserver`来初始化开发服务器然后在浏览器中打开 http://127.0.0.1:8000/admin/actions/action/add/ 。你会看到如下页面可以创建一个新的*Action*对象：

![django-6-3](http://upload-images.jianshu.io/upload_images/3966530-365fd0854708ae28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如你所见，只有*target_ct*和*target_id*两个字段是映射为真实的数据库字段显示，并且*GenericForeignKey*字段不在这儿出现。*target_ct*允许你选择任何一个在你的Django项目中注册的模型（models）。你可以限制内容类型从一个限制的模型（models）集合中选择通过在*target-ct*字段中使用*limit_choices_to*属性：*limit_choices_to*属性允许你限制*ForeignKey*字段的内容通过给予一个特定值的集合。

在*actions*应用目录下创建一个新的文件命名为*utils.py*。我们会定义一个快捷函数，该函数允许我们使用一种简单的方式创建新的*Action*对象。编辑这个新的文件添加如下代码给它：

```python
from django.contrib.contenttypes.models import ContentType
from .models import Action
def create_action(user, verb, target=None):
    action = Action(user=user, verb=verb, target=target)
    action.save()
```

`create_action()`函数允许我们创建*actions*，该*actions*可以包含一个*target*对象或不包含。我们可以使用这个函数在我们代码的任何地方添加新的*actions*给活动流（activity stream）。

### 在活动流（activity stream）中避免重复的操作

有时候你的用户可能多次执行同个动作。他们可能在短时间内多次点击 *like/unlike* 按钮或者多次执行同样的动作。这会导致你停止存储和显示重复的动作。为了避免这种情况我们需要改善`create_action()`函数来避免大部分的重复动作。

编辑*actions*应用中的*utils.py*文件使它看上去如下所示：

```python
import datetime
from django.utils import timezone
from django.contrib.contenttypes.models import ContentType
from .models import Action

def create_action(user, verb, target=None):
    # check for any similar action made in the last minute
    now = timezone.now()
    last_minute = now - datetime.timedelta(seconds=60)
    similar_actions = Action.objects.filter(user_id=user.id,
                                            verb= verb,
                                        timestamp__gte=last_minute)
    if target:
        target_ct = ContentType.objects.get_for_model(target)
        similar_actions = similar_actions.filter(
                                            target_ct=target_ct,
                                            target_id=target.id)
    if not similar_actions:
        # no existing actions found
        action = Action(user=user, verb=verb, target=target)
        action.save()
        return True
    return False
```

我们通过修改`create_action()`函数来避免保存重复的动作并且返回一个布尔值来告诉该动作是否保存。下面来解释我们是如何避免重复动作的：

* 首先，我们通过Django提供的`timezone.now()`方法来获取当前时间。这个方法同`datetime.datetime.now()相同，但是返回的是一个*timezone-aware*对象。Django提供一个设置叫做*USE_TZ*用来启用或关闭时区的支持。通过使用*startproject*命令创建的默认*settings.py*包含`USE_TZ=True`。
* 我们使用*last_minute*变量来保存一分钟前的时间，然后我们取回用户从那以后执行的任意一个相同操作。
* 我们会创建一个*Action*对象如果在最后的一分钟内没有存在同样的动作。我们会返回*True*如果一个*Action*对象被创建，否则返回*False*。

### 添加用户动作给活动流（activity stream）
是时候添加一些动作给我们的视图(views)来给我的用户构建活动流（activity stream）了。我们将要存储一个动作为以下的每一个实例：

* 一个用户给某张图片打上书签
* 一个用户喜欢或不喜欢某张图片
* 一个用户创建一个账户
* 一个用户关注或不关注某个用户

编辑*images*应用下的*views.py*文件添加以下导入：

    from actions.utils import create_action
    
在*image_create*视图（view）中，在保存图片之后添加`create-action()`，如下所示：

```python
new_item.save()
create_action(request.user, 'bookmarked image', new_item)
```

在*image_like*视图（view）中，在添加用户给*users_like*关系之后添加`create_action()`，如下所示：

```python
image.users_like.add(request.user)
create_action(request.user, 'likes', image)
```

现在编辑*account*应用中的*view.py*文件添加以下导入：

    from actions.utils import create_action
    
在*register*视图（view）中，在创建*Profile*对象之后添加`create-action()`，如下所示：

```python
new_user.save()
profile = Profile.objects.create(user=new_user)
create_action(new_user, 'has created an account')
```

在*user_follow*视图（view）中添加`create_action()`，如下所示：

```python
Contact.objects.get_or_create(user_from=request.user,user_to=user)
create_action(request.user, 'is following', user)
```

就像你所看到的，感谢我们的*Action*模型（model）和我们的帮助函数，现在保存新的动作给活动流（activity stream）是非常简单的。

### 显示活动流（activity stream）

最后，我们需要一种方法来给每个用户显示活动流（activity stream）。我们将会在用户的dashboard中包含活动流（activity stream）。编辑*account*应用的*views.py*文件。导入*Action*模型然后修改*dashboard*视图（view）如下所示：

```python
from actions.models import Action

@login_required
def dashboard(request):
    # Display all actions by default
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list('id',flat=True)
    if following_ids:
        # If user is following others, retrieve only their actions
        actions = actions.filter(user_id__in=following_ids)
    actions = actions[:10]
    
    return render(request,
                  'account/dashboard.html',
                  {'section': 'dashboard',
                    'actions': actions})
```

在这个视图（view），我们从数据库取回所有的动作（actions），不包含当前用户执行的动作。如果当前用户还没有关注过任何人，我们展示在平台中的其他用户的最新动作执行。这是一个默认的行为当当前用户还没有关注过任何其他的用户。如果当前用户已经关注了其他用户，我们就限制查询只显示当前用户关注的用户的动作执行。最后，我们限制结果只返回最前面的10个动作。我们在这儿并不使用`order_by()`，因为我们依赖之前已经在*Action*模型（model）的*Meta*的排序选项。最新的动作会首先返回，因为我们在*Action*模型（model）中设置过`ordering = ('-created',)`。

## 优化涉及被关联的对想的查询集（QuerySets）

每次你取回一个*Aciton*对象，你都可能存取它的有关联的*User*对象，
并且可能这个用户也关联它的*Profile*对象。Django ORM提供了一个简单的方式一次性取回有关联的对象，避免对数据库进行额外的查询。

### 使用*select_related*

Django提供了一个叫做`select_related()`的查询集（QuerySets）方法允许你取回关系为一对多的关联对象。该方法将会转化成一个单独的，更加复杂的查询集（QuerySets），但是你可以避免额外的查询当存取这些关联对象。*select_relate*方法是给*ForeignKey*和*OneToOne*字段使用的。它通过执行一个 *SQL JOIN*并且包含关联对象的字段在*SELECT* 声明中。

为了利用`select_related()`,编辑之前代码中的以下行(译者注：请注意双下划线）：

    actions = actions.filter(user_id__in=following_ids)

添加*select_related*在你将要使用的字段上：

```python
actions = actions.filter(user_id__in=following_ids)\
                    .select_related('user', 'user__profile')
```

我们使用`user__profile`（译者注：请注意是双下划线）来连接profile表在一个单独的*SQL*查询中。如果你调用`select_related()`而不传入任何参数，它会取回所有*ForeignKey*关系的对象。给`select_related()`限制的关系将会在随后一直访问。

> 小心的使用`select_related()`将会极大的提高执行时间

### 使用*prefetch_related*

如你所见，`select_related()`将会帮助你提高取回一对多关系的关联对象的执行效率。但是，`select_related()`无法给多对多或者多对一关系（ManyToMany或者倒转*ForeignKey*字段）工作。Django提供了一个不同的查询集（QuerySets）方法叫做*prefetch_realted*，该方法在`select_related()`方法支持的关系上增加了多对多和多对一的关系。`prefetch_related()`方法为每一种关系执行单独的查找然后对各个结果进行连接通过使用Python。这个方法还支持*GeneriRelation*和*GenericForeignKey*的预先读取。

完成你的查询通过为它添加`prefetch_related()`给目标*GenericForeignKey*字段，如下所示：

```python
actions = actions.filter(user_id__in=following_ids)\
                 .select_related('user', 'user__profile')\
                 .prefetch_related('target')
```

这个查询现在已经被充分利用用来取回包含关联对象的用户动作（actions）。

### 为*actions*创建模板（templates）

我们要创建一个模板（template）用来显示一个独特的*Action*对象。在*actions*应用中创建一个新的目录命名为*templates*。添加如下文件结构：

```
actions/
    action/
        detail.html
```

编辑*actions/action/detail.html*模板（template）文件添加如下代码：

```html
明天添加
```

这个模板用来显示一个*Action*对象。首先，我们使用`{% with %}`模板标签（template tag）来获取用户操作的动作（action）和他们的profile。然后，我们显示目标对象的图片如果*Action*对象有一个关联的目标对象。最后，如果有执行过的动作（action），包括动作和目标对象，我们就显示链接给用户。

现在，编辑*account/dashboard.html*模板（template）添加如下代码到*content*区块下方：

```html
<h2>What's happening</h2>
<div id="action-list">
    {% for action in actions %}
        {% include "actions/action/detail.html" %}
    {% endfor %}
</div>
```

在浏览器中打开 http://127.0.0.1:8000/account/ 。登录一个存在的用户并且该用户执行过一些操作已经被存储在数据库中。然后，登录其他用户，关注之前登录的用户，在dashboard页面可以看到生成的动作流。如下所示：

![django-6-4](http://upload-images.jianshu.io/upload_images/3966530-b25d0a8feb617640.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们刚刚创建了一个完整的活动流（activity stream）给我们的用户并且我们还能非常容易的添加新的用户动作给它。你还可以添加无限的滚动功能给活动流（activity stream）通过集成AJAX分页处理，和我们之前在*image_list*视图（view）使用过的一样。

###给非规范化（denormalizing）计数使用信号

有一些场景，你想要使你的数据非规范化。非规划化使指在一定的程度上制造一些数据冗余用来优化读取的性能。你必须十分小心的使用非规划化并且只有在你真的非常需要它的时候才能使用。你会发现非规划化的最大问题就是保持你的非规范化数据更新是非常困难的。

我们将会看到一个例子关于如何改善（improve）我们的查询通过使用非规范化计数。缺点就是我们不得不保持冗余数据的更新。我们将要从我们的*Image*模型（model）中使数据非规范化然后使用Django信号来保持数据的更新。

###使用信号进行工作

Django自带一个信号调度程序允许*receiver*函数在某个动作出现的时候去获取通知。信号非常有用，当你需要你的代码去执行某些事件的时候同时正在发生其他事件。你还能够创建你自己的信号这样一来其他人可以在某个事件发生的时候获得通知。

Django模型（models）提供了几个信号，它们位于*django.db.models.signales*。举几个例子：

* *pre_save* 和 *post_save*：前者会在调用模型（model）的`save()`方法前发送信号，后者反之。
* *pre_delete* 和 *post_delete*：前者会在调用模型（model）或查询集（QuerySets）的`delete()`方法之前发送信号，后者反之。
* *m2m_changed*：当在一个模型（model）上的*ManayToManayField*被改变的时候发送信号。

以上只是Django提供的一小部分信号。你可以通过访问 https://docs.djangoproject.com/en/1.8/ref/signals/ 获得更多信号资料。

打个比方，你想要获取热门图片。你可以使用Django的聚合函数来获取图片，通过图片获取的用户喜欢数量来进行排序。要记住你已经使用过Django聚合函数在*第三章 扩展你的blog应用*。以下代码将会获取图片并进行排序通过它们被用户喜欢的数量：

```python
from django.db.models import Count
from images.models import Image
images_by_popularity = Image.objects.annotate(
    total_likes=Count('users_like')).order_by('-total_likes')
```

但是，通过统计图片的总喜欢数量进行排序比直接使用一个已经存储总统计数的字段进行排序要消耗更多的性能。你可以添加一个字段给*Image*模型（model）用来非规范化喜欢的数量用来提升涉及该字段的查询的性能。那么，问题来了，我们该如何保持这个字段是最新更新过的。

编辑*images*应用下的*models.py*文件，给*Image*模型（model）添加以下字段：

```python
total_likes = models.PositiveIntegerField(db_index=True,
                                          default=0)
```

*total_likes*字段允许我们给每张图片存储被用户喜欢的总数。非规范化数据非常有用当你想要使用他们来过滤或排序查询集（QuerySets）。

> 在你使用非规范化字段之前你必须考虑下其他几种提高性能的方法。考虑下数据库索引，最佳化查询以及缓存在开始规范化你的数据之前。

运行以下命令将新添加的字段迁移到数据库中：

    python manage.py makemigrations images
    
你会看到如下输出：

```python
Migrations for 'images':
    0002_image_total_likes.py:
        - Add field total_likes to image
```

接着继续运行以下命令来应用迁移：

    python manage.py migrate images
    
输出中会包含以下内容：

    Applying images.0002_image_total_likes... OK
    
我们要给*m2m_changed*信号附加一个*receiver*函数。在*images*应用目录下创建一个新的文件命名为*signals.py*。给该文件添加如下代码：

```python
from django.db.models.signals import m2m_changed
from django.dispatch import receiver
from .models import Image
@receiver(m2m_changed, sender=Image.users_like.through)
def users_like_changed(sender, instance, **kwargs):
    instance.total_likes = instance.users_like.count()
    instance.save()
```

首先，我们使用`receiver()`装饰器将`users_like_changed`函数注册成一个*receiver*函数，然后我们将该函数附加给*m2m_changed*信号。我们将这个函数与*Image.users_like.through*连接，这样这个函数只有当*m2m_changed*信号被*Image.users_like.through*执行的时候才被调用。还有一个可以替代的方式来注册一个*receiver*函数，由使用*Signal*对象的`connect()`方法组成。

> Django信号是同步阻塞的。不要使用异步任务导致信号混乱。但是，你可以联合两者来执行异步任务当你的代码只接受一个信号的通知。

你必须连接你的*receiver*函数给一个信号，只有这样它才会被调用当连接的信号发送的时候。有一个推荐的方法用来注册你的信号是在你的应用配置类中导入它们到`ready()`方法中。Django提供一个应用注册允许你对你的应用进行配置和内省。

###典型的应用配置类

django允许你指定配置类给你的应用们。为了提供一个自定义的配置给你的应用，创建一个继承*django.apps*的*Appconfig*类的自定义类。这个应用配置类允许你为应用存储元数据和配置并且提供
内省。

你可以通过访问 https://docs. djangoproject.com/en/1.8/ref/applications/ 获取更多关于应用配置的信息。

为了注册你的信号*receiver*函数，当你使用`receiver()`装饰器的时候，你只需要导入信号模块，这些信号模块被包含在你的应用的*AppConfig*类中的`ready()`方法中。这个方法在应用注册被完整填充的时候就调用。其他给你应用的初始化都可以被包含在这个方法中。

在*images*应用目录下创建一个新的文件命名为*apps.py*。为该文件添加如下代码：

```python
from django.apps import AppConfig
class ImagesConfig(AppConfig):
    name = 'images'
    verbose_name = 'Image bookmarks'
    def ready(self):
        # import signal handlers
        import images.signals
```

*name*属性定义该应用完整的Python路径。*verbose_name*属性设置了这个应用可读的名字。它会在管理站点中显示。`ready()`方法就是我们为这个应用导入信号的地方。

现在我们需要告诉Django我们的应用配置位于哪里。编辑位于*images*应用目录下的*__init__.py*文件添加如下内容：

    default_app_config = 'images.apps.ImagesConfig'
    
打开你的浏览器浏览一个图片的详细页面然后点击**like**按钮。再进入管理页面看下该图片的*total_like*属性。你会看到*total_likes*属性已经更新了最新的like数如下所示：

![django-6-5](http://upload-images.jianshu.io/upload_images/3966530-5e1e8cfc6b3cfcbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在，你可以使用*totla_likes*属性来进行热门图片的排序或者在任何地方显示这个值，从而避免了复杂的查询操作。以下获取图片的查询通过图片的喜欢数量进行排序：

```python
images_by_popularity = Image.objects.annotate(
    likes=Count('users_like')).order_by('-likes')
```

现在我们可以用新的查询来代替上面的查询：

    images_by_popularity = Image.objects.order_by('-total_likes')

以上查询的返回结果只需要很少的SQL查询性能。以上就是一个例子关于如何使用Django信号。

> 小心使用信号，因为它们会给理解控制流制造困难。在很多场景下你可以避免使用信号如果你知道哪个接收器需要被通知。

###使用Redis来存储视图（views）项

Redis是一个高级的*key-value*（键值）数据库允许你保存不同类型的数据并且在*I/O*(输入/输出）操作上非常非常的快速。Redis可以在内存中存储任何东西，但是这些数据能够持续通过偶尔存储数据集到磁盘中或者添加每一条命令到日志中。Redis是非常出彩的通过与其他的键值存储对比：它提供了一个强大的设置命令，并且支持多种数据结构，例如string，hashes，lists，sets，ordered sets，甚至bitmaps和HyperLogLogs。

SQL最适合用于模式定义的持续数据存储，而Redis提供了许多优势当需要处理快速变化的数据，易失性存储，或者需要一个快速缓存的时候。让我们看下Redis是如何被使用的，当构建新的功能到我们的项目中。

###安装Redis

从 http://redis.io/download 下载最新的Redis版本。解压*tar.gz*文件，进入*redis*目录然后编译Redis通过使用以下make命令：

```shell
cd redis-3.0.4(译者注：版本根据自己下载的修改)
make （译者注：这里是假设你使用的是linux或者mac系统才有make命令，windows如何操作请看下官方文档）
```

在Redis安装完成后允许以下shell命令来初始化Redis服务：

    src/redis-server
    
你会看到输出的结尾如下所示：

```shell
# Server started, Redis version 3.0.4
* DB loaded from disk: 0.001 seconds
* The server is now ready to accept connections on port 6379
```

默认的，Redis运行会占用6379端口，但是你也可以指定一个自定义的端口通过使用`--port`标志，例如：`redis-server --port 6655`。当你的服务启动完毕，你可以在其他的终端中打开Redis客户端通过使用如下命令：

    src/redis-cli
    
你会看到Redis客户端shell如下所示：

    127.0.0.1:6379>
    
Redis客户端允许你在当前shell中立即执行Rdis命令。来我们来尝试一些命令。键入*SET*命令在Redis客户端中存储一个值到一个键中：

```shell
127.0.0.1:6379> SET name "Peter"
ok
```

以上的命令创建了一个带有字符串“Peter”值的*name*键到Redis数据库中。*OK*输出表明该键已经被成功保存。然后，使用*GET*命令获取之前的值，如下所示：

```shell
127.0.0.1:6379> GET name
"Peter"
```

你还可以检查一个键是否存在通过使用*EXISTS*命令。如果检查的键存在会返回1，反之返回0：

```shell
127.0.0.1:6379> EXISTS name
(integer) 1
```

你可以给一个键设置到期时间通过使用*EXPIRE*命令，该命令允许你设置该键能在几秒内存在。另一个选项使用*EXPIREAT*命令来期望一个Unix时间戳。键的到期消失是非常有用的当将Redis当做缓存使用或者存储易失性的数据：

```shell
127.0.0.1:6379> GET name
"Peter"
127.0.0.1:6379> EXPIRE name 2
(integer) 1
Wait for 2 seconds and try to get the same key again:
127.0.0.1:6379> GET name
(nil)
```

（nil）响应是一个空的响应说明没有找到键。你还可以通过使用*DEL*命令删除任意键，如下所示：

```shell
127.0.0.1:6379> SET total 1
OK
127.0.0.1:6379> DEL total
(integer) 1
127.0.0.1:6379> GET total
(nil)
```

以上只是一些键选项的基本命令。Redis包含了庞大的命令设置给一些数据类型，例如strings，hashes，sets，ordered sets等等。你可以通过访问 http://redis.io/commands 看到所有Reids命令以及通过访问 http://redis.io/topics/data-types 看到所有Redis支持的数据类型。

###通过Python使用Redis

我们需要绑定Python和Redis。通过pip渠道安装*redis-py*命令如下：

    pip install redis==2.10.3（译者注：版本可能有更新，如果需要最新版本，可以不带上'==2.10.3'后缀）
    
你可以访问 http://redis-py.readthedocs.org/ 得到redis-py文档。

*redis-py*提供两个类用来与Redis交互：*StrictRedis*和*Redis*。两者提供了相同的功能。*StrictRedis*类尝试遵守官方的Redis命令语法。*Redis*类型继承*Strictredis*重写了部分方法来提供向后的兼容性。我们将会使用*StrictRedis*类，因为它遵守Redis命令语法。打开Python shell执行以下命令：

```
>>> import redis
>>> r = redis.StrictRedis(host='localhost', port=6379, db=0)
```

上面的代码创建了一个与Redis数据库的连接。在Redis中，数据库通过一个整形索引替代数据库名字来辨识。默认的，一个客户端被连接到数据库 0 。Reids数据库可用的数字设置到16，但是你可以在*redis.conf*文件中修改这个值。

现在使用Python shell设置一个键：

```
>>> r.set('foo', 'bar')
True
```

以上命令返回Ture表明这个键已经创建成功。现在你可以使用`get()`命令取回该键：

```
>>> r.get('foo')
'bar'
```

如你所见，*StrictRedis*方法遵守Redis命令语法。

让我们集成Rdies到我们的项目中。编辑*bookmarks*项目的*settings.py*文件添加如下设置：

```python
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 0
```

以上设置了Redis服务器和我们将要在项目中使用到的数据库。

###存储视图（vies）项到Redis中

让我们存储一张图片被查看的总次数。如果我们通过Django ORM来完成这个操作，它会在每次该图片显示的时候执行一次SQL UPDATE声明。使用Redis，我们只需要对一个计数器进行增量存储在内存中，从而带来更好的性能。

编辑*images*应用下的*views.py*文件，添加如下代码：

```python
import redis
from django.conf import settings
# connect to redis
r = redis.StrictRedis(host=settings.REDIS_HOST,
                      port=settings.REDIS_PORT,
                      db=settings.REDIS_DB)
```

在这儿我们建立了Redis的连接为了能在我们的视图（views)中使用它。编辑*images_detail*视图（view）使它看上去如下所示：

```python
def image_detail(request, id, slug):
image = get_object_or_404(Image, id=id, slug=slug)
# increment total image views by 1
total_views = r.incr('image:{}:views'.format(image.id)) 
return render(request,
              'images/image/detail.html',
              {'section': 'images',
               'image': image,
               'total_views': total_views})
```

在这个视图（view）中，我们使用*INCR*命令，它会从1开始增量一个键的值，在执行这个操作之前如果键不存在，它会将值设定为0.`incr()`方法在执行操作后会返回键的值，然后我们可以存储该值到*total_views*变量中。我们构建Rddis键使用一个符号，比如 *object-type:id:field (for example image:33:id)* 。

> 对Redis的键进行命名有一个惯例是使用冒号进行分割来创建键的命名空间。做到这点，键的名字会特别冗长，有关联的键会分享部分相同的模式在它们的名字中。

编辑*image/detail.html*模板（template）在已有的`<span class="count">`元素之后添加如下代码：

```html
<span class="count">
     <span class="total">{{ total_views }}</span>
     view{{ total_views|pluralize }}
</span>
```

现在在浏览器中打开一张图片的详细页面然后多次加载该页面。你会看到每次该视图（view）被执行的时候，总的观看次数会增加 1 。如下所示：

![django-6-6](http://upload-images.jianshu.io/upload_images/3966530-7faeef32eacf1b01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你已经成功的集成Redis到你的项目中来存储项统计。

###存储一个排名到Reids中

让我们使用Reids构建更多的功能。我们要在我们的平台中创建一个最多浏览次数的图片排行。为了构建这个排行我们将要使用Redis分类集合。一个分类集合是一个非重复的字符串采集，其中每个成员和一个分数关联。其中的项根据它们的分数进行排序。

编辑*images*引用下的*views.py*文件，使*image_detail*视图（view）看上去如下所示：

```python
def image_detail(request, id, slug):
image = get_object_or_404(Image, id=id, slug=slug)
# increment total image views by 1
total_views = r.incr('image:{}:views'.format(image.id)) # increment image ranking by 1 
r.zincrby('image_ranking', image.id, 1)
return render(request,
              'images/image/detail.html',
              {'section': 'images',
               'image': image,
               'total_views': total_views})
```

我们使用`zincrby()`命令存储图片视图（views）到一个分类集合中通过键`image:ranking`。我们存储图片*id*，和一个分数1，它们将会被加到分类集合中这个元素的总分上。这将允许我们在全局上持续跟踪所有的图片视图（views），并且有一个分类集合，该分类集合通过图片的浏览次数进行排序。

现在创建一个新的视图（view）用来展示最多浏览次数图片的排行。在*views.py*文件中添加如下代码：

```python
@login_required
def image_ranking(request):
    # get image ranking dictionary
    image_ranking = r.zrange('image_ranking', 0, -1,
                             desc=True)[:10]
    image_ranking_ids = [int(id) for id in image_ranking]
    # get most viewed images
    most_viewed = list(Image.objects.filter(
                       id__in=image_ranking_ids))
    most_viewed.sort(key=lambda x: image_ranking_ids.index(x.id))
    return render(request,
                  'images/image/ranking.html',
                  {'section': 'images',
                   'most_viewed': most_viewed})
```

以上就是*image_ranking*视图。我们使用`zrange()`命令获得分类集合中的元素。这个命令期望一个自定义的范围，最低分和最高分。通过将 0 定为最低分， -1 为最高分，我们告诉Redis返回分类集合中的所有元素。最终，我们使用`[:10]`对结果进行切片获取最前面十个最高分的元素。我们构建一个返回的图片IDs的列，然后我们将该列存储在*image_ranking_ids*变量中，这是一个整数列。我们通过这些IDs取回对应的*Image*对象，并将它们强制转化为列通过使用`list()`函数。强制转化查询集（QuerySets）的执行是非常重要的，因为接下来我们要在该列上使用列的`sort()`方法（就是因为这点所以我们需要的是一个对象列而不是一个查询集（QuerySets））。我们排序这些*Image*对象通过它们在图片排行中的索引。现在我们可以在我们的模板（template）中使用*most_viewed*列来显示10个最多浏览次数的图片。

创建一个新的*image/ranking.html*模板（template）文件，添加如下代码：

```html
{% extends "base.html" %}

{% block title %}Images ranking{% endblock %}

{% block content %}
    <h1>Images ranking</h1>
     <ol>
       {% for image in most_viewed %}
         <li>
           <a href="{{ image.get_absolute_url }}">
             {{ image.title }}
           </a> 
         </li>
       {% endfor %}
     </ol>
{% endblock %}
```

这个模板（template）非常简单明了，我们只是对包含在*most_viewed*中的*Image*对象进行迭代。

最后为新的视图（view）创建一个URL模式。编辑*images*应用下的*urls.py*文件，添加如下内容：

    url(r'^ranking/$', views.image_ranking, name='create'),
    
在浏览器中打开 http://127.0.0.1:8000/images/ranking/ 。你会看到如下图片排行：

![django-6-7](http://upload-images.jianshu.io/upload_images/3966530-e3e6a0ac2b862a51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###Redis的下一步

Redis并不能替代你的SQL数据库，但是它是一个内存中的快速存储，更适合某些特定任务。将它添加到你的栈中使用当你真的感觉它很需要。以下是一些适合Redis的场景：

* Counting：如你之前看到的，通过Redis管理计数器非常容易。你可以使用`incr()`和`incrby()。
* Storing latest items：你可以添加项到一个列的开头和结尾通过使用`lpush()`和`rpush()`。移除和返回开头和结尾的元素通过使用`lpop()`以及`rpop()`。你可以削减列的长度通过使用`ltrim()`来维持它的长度。
* Queues：除了push和pop命令，Redis还提供堵塞的队列命令。
* Caching：使用`expire()`和`expireat()`允许你将Redis当成缓存使用。你还可以找到第三方的Reids缓存后台给Django使用。
* Pub/Sub：Redis提供命令给订阅或不订阅，并且给渠道发送消息。
* Rankings and leaderboards：Redis使用分数的分类集合使创建排行榜非常的简单。
* Real-time tracking：Redis快速的I/O(输入/输出)使它能完美支持实时场景。

###总结

在本章中，你构建了一个粉丝系统和一个用户活动流（activity stream）。你学习了Django信号是如何进行工作并且在你的项目中集成了Redis。

在下一章中，你会学习到如何构建一个在线商店。你会创建一个产品目录并且通过会话（sessions）创建一个购物车。你还会学习如何通过Celery执行异步任务。

###译者总结：

这一章好长啊！最后部分的Redis感觉最实用。准备全书翻译好后再抽时间把翻译好的所有章节全部重新校对下！那么大家下章再见！祈祷我年终中大奖！



