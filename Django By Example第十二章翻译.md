书籍出处：https://www.packtpub.com/web-development/django-example
原作者：Antonio Melé

**（译者注：第十二章，全书最后一章，终于到这章了。）**

#第十二章
##构建一个API

在上一章中，你构建了一个学生注册系统和课程报名。你创建了用来展示课程内容的视图以及如何使用Django的缓存框架。在这章中，你将学习如何做到以下几点：

* 构建一个 RESTful API
* 用API视图操作认证和权限
* 创建API视图放置和路由

##构建一个RESTful API

你可能想要创建一个接口给其他的服务来与你的web应用交互。通过构建一个API，你可以允许第三方来消费信息以及程序化的操作你的应用程序。

你可以通过很多方法构成你的API，但是我们最鼓励你遵循REST原则。REST体系结构来自Representational State Transfer。RESTful API是基于资源的（resource-based）。你的模型代表资源和HTTP方法例如GET,POST,PUT,以及DELETE是被用来取回，创建，更新，以及删除对象的。HTTP响应代码也可以在上下文中使用。不同的HTTP响应代码的返回用来指示HTTP请求的结果，例如，2XX响应代码用来表示成功，4XX表示错误，等等。

在RESTful API中最通用的交换数据是JSON和XML。我们将要为我们的项目构建一个JSON序列化的REST API。我们的API会提供以下功能：

* 获取科目
* 获取可用的课程
* 获取课程内容
* 课程报名

我们可以通过创建定制视图从Django开始构建一个API。当然，有很多第三方的模块可以给你的项目简单的创建一个API，其中最有名的就是Django Rest Framework。

##安装Django Rest Framework

Django Rest Framework允许你为你的项目方便的构建REST APIs。你可以通过访问 http://www.django-rest-framework.org 找到所有REST Framework信息。

打开shell然后通过以下命令安装这个框架：

    pip install djangorestframework=3.2.3
    
编辑*educa*项目的*settings.py*文件，在*INSTALLED_APPS*设置中添加*rest_framework*来激活这个应用，如下所示：

```python
INSTALLED_APPS = (       # ...       'rest_framework',   )
```

之后，添加如下代码到*settings.py*文件中：

```python
REST_FRAMEWORK = {       'DEFAULT_PERMISSION_CLASSES': [
'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'       ]}
```

你可以使用*REST_FRAMEWORK*设置为你的API提供一个指定的配置。REST Framework提供了一个广泛的设置去配置默认的行为。*DEFAULT_PERMISSION_CLASSES*配置指定了去读取，创建，更新或者删除对象的默认权限。我们设置*DjangoModelPermissionsOrAnonReadOnly*作为唯一的默认权限类。这个类依赖与Django的权限系统允许用户去创建，更新，或者删除对象，同时提供只读的访问给陌生人用户。你会在之后学习更多关于权限的方面。

如果要找到一个完整的REST框架可用设置列表，你可以访问 http://www.django-rest-framework.org/api-guide/settings/ 。

##定义序列化器

设置好REST Framework之后，我们需要指定我们的数据将会如何序列化。输出的数据必须被序列化成指定的格式，并且输出的数据将会给进程去序列化。REST框架提供了以下类来给单个对象去构建序列化：

* Serializer：给一般的Python类实例提供序列化。
* ModelSerializer：给模型实例提供序列化。
* HyperlinkedModelSerializer：类似与*ModelSerializer*，但是代表与链接而不是主键的对象关系。

让我们构建我们的第一个序列化器。在*courses*应用目录下创建以下文件结构：

```shell
api/    __init__.py    serializers.py
```

我们将会在*api*目录中构建所有的API功能为了保持一切都有良好的组织。编辑*serializeers.py*文件，然后添加以下代码：

```python
from rest_framework import serializersfrom ..models import Subject
class SubjectSerializer(serializers.ModelSerializer):    class Meta:        model = Subject        fields = ('id', 'title', 'slug')
```

