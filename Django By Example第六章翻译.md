# 第六章

中介模型（intermediate model）
查询集（QuerySets）

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

编辑你的项目的*setting.py*文件，添加如下代码：

```python
ABSOLUTE_URL_OVERRIDES = {    'auth.user': lambda u: reverse_lazy('user_detail',                                        args=[u.username])}
```