以上是给*Subject*模型使用的序列化器。序列化器以一种类似的方式被定义给Django
的*From*和*ModelForm*类。*Meta*类允许你去指定模型序列化以及给序列化包含的字段。所有的模型字段都会被包含如果你没有设置一个*fields*属性。

让我们尝试我们的序列化器。打开命令行通过`python manage.py shell*开始Django shell。运行以下代码：

```python
from courses.models import Subjectfrom courses.api.serializers import SubjectSerializersubject = Subject.objects.latest('id')serializer = SubjectSerializer(subject)serializer.data
```

在上面的例子中，我们拿到了一个*Subject*对象，创建了一个*SubjectSerializer*的实例，并且访问序列化的数据。你会得到以下输出：

```shell
{'slug': 'music', 'id': 4, 'title': 'Music'}
```

如你所见，模型数据被转换成了Python的数据类型。

##了解解析器和渲染器

在你在一个HTTP响应中返回序列化数据之前，这个序列化数据必须使用指定的格式进行渲染。同样的，当你拿到一个HTTP请求，在你使用这个数据操作之前你必须解析传入的数据并且反序列化这个数据。REST Framework包含渲染器和解析器来执行以上操作。

让我们看下如何解析传入的数据。给予一个JSON字符串输入，你可以使用REST康佳提供的*JSONParser*类来转变它成为一个Python对象。在Python shell中执行以下代码：

```python
from io import BytesIOfrom rest_framework.parsers import JSONParserdata = b'{"id":4,"title":"Music","slug":"music"}'JSONParser().parse(BytesIO(data))
```

你将会拿到以下输出：

    {'id': 4, 'title': 'Music', 'slug': 'music'}
    
REST Framework还包含*Renderer*类，该类允许你去格式化API响应。框架会查明通过的内容使用的是哪种渲染器。它对响应进行检查，根据请求的*Accept*头去预判内容的类型。除此以外，渲染器可以通过URL的格式后缀进行预判。举个例子，访问将会出发*JSONRenderer*为了返回一个JSON响应。

回到shell中，然后执行以下代码去从提供的序列化器例子中渲染*serializer*对象：

```python
from rest_framework.renderers import JSONRendererJSONRenderer().render(serializer.data)
```

你会看到以下输出：

   b'{"id":4,"title":"Music","slug":"music"}'
   
我们使用*JSONRenderer*去渲染序列化数据为JSON。默认的，REST Framework使用两种不同的渲染器：*JSONRenderer*和*BrowsableAPIRenderer*。后者提供一个web接口可以方便的浏览你的API。你可以通过*REST_FRAMEWORK*设置的*DEFAULT_RENDERER_CLASSES*选项改变默认的渲染器类。

 你可以找到更多关于渲染器和解析器的信息通过访问 http://www.django-rest-framework.org/api-guide/renderers/ 以及 http://www.django-rest- framework.org/api-guide/parsers/ 。
 
##构建列表和详情视图
 
 REST Framework自带一组通用视图和mixins，你可以用来构建你自己的API。它们提供了获取，创建，更新以及删除模型对象的功能。你可以看到所有REST Framework提供的通用mixins和视图，通过访问 http://www.django-rest-framework.org/api-guide/generic-views/ 。
 
让我们创建列表和详情视图去取回*Subject*对象们。在*courses/api/*目录下创建一个新的文件并命名为*views.py*。添加如下代码：

```python
from rest_framework import genericsfrom ..models import Subjectfrom .serializers import SubjectSerializerclass SubjectListView(generics.ListAPIView):    queryset = Subject.objects.all()    serializer_class = SubjectSerializerclass SubjectDetailView(generics.RetrieveAPIView):    queryset = Subject.objects.all()    serializer_class = SubjectSerializer
```

在这串代码中，我们使用REST Framework提供的*ListAPIView*和*RetrieveAPIView*视图。我们给给予的关键值包含了一个*pk* URL参数给详情视图去取回对象。两个视图都有以下属性：

* queryset：基础查询集用来取回对象。
* serializer_class：这个类用来序列化对象。

让我们给我们的视图添加URL模式。在*courses/api/*目录下创建新的文件并命名为*urls.py*并使之看上去如下所示：

```python
from django.conf.urls import urlfrom . import viewsurlpatterns = [       url(r'^subjects/$',           views.SubjectListView.as_view(),           name='subject_list'),       url(r'^subjects/(?P<pk>\d+)/$',           views.SubjectDetailView.as_view(),           name='subject_detail'),]
```

编辑*educa*项目的主*urls.py*文件并且包含以下API模式：

```python
urlpatterns = [    # ...    url(r'^api/', include('courses.api.urls', namespace='api')),]
```

我们给我们的API URLs使用*api*命名空间。确保你的服务器已经通过命令`python manange.py runserver`启动。打开shell然后通过*cURL*获取URL http://127.0.0.1:8000/api/subjects/ 如下所示：

    $ curl http://127.0.0.1:8000/api/subjects/
    
你会获取类似以下的响应：

```shell
[{"id":2,"title":"Mathematics","slug":"mathematics"},{"id":4,"title":"Music","slug":"music"},{"id":3,"title":"Physics","slug":"physics"},{"id":1,"title":"Programming","slug":"programming"}]
```

这个HTTP响应包含JSON格式的一个*Subject*对象列。如果你的操作系统没有安装过*cURL*，你可还可以使用其他的工具去发送定制HTTP请求例如一个浏览器扩展 Postman ，这个扩展你可以在 https://www.getpostman.com 找到。

在你的浏览器中打开 http://127.0.0.1:8000/api/subjects/ 。你会看到如下所示的REST Framework的可浏览API：

![django-12-1](http://ohqrvqrlb.bkt.clouddn.com/django-12-1.png)

这个HTML界面由*BrowsableAPIRenderer*渲染器提供。它展示了结果头和内容并且允许执行请求。你还可以在URL包含一个*Subject*对象的id来访问该对象的API详情视图。在你的浏览器中打开 http://127.0.0.1:8000/api/subjects/1/ 。你将会看到一个单独的渲染成JSON格式的*Subject*对象。

##创建嵌套的序列化

我们将要给*Course*模型创建一个序列化。编辑*api/serializers.py*文件并添加以下代码：

```python
from ..models import Courseclass CourseSerializer(serializers.ModelSerializer):    class Meta:
        model = Course
        fields = ('id', 'subject', 'title', 'slug', 'voerview',
                  'created', 'owner', 'modules')
```

让我们看下一个*Course*对象是如何被序列化的。打开shell，运行`python manage.py shell`，然后运行以下代码：

```python
from rest_framework.renderers import JSONRendererfrom courses.models import Coursefrom courses.api.serializers import CourseSerializercourse = Course.objects.latest('id')serializer = CourseSerializer(course)JSONRenderer().render(serializer.data)
```

你将会通过我们包含在*CourseSerializer*中的字段获取到一个JSON对象。你可以看到*modules*管理器的被关联对象呗序列化成一列关键值，如下所示：

    "modules": [17, 18, 19, 20, 21, 22]我们想要包含个多的信息关于每一个模块，所以我们需要序列化*Module*对象以及嵌套它们。修改*api/serializers.py*文件提供的代码，使之看上去如下所示：

```python
from rest_framework import serializersfrom ..models import Course, Module   
class ModuleSerializer(serializers.ModelSerializer):    class Meta:        model = Module        fields = ('order', 'title', 'description')class CourseSerializer(serializers.ModelSerializer):    modules = ModuleSerializer(many=True, read_only=True)    class Meta:        model = Course        fields = ('id', 'subject', 'title', 'slug', 'overview',                    'created', 'owner', 'modules')
```

我们给*Module*模型定义了一个*ModuleSerializer*去提供序列化。之后我们添加一个*modules*属性给*CourseSerializer*去嵌套*ModuleSerializer*序列化器。我们设置*many=True*去表明我们正在序列化多个对象。*read_only*参数表明这个字段是只读的并且不可以被包含在任何输入中去创建或者升级对象。

打开shell并且再次创建一个*CourseSerializer*的实例。使用*JSONRenderer*渲染序列化器的*data*属性。这一次，被排列的模块会被通过嵌套的*ModuleSerializer*序列化器给序列化，如下所示：

```shell
"modules": [        {           "order": 0,           "title": "Django overview",           "description": "A brief overview about the Web Framework."
        }, 
        {
            "order": 1,            "title": "Installing Django",            "description": "How to install Django."        },        ... 
]
```

你可以找到更多关于序列化器的内容，通过访问 http://www.django-rest-framework.org/api-guide/serializers/。

##构建定制视图

REST Framework提供一个*APIView*类，这个类基于Django的*View*类构建API功能。*APIView*类与*View*在使用REST Framework的定制*Request*以及*Response*对象时不同，并且操作*APIException*例外的返回合适的HTTP响应。它还有一个内建的验证和认证系统去管理视图的访问。

我们将要创建一个视图给用户去对课程进行报名。编辑*api/views.py*文件并且添加以下代码：

```python
from django.shortcuts import get_object_or_404from rest_framework.views import APIViewfrom rest_framework.response import Responsefrom ..models import Courseclass CourseEnrollView(APIView):    def post(self, request, pk, format=None):        course = get_object_or_404(Course, pk=pk)        course.students.add(request.user)        return Response({'enrolled': True})
```

*CourseEnrollView*视图操纵用户对课程进行报名。以上代码解释如下：

* 我们创建了一个定制视图，是*APIView*的子类。
* 我们给POST操作定义了一个*post()*方法。其他的HTTP方法都不允许放这个这个视图。
* 我们预计一个*pk*URL参数会博涵一个课程的ID。我们通过给予的*pk*参数获取这个课程，并且如果这个不存在的话就抛出一个404异常。
* 我们添加当前用户给*Course*对象的*students*多对多关系并放回一个成功响应。

编辑*api/urls.py*文件并且给*CourseEnrollView*视图添加以下URL模式：

```python
url(r'^courses/(?P<pk>\d+)/enroll/$',       views.CourseEnrollView.as_view(),       name='course_enroll'),
```

理论上，我们现在可以执行一个POST请求去给当前用户对一个课程进行报名。但是，我们需要辨认这个用户并且阻止为认证的用户来访问这个视图。让我们看下API认证和权限是如何工作的。

##操纵认证

REST Framework提供认证类去辨别用户执行的请求。如果认证成功，这个框架会在*request.user*中设置认证的*User*对象。如果没有用户被认证，一个Django的*AnonymousUser*实例会被代替。

REST Framework提供以下认证后台：

* BasicAuthentication：HTTP基础认证。用户和密码会被编译为Base64并被客户端设置在*Authorization* HTTP头中。你可以学习更多关于它的内容，通过访问 https://en.wikipedia.org/wiki/Basic_access_authentication 。
* TokenAuthentication：基于token的认证。一个*Token*模型被用来存储用户tokens。用来认证的*Authorization* HTTP头里面拥有包含token的用户。
* SessionAuthentication：使用Djnago的会话后台（session backend）来认证。这个后台从你的网站前端来执行认证AJAX请求给API是非常有用的。

你可以创建一个通过继承REST Framework提供的*BaseAuthentication*类的子类以及重写*authenticate()*方法来构建一个定制的认证后台。

你可以在每个视图的基础上设置认证，或者通过*DEFAULT_AUTHENTICATION_CLASSES*设置为全局认证。

>认证只能失败用户正在执行的请求。它无法允许或者组织视图的访问。你必须使用权限去限制视图的访问。

你可以找到关于认证的所有信息，通过访问 http://www.django-rest- framework.org/api-guide/authentication/ 。

让我们给我们的视图添加*BasicAuthentication*。编辑*courses*应用的*api/views.py*文件，然后给*CourseEnrollView*添加一个*authentication_classes*属性，如下所示：

```python
from rest_framework.authentication import BasicAuthenticationclass CourseEnrollView(APIView):    authentication_classes = (BasicAuthentication,)    # ...
```

用户将会被设置在HTTP请求中的*Authorization*头里面的证书进行识别。

##给视图添加权限

REST Framework包含一个权限系统去限制视图的访问。一些REST Framework的内置权限如下所示：

* AllowAny：无限制的访问，无论当前用户是否通过认证。
* IsAuthenticated：只允许通过认证的用户。
* IsAuthenticatedOrReadOnly：通过认证的用户拥有完整的权限。陌生用户只允许去还行可读的方法，例如GET, HEAD或者OPETIONS。
* DjangoModelPermissions：权限与*django.contrib.auth*进行了捆绑。视图需要一个*queryset*属性。只有分配了模型权限的并经过认证的用户才能获得权限。
* DjangoObjectPermissions：基于每个对象基础上的Django权限。

如果用户没有权限，他们通常会获得以下某个HTTP错误：

* HTTP 401：无认证。
* HTTP 403：没有权限。

你可以获得更多的关于权限的信息，通过访问 http://www.django-rest- framework.org/api-guide/permissions/ 。

编辑*courses*应用的*api/views.py*文件然后给*CourseEnrollView*添加一个*permission_classes*属性，如下所示：

```python
from rest_framework.authentication import BasicAuthenticationfrom rest_framework.permissions import IsAuthenticatedclass CourseEnrollView(APIView):    authentication_classes = (BasicAuthentication,)    permission_classes = (IsAuthenticated,)    # ...
```

我们包含了*IsAuthenticated*权限。这个权限将会组织陌生用户访问这个视图。现在，我们可以之sing一个POST请求给我们的新的API方法。

确保开发服务器正在运行。打开shell然后运行以下命令：

    curl -i –X POST http://127.0.0.1:8000/api/courses/1/enroll/
    
你将会得到以下响应：

```shell
HTTP/1.0 401 UNAUTHORIZED...{"detail": "Authentication credentials were not provided."}
```

如我们所预料的，我们得到了一个401 HTTP code，因为我们没有认证过。让我们带上我们的一个用户进行下基础认证。运行以下命令：

```shell
curl -i -X POST -u student:password http://127.0.0.1:8000/api/courses/1/enroll/
```

使用一个已经存在的用户的证书替换*student:password*。你会得到以下响应：

```shell
HTTP/1.0 200 OK...{"enrolled": true}
```

你可以额访问管理站点然后检查到上面命令中的用户已经完成了课程的报名。

###创建视图设置和路由

*ViewSets*允许你去定义你的API的交互并且让REST Framework通过一个*Router*对象动态的构建URLs。通过使用视图设置，你可以避免给多个视图重复编写相同的逻辑。视图设置包含典型的创建，获取，更新，删除选项操作，它们是 *list()*,*create()*,*retrieve()*,*update()*,*partial_update()*以及*destroy()*。

让我们给*Course*模型创建一个视图设置。编辑*api/views.py*文件然后添加以下代码：

```python
from rest_framework import viewsetsfrom .serializers import CourseSerializerclass CourseViewSet(viewsets.ReadOnlyModelViewSet):    queryset = Course.objects.all()    serializer_class = CourseSerializer
```

我们创建了一个继承*ReadOnlyModelViewSet*类的子类，被继承的类提供了只读的操作 *list()*和*retrieve()*，前者用来排列对象，后者用来取回一个单独的对象。编辑*api/urls.py*文件并且给我们的视图设置创建一个路由，如下所示：

```python
from django.conf.urls import url, includefrom rest_framework import routersfrom . import viewsrouter = routers.DefaultRouter()router.register('courses', views.CourseViewSet)urlpatterns = [    # ...    url(r'^', include(router.urls)),]
```

我们创建两个一个*DefaultRouter*对象并且通过*courses*前缀注册了我们的视图设置。这个路由负责给我们的视图动态的生成URLs。

在你的浏览器中打开 http://127.0.0.1:8000/api/ 。你会看到路由排列除了所有的视图设置在它的基础URL中，如下图所示：

![django-12-2](http://ohqrvqrlb.bkt.clouddn.com/django-12-2.png)

你可以访问 http://127.0.0.1:8000/api/courses/ 去获取课程的列表。

你可以学习到跟多关于视图设置的内容，通过访问 http://www.django-rest-framework.org/api-guide/viewsets/  。你也可以找到更多关于路由的信息，通过访问 http://www.django-rest-framework.org/api-guide/routers/ 。

###给视图设置添加额外的操作

你可以给视图设置添加额外的操作。让我们修改我们之前的*CourseEnrollView*视图成为一个定制的视图设置操作。编辑*api/views.py*文件然后修改*CourseViewSet*类如下所示：

```python
from rest_framework.decorators import detail_routeclass CourseViewSet(viewsets.ReadOnlyModelViewSet):    queryset = Course.objects.all()    serializer_class = CourseSerializer    @detail_route(methods=['post'],                authentication_classes=[BasicAuthentication],                permission_classes=[IsAuthenticated])    def enroll(self, request, *args, **kwargs):        course = self.get_object()        course.students.add(request.user)        return Response({'enrolled': True})
```

我们添加了一个定制*enroll()*方法相当于给这个视图设置的一个额外的操作。以上的代码解释如下：

* 我们使用框架的*detail_route*装饰器去指定这个类是一个在一个单独对象上被执行的操作。
* 这个装饰器允许我们给这个操作添加定制属性。我们指定这个视图只允许POST方法，并且设置了认证和权限类。
* 我们使用*self.get_object()*去获取*Course*对象。
* 我们给*students*添加当前用户的多对多关系并且返回一个定制的成功响应。

编辑*api/urls.py*文件并移除以下URL，因为我们不再需要它们：

```python
url(r'^courses/(?P<pk>[\d]+)/enroll/$',       views.CourseEnrollView.as_view(),       name='course_enroll'),
```

之后编辑*api/views.py*文件并且移除*CourseEnrollView*类。

这个用来在课程中报名的URL现在已经是由路由动态的生成。这个URL保持不变，因为它使用我们的操作名*enroll*动态的进行构建。

###创建定制权限

我们想要学生可以访问他们报名过的课程的内容。只有在这个课程中报名过的学生才能访问这个课程的内容。最好的方法就是通过一个定制的权限类。Django提供了一个*BasePermission*类允许你去定制以下功能：

* has_permission()：视图级的权限检查。
* has_object_permission()：实例级的权限检查。

以上方法会返回*True*来允许访问，相反就会返回*False*。在*courses/api/*中创建一个新的文件并命名为*permissions.py*。添加以下代码：

```python
from rest_framework.permissions import BasePermissionclass IsEnrolled(BasePermission):    def has_object_permission(self, request, view, obj):        return obj.students.filter(id=request.user.id).exists()
```

我们创建了一个继承*BasePermission*类的子类，并且重写了*has_object_permission()*。我们检查执行请求的用户是否存在*Course*对象的*students*关系。我们下一步将要使用*IsEnrolled*权限。

###序列化课程内容

我们需要序列化课程内容。*Content*模型包含一个通用的外键允许我们去关联不同的内容模型对象。然而，我们给上一章中给所有的内容模型添加了一个公用的*render()*方法。我们可以使用这个方法去提供渲染过的内容给我们的API。

编辑*courses*应用的*api/serializers.py*文件并且添加以下代码：

```python
from ..models import Content
class ItemRelatedField(serializers.RelatedField):    def to_representation(self, value):        return value.render()
        class ContentSerializer(serializers.ModelSerializer):    item = ItemRelatedField(read_only=True)    class Meta:        model = Content        fields = ('order', 'item')
```

在以上代码中，我们通过子类化REST Framework提供的*RealtedField*序列化器字段定义了一个定制字段并且重写了*to_representation()*方法。我们给*Content*模型定义了*ContentSerializer*序列化器并且使用定制字段给*item*生成外键。

我们需要一个替代序列化器给*Module*模型来包含它的内容，以及一个扩展的*Course*序列化器。编辑*api/serializers.py*文件并且添加以下代码：

```python
   class ModuleWithContentsSerializer(serializers.ModelSerializer):       contents = ContentSerializer(many=True)       class Meta:           model = Module           fields = ('order', 'title', 'description', 'contents')
           
   class CourseWithContentsSerializer(serializers.ModelSerializer):       modules = ModuleWithContentsSerializer(many=True)       class Meta:           model = Course           fields = ('id', 'subject', 'title', 'slug',                     'overview', 'created', 'owner', 'modules')           
```

让我们创建一个视图来模仿*retrieve()*操作的行为但是包含课程内容。编辑*api/views.py*文件添加以下方法给*CourseViewSet*类：

```python
   from .permissions import IsEnrolled   from .serializers import CourseWithContentsSerializer
      class CourseViewSet(viewsets.ReadOnlyModelViewSet):       # ...       @detail_route(methods=['get'],                     serializer_class=CourseWithContentsSerializer,                     authentication_classes=[BasicAuthentication],                     permission_classes=[IsAuthenticated,                                         IsEnrolled])       def contents(self, request, *args, **kwargs):           return self.retrieve(request, *args, **kwargs)
```

以上的方法解释如下：

* 我们使用*detail_route*装饰器去指定这个操作是在一个单独的对象上进行执行。
* 我们指定只有GET方法允许访问这个操作。
* 我们使用新的*CourseWithContentsSerializer*序列化器类来包含渲染过的课程内容。
* 我们使用*IsAuthenticated*和我们的定制*IsEnrolled*权限。只要做到了这点，我们可以确保只有在这个课程中报名的用户才能访问这个课程的内容。
* 我们使用存在的*retrieve()*操作去返回课程对象。

在你的浏览器中打开 http://127.0.0.1:8000/api/courses/1/contents/ 。如果你使用正确的证书访问这个视图，你会看到这个课程的每一个模块都包含给渲染过的课程内容的HTML，如下所示：

```html
   {   "order": 0,   "title": "Installing Django",
   "description": "",   "contents": [        {        "order": 0,        "item": "<p>Take a look at the following video for installing Django:</p>\n"
        }, 
        {
        "order": 1,        "item": "\n<iframe width=\"480\" height=\"360\" src=\"http://www.youtube.com/embed/bgV39DlmZ2U?wmode=opaque\" frameborder=\"0\" allowfullscreen></iframe>\n\n"        } 
    ]    }   
```

你已经构建了一个简单的API允许其他服务器来程序化的访问课程应用。REST Framework还允许你通过*ModelViewSet*视图设置去管理创建以及编辑对象。我们已经覆盖了Django Rest Framework的主要部分，但是你可以找到该框架更多的特性，通过查看它的文档，地址在 http://www.django-rest-framework.org/ 。

###总结

在这章中，你创建了一个RESTful API给其他的服务器去与你的web应用交互。

一个额外的章节**第十三章，Going Live**需要在线下载：https://www.packtpub.com/sites/default/files/downloads/Django_By_Example_GoingLive.pdf 。第十三章将会教你如何使用uWSGI以及NGINX去构建一个生产环境。你还将学习到如何去导入一个定制中间件以及创建定制管理命令。

你已经到达这本书的结尾。恭喜你！你已经学习到了通过Django构建一个成功的web应用所需的技能。这本书指导你通过其他的技术与Django集合去开发了几个现实生活能用到的项目。现在你已经准备好去创建你自己的Django项目了，无论是一个简单的样品还是一个一个强大的wen应用。

祝你下一次Django的冒险好运！

###译者总结

**完结撒花！！！！！！！！！！！！！！！！！！！！！！！！！！！！！**

**2016年12月7日开始翻译第一章到今天2017年3月29日终于全书翻译完成！**

**对我而言真的是一次非常大胆的尝试，但终于坚持下来了！！！！！！！！**

**要相信自己做的到！**

**请允许我多打几个感叹号表达我的心情！**

**感谢所有支持和鼓励的人！**

**精校工作也正在不断的进行中，请大家原谅一开始的渣翻！**

**其实还有很多很多想说的话，但是不知道说什么好！就这样吧！**






