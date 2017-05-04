#**第十一章**
#**缓存内容**

（译者 @ucag 注：这是倒数第二章，最后一个项目即将完成。 @夜夜月 将会接过翻译的最后一棒。这本书的翻译即将完成。这也是我翻译的最后一章，作为英语专业的学生，我对于翻译有了更多的[思考](http://www.jianshu.com/p/e8365dbf4a71)。同样，校对是 @夜夜月）

（译者 @夜夜月注：赞，倒数第二章了，接下来就是最后一章了！）

在上一章中，你使用模型继承和一般关系创建了一个灵活的课程内容模型。你也使用基于类的视图，表单集，以及 内容的 AJAX 排序，创建了一个课程管理系统。在这一章中，你将会：

* 创建展示课程信息的公共视图
* 创建一个学生注册系统
* 在*courses*中管理学生注册
* 创建多样化的课程内容
* 使用缓存框架缓存内容

我们将会从创建一个课程目录开始，好让学生能够浏览当前的课程以及注册这些课程。

##**展示课程**
对于课程目录，我们需要创建以下的功能：

* 列出所有的可用课程，可以通过可选科目过滤
* 展示一个单独的课程概览

编辑 courses 应用的 `views.py` ，添加以下代码：

```python`
from django.db.models import Count
from .models import Subject

class CourseListView(TemplateResponseMixin, View):
	model = Course
	template_name = 'courses/course/list.html'
	def get(self, request, subject=None):
		subjects = Subject.objects.annotate(
		              total_courses=Count('courses'))
		courses = Course.objects.annotate(
		              total_modules=Count('modules'))
		if subject:
			subject = get_object_or_404(Subject, slug=subject)
			courses = courses.filter(subject=subject)
	   return self.render_to_response({'subjects':subjects,
									           'subject': subject,
									           'courses': courses})
```

这是 `CourseListView` 。它继承了 `TemplateResponseMixin` 和 `View` 。在这个视图中，我们实现了下面的功能：

* 1 我们检索所有的课程，包括它们当中的每个课程总数。我们使用 ORM 的 `annotate()` 方法和 `Count()` 聚合方法来实现这一功能。
* 2 我们检索所有的可用课程，包括在每个课程中包含的模块总数。
* 3 如果给了科目的 slug URL 参数，我们就检索对应的课程对象，然后我们将会把查询限制在所给的科目之内。
* 4 我们使用 `TemplateResponseMixin` 提供的 `render_to_response()` 方法来把对象渲染到模板中，然后返回一个 HTTP 响应。

让我们创建一个详情视图来展示单一课程的概览。在 `views.py` 中添加以下代码：

```python
from django.views.generic.detail import DetailView

class CourseDetailView(DetailView):
	model = Course
	template_name = 'courses/course/detail.html'
```
	
这个视图继承了 Django 提供的通用视图 `DetailView` 。我们定义了 `model` 和 `template_name` 属性。Django 的 `DetailView` 期望一个 **主键(pk)** 或者 **slug** URL 参数来检索对应模型的一个单一对象。然后它就会渲染 `template_name` 中的模板，同样也会把上下文中的对象渲染进去。

编辑 `educa` 项目中的主 `urls.py` 文件，添加以下 URL 模式：

```python
from courses.views import CourseListView

urlpatterns = [
	# ...
	url(r'^$', CourseListView.as_view(), name='course_list'),
]
```
	
我们把 `course_list` 的 URL 模式添加进了项目的主 `urls.py` 中，因为我们想要把课程列表展示在 `http://127.0.0.1:8000/` 中，然后 `courses` 应用的所有的其他 URL 都有 `/course/` 前缀。

编辑 `courses` 应用的 `urls.py` ，添加以下 URL 模式：

```python
url(r'^subject/(?P<subject>[\w-]+)/$',
	views.CourseListView.as_view(),
	name='course_list_subject'),

url(r'^(?P<slug>[\w-]+)/$',
	views.CourseDetailView.as_view(),
	name='course_detail'),
```

我们定义了以下 URL 模式：

* `course_list_subject`：用于展示一个科目的所有课程
* `course_detail`：用于展示一个课程的概览

让我们为 `CourseListView` 和 `CourseDetailView` 创建模板。在 `courses` 应用的 `templates/courses/` 路径下创建以下路径：

* course/
* list.html
* detail.html

编辑 `courses/course/list.html` 模板，写入以下代码：

```html
{% extends "base.html" %}

{% block title %}
    {% if subject %}
        {{ subject.title }} courses
    {% else %}
        All courses
    {% endif %}
{% endblock %}

{% block content %}
<h1>
    {% if subject %}
        {{ subject.title }} courses
    {% else %}
        All courses
    {% endif %}
</h1>
<div class="contents">
    <h3>Subjects</h3>
    <ul id="modules">
        <li {% if not subject %}class="selected"{% endif %}>
            <a href="{% url "course_list" %}">All</a>
        </li>
        {% for s in subjects %}
            <li {% if subject == s %}class="selected"{% endif %}>
                <a href="{% url "course_list_subject" s.slug %}">
                    {{ s.title }}
                    <br><span>{{ s.total_courses }} courses</span>
                </a>
            </li>
        {% endfor %}
    </ul>
</div>
<div class="module">
    {% for course in courses %}
        {% with subject=course.subject %}
            <h3><a href="{% url "course_detail" course.slug %}">{{ course.title }}</a></h3>
            <p>
                <a href="{% url "course_list_subject" subject.slug %}">{{ subject }}</a>.
                {{ course.total_modules }} modules.
                Instructor: {{ course.owner.get_full_name }}
            </p>
        {% endwith %}
    {% endfor %}
</div>
{% endblock %}
```

这个模板用于展示可用课程列表。我们创建了一个 HTML 列表来展示所有的 `Subject` 对象，然后为它们每一个都创建了一个链接，这个链接链接到 `course_list_subject` 的 URL 。 如果存在当前科目，我们就把 `selected` HTML 类添加到当前科目中高亮显示该科目。我们迭代每个 `Course` 对象，展示模块的总数和教师的名字。

使用 `python manage.py runserver` 打开代发服务器，访问 http://127.0.0.1:8000/ 。你看到的应该是像下面这个样子：

![django-11-1](http://ohqrvqrlb.bkt.clouddn.com/django-11-1.png)

左边的侧边栏包含了所有的科目，包括每个科目的课程总数。你可以点击任意一个科目来筛选展示的课程。

编辑 `courses/course/detail.html` 模板，添加以下代码：

```html
{% extends "base.html" %}

{% block title %}
    {{ object.title }}
{% endblock %}

{% block content %}
    {% with subject=course.subject %}
        <h1>
            {{ object.title }}
        </h1>
        <div class="module">
            <h2>Overview</h2>
            <p>
                <a href="{% url "course_list_subject" subject.slug %}">{{ subject.title }}</a>.
                {{ course.modules.count }} modules.
                Instructor: {{ course.owner.get_full_name }}
            </p>
            {{ object.overview|linebreaks }}
        </div>
    {% endwith %}
{% endblock %}
```

在这个模板中，我们展示了每个单一课程的概览和详情。访问 http://127.0.0.1:8000/ ，点击任意一个课程。你就应该看到有下面结构的页面了：

![django-11-2](http://ohqrvqrlb.bkt.clouddn.com/django-11-2.png)

我们已经创建了一个展示课程的公共区域了。下面，我们需要让用户可以注册为学生以及注册他们的课程。

##**添加学生注册**
使用下面的命令创键一个新的应用：

```
python manage.py startapp students
```

编辑 `educa` 项目的 `settings.py` ，把 `students` 添加进 `INSTALLED_APPS` 设置中：

```python
INSTALLED_APPS = (
	# ...
	'students',
)
```

##**创建一个学生注册视图**
编辑 `students` 应用的 `views.py` ，写入下下面的代码：

```python
from django.core.urlresolvers import reverse_lazy
from django.views.generic.edit import CreateView
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import authenticate, login

class StudentRegistrationView(CreateView):
	template_name = 'students/student/registration.html'
	form_class = UserCreationForm
	success_url = reverse_lazy('student_course_list')
	
	def form_valid(self, form):
		result = super(StudentRegistrationView,
						self).form_valid(form)
		cd = form.cleaned_data
		user = authenticate(username=cd['username'],
							password=cd['password1'])
		login(self.request, user)
		return result
```

这个视图让学生可以注册进我们的网站里。我们使用了可以提供创建模型对象功能的通用视图 `CreateView` 。这个视图要求以下属性：

* `template_name`：渲染这个视图的模板路径。
* `form_class`：用于创建对象的表单，我们使用 Django 的 `UserCreationForm` 作为注册表单来创建 `User` 对象。
* `success_url`：当表单成功提交时要将用户重定向到的 URL 。我们逆序了 `student_course_list` URL，我们稍候将会将建它来展示学生已报名的课程。

当合法的表单数据被提交时 `form_valid()` 方法就会执行。它必须返回一个 HTTP 响应。我们覆写了这个方法来让用户在成功注册之后登录。

在 students 应用路径下创建一个新的文件，命名为 `urls.py` ，添加以下代码：

```python
from django.conf.urls import url
from . import views

urlpatterns = [
    url(r'^register/$',
	       views.StudentRegistrationView.as_view(),
	       name='student_registration'),
]
```

编辑 `educa` 的主 `urls.py` ，然后把 `students` 应用的 URLs 引入进去：

```python
url(r'^students/', include('students.urls')),
```

在 `students` 应用内创建如下的文件结构:

```
templates/
	students/
		student/
			registration.html
```

编辑 `students/student/registration.html` 模板，然后添加以下代码：

```html
{% extends "base.html" %}

{% block title %}
    Sign up
{% endblock %}

{% block content %}
    <h1>
        Sign up
    </h1>
    <div class="module">
        <p>Enter your details to create an account:</p>
        <form action="" method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Create my account"></p>
        </form>
    </div>
{% endblock %}
```

最后编辑 `educa` 的设置文件，添加以下代码：

```python
from django.core.urlresolvers import reverse_lazy
LOGIN_REDIRECT_URL = reverse_lazy('student_course_list')
```

这个是由 `auth` 模型用来给用户在成功的登录之后重定向的设置，如果请求中没有 next 参数的话。

打开开发服务器，访问 http://127.0.0.1:8000/students/register/ ，你可以看到像这样的注册表单：

![django-11-3](http://ohqrvqrlb.bkt.clouddn.com/django-11-3.png)


##**报名**
在用户创建一个账号之后，他们应该就可以在 `courses` 中报名了。为了保存报名表，我们需要在 `Course` 和 `User` 模型之间创建一个多对多关系。编辑 `courses` 应用的 `models.py` 然后把下面的字段添加进 `Course` 模型中：

```python
students = models.ManyToManyField(User,
						related_name='courses_joined',
						blank=True)
```

在 shell 中执行下面的命令来创建迁移：

```
python manage.py makemigrations
```

你可以看到类似下面的输出：

```
Migrations for 'courses':
	0004_course_students.py:
	   - Add field students to course
```

接下来执行下面的命令来应用迁移：

```
python manage.py migrate
```

你可以看到以下输出：

```
Operations to perform:
	Apply all migrations: courses
Running migrations:
	Rendering model states... DONE
	Applying courses.0004_course_students... OK
```

我们现在就可以把学生和他们报名的课程相关联起来了。
让我们创建学生报名课程的功能吧。

在 `students` 应用内创建一个新的文件，命名为 `forms.py` ，添加以下代码：

```python
from django import forms
from courses.models import Course

class CourseEnrollForm(forms.Form):
	course = forms.ModelChoiceField(queryset=Course.objects.all(),
									widget=forms.HiddenInput)
```

我们将会把这张表用于学生报名。`course` 字段是学生报名的课程。所以，它是一个 `ModelChoiceField` 。我们使用 `HiddenInput` 控件，因为我们不打算把这个字段展示给用户。我们将会在 `CourseDetailView` 视图中使用这个表单来展示一个报名按钮。

编辑  `students` 应用的 `views.py` ，添加以下代码：

```python
from django.views.generic.edit import FormView
from braces.views import LoginRequiredMixin
from .forms import CourseEnrollForm

class StudentEnrollCourseView(LoginRequiredMixin, FormView):
	course = None
	form_class = CourseEnrollForm
	
	def form_valid(self, form):
		self.course = form.cleaned_data['course']
		self.course.students.add(self.request.user)
		return super(StudentEnrollCourseView,
						self).form_valid(form)

	def get_success_url(self):
		return reverse_lazy('student_course_detail',
								args=[self.course.id])
```

这就是 `StudentEnrollCourseView` 。它负责学生在 `courses` 中报名。新的视图继承了 `LoginRequiredMixin` ，所以只有登录了的用户才可以访问到这个视图。我们把 `CourseEnrollForm`表单用在了 `form_class` 属性上，同时我们也定义了一个 `course` 属性来储存所给的 `Course` 对象。当表单合法时，我们把当前用户添加到课程中已报名学生中去。

如果表单提交成功，`get_success_url` 方法就会返回用户将会被重定向到的 URL 。这个方法相当于 `success_url` 属性。我们反序 `student_course_detail` URL ,我们稍候将会创建它来展示课程中的学生。

编辑 `students` 应用的 `urls.py` ，添加以下 URL 模式：

```python
url(r'^enroll-course/$',
	views.StudentEnrollCourseView.as_view(),
	name='student_enroll_course'),
```

让我们把报名按钮表添加进课程概览页。编辑 `course` 应用的 `views.py` ，然后修改 `CourseDetailView` 让它看起来像这样：

```python
from students.forms import CourseEnrollForm

class CourseDetailView(DetailView):
	model = Course
	template_name = 'courses/course/detail.html'
	
	def get_context_data(self, **kwargs):
		context = super(CourseDetailView,
						self).get_context_data(**kwargs)
		context['enroll_form'] = CourseEnrollForm(
						initial={'course':self.object})
		return context
```

我们使用 `get_context_data()` 方法来在渲染进模板中的上下文里引入报名表。我们初始化带有当前 `Course` 对象的表单的隐藏 course 字段，这样它就可以被直接提交了。

编辑 `courses/course/detail.html` 模板，然后找到下面的这一行：

```html
{{ object.overview|linebreaks }}
```

起始行应该被替换为下面的这几行：

```html
{{ object.overview|linebreaks }}
{% if request.user.is_authenticated %}
    <form action="{% url "student_enroll_course" %}" method="post">
        {{ enroll_form }}
        {% csrf_token %}
        <input type="submit" class="button" value="Enroll now">
    </form>
{% else %}
    <a href="{% url "student_registration" %}" class="button">
        Register to enroll
    </a>
{% endif %}
```

这个按钮就是用于报名的。如果用户是被认证过的，我们就展示包含了隐藏表单字段的报名按钮，这个表单指向了 `student_enroll_course` URL。如果用户没有被认证，我们将会展示一个注册链接。

确保已经打开了开发服务器，访问 http://127.0.0.1:8000/ ，然后点击一个课程。如果你登录了，你就可以在底部看到 **ENROLL NOW** 按钮，就像这样：

![django-11-4](http://ohqrvqrlb.bkt.clouddn.com/django-11-4.png)

如果你没有登录，你就会看到一个**Register to enroll** 的按钮。

##**获取课程内容**
我们需要一个视图来展示学生已经报名的课程，和一个获取当前课程内容的视图。编辑 `students` 应用的 `views.py` ，添加以下代码：

```python
from django.views.generic.list import ListView
from courses.models import Course

class StudentCourseListView(LoginRequiredMixin, ListView):
	model = Course
	template_name = 'students/course/list.html'
	
	def get_queryset(self):
		qs = super(StudentCourseListView, self).get_queryset()
		return qs.filter(students__in=[self.request.user])
```

这个是用于列出学生已经报名课程的视图。它继承 `LoginRequiredMixin` 来确保只有登录的用户才可以连接到这个视图。同时它也继承了通用视图 `ListView` 来展示 `Course` 对象列表。我们覆写了 `get_queryset()` 方法来检索用户已经报名的课程。我们通过学生的 `ManyToManyField` 字段来筛选查询集以达到这个目的。

把下面的代码添加进 `views.py` 文件中：

```python
from django.views.generic.detail import DetailView

class StudentCourseDetailView(DetailView):
	model = Course
	template_name = 'students/course/detail.html'
	
	def get_queryset(self):
		qs = super(StudentCourseDetailView, self).get_queryset()
		return qs.filter(students__in=[self.request.user])
	
	def get_context_data(self, **kwargs):
		context = super(StudentCourseDetailView,
						self).get_context_data(**kwargs)
		# get course object
		course = self.get_object()
		if 'module_id' in self.kwargs:
			# get current module
			context['module'] = course.modules.get(
							id=self.kwargs['module_id'])
		else:
			# get first module
			context['module'] = course.modules.all()[0]
		return context
```

这是 `StudentCourseDetailView` 。我们覆写了 `get_queryset` 方法把查询集限制在用户报名的课程之内。我们同样也覆写了 `get_context_data()` 方法来把课程的一个模块赋值在上下文内，如果给了 `model_id` URL 参数的话。否则，我们就赋值课程的第一个模块。这样，学生就可以在课程之内浏览各个模块了。

编辑 `students` 应用的 `urls.py` ，添加以下 URL 模式：

```python
url(r'^courses/$',
	views.StudentCourseListView.as_view(),
	name='student_course_list'),

url(r'^course/(?P<pk>\d+)/$',
	views.StudentCourseDetailView.as_view(),
	name='student_course_detail'),

url(r'^course/(?P<pk>\d+)/(?P<module_id>\d+)/$',
	views.StudentCourseDetailView.as_view(),
	name='student_course_detail_module'),
```

在 `students` 应用的 `templates/students/` 路径下创建以下文件结构：

```
course/
	detail.html
	list.html
```

编辑 `students/course/list.html` 模板，然后添加下列代码：

```html
{% extends "base.html" %}

{% block title %}My courses{% endblock %}

{% block content %}
    <h1>My courses</h1>

    <div class="module">
        {% for course in object_list %}
            <div class="course-info">
                <h3>{{ course.title }}</h3>
                <p><a href="{% url "student_course_detail" course.id %}">Access contents</a></p>
            </div>
        {% empty %}
            <p>
                You are not enrolled in any courses yet.
                <a href="{% url "course_list" %}">Browse courses</a>to enroll in a course.
            </p>
        {% endfor %}
    </div>
{% endblock %}
```

这个模板展示了用户报名的课程。编辑 `students/course/detail.html` 模板，添加以下代码：

```html
{% extends "base.html" %}

{% block title %}
    {{ object.title }}
{% endblock %}

{% block content %}
    <h1>
        {{ module.title }}
    </h1>
    <div class="contents">
        <h3>Modules</h3>
        <ul id="modules">
        {% for m in object.modules.all %}
            <li data-id="{{ m.id }}" {% if m == module %}
class="selected"{% endif %}>
                <a href="{% url "student_course_detail_module" object.id m.id %}">
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
    </div>
    <div class="module">
        {% for content in module.contents.all %}
            {% with item=content.item %}
                <h2>{{ item.title }}</h2>
                {{ item.render }}
            {% endwith %}
        {% endfor %}
    </div>
{% endblock %}
```

这个模板用于报名了的学生连接到课程内容。首先我们创建了一个包含所有课程模块的 HTML 列表且高亮当前模块。然后我们迭代当前的模块内容，之后使用 `{{ item.render }}` 来连接展示内容。接下来我们将会在内容模型中添加 `render()` 方法。这个方法将会负责精准的展示内容。

##**渲染不同类型的内容**
我们需要提供一个方法来渲染不同类型的内容。编辑 `course` 应用的 `models.py` ，把 `render()` 方法添加进 `ItemBase` 模型中：

```python
from django.template.loader import render_to_string
from django.utils.safestring import mark_safe

class ItemBase(models.Model):
	# ...
	def render(self):
		return render_to_string('courses/content/{}.html'.format(
							self._meta.model_name), {'item': self})
```

这个方法使用了 `render_to_string()` 方法来渲染模板以及返回一个作为字符串的渲染内容。每种内容都使用以内容模型命名的模板渲染。我们使用 `self._meta.model_name` 来为 `la` 创建合适的模板名。 `render()` 方法提供了一个渲染不同页面的通用接口。

在 `courses` 应用的 `templates/courses/` 路径下创建如下文件结构：

```
content/
	text.html
	file.html
	image.html
	video.html
```

编辑 `courses/content/text.html` 模板，写入以下代码：

```html
{{ item.content|linebreaks|safe }}
```

编辑 `courses/content/file.html` 模板，写入以下代码：

```html
<p><a href="{{ item.file.url }}" class="button">Download file</a></p>
```

编辑 `courses/content/image.html` 模板，写入以下代码：

```html
<p><img src="{{ item.file.url }}"></p>
```

为了使上传带有 `ImageField` 和 `FielField` 的文件工作，我们需要配置我们的项目以使用开发服务器提供媒体文件服务。编辑你的项目中的 `settings.py` ，添加以下代码：

```python
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media/')
```

记住 `MEDIA_URL` 是服务上传文件的基本 URL 路径， `MEDIA_ROOT` 是放置文件的本地路径。

编辑你的项目的主 `urls.py` ，添加以下 imports：

```python
from django.conf import settings
from django.conf.urls.static import static
```

然后，把下面这几行写入文件的结尾：

```python
urlpatterns += static(settings.MEDIA_URL,
				document_root=settings.MEDIA_ROOT)
```

你的项目现在已经准备好使用开发服务器上传和服务文件了。记住开发服务器不能用于生产环境中。我们将会在下一章中学习如何配置生产环境。

我们也需要创建一个模板来渲染 `Video` 对象。我们将会使用 django-embed-video 来嵌入视频内容。 Django-embed-video 是一个第三方 Django 应用，它使你可以通过提供一个视频的公共 URL 来在模板中嵌入视频，类似来自 YouTube 或者 Vimeo 的资源。

使用下面的命令来安装这个包：

```
pip isntall django-embed-video==1.0.0
```

然后编辑项目的 `settings.py` 然后添加 `embed_video` 到 `INSTALLED_APPS`设置 中。你可以在这里找到 django-embed-video 的文档：
http://django-embed-video.readthedocs.org/en/v1.0.0/ 。

编辑 `courses/content/video.html` 模板，写入以下代码：

```html
{% load embed_video_tags %}
{% video item.url 'small' %}
```

现在运行开发服务器，访问 http://127.0.0.1:8000/course/mine/ 。用属于教师组或者超级管理员的用户访问站点，然后添加一些内容到一个课程中。为了引入视频内容，你也可以复制任何一个 YouTube 视频 URL ，比如 ：https://www.youtube.com/watch?n=bgV39DlmZ2U ,然后把它引入到表单的 `url` 字段中。在添加内容到课程中之后，访问 http://127.0.0.1:8000/ ，点击课程然后点击**ENROLL NOW**按钮。你就可以在课程中报名了，然后被重定向到 `student_course_detail` URL 。下面这张图片展示了一个课程内容样本：

![django-11-5](http://ohqrvqrlb.bkt.clouddn.com/django-11-5.png)

真棒！你已经创建了一个渲染课程的通用接口了，它们中的每一个都会被用特定的方式渲染。

##**使用缓存框架**
你的应用的 HTTP 请求通常是数据库链接，数据处理，和模板渲染的。就处理数据而言，它的开销可比服务一个静态网站大多了。

请求开销在你的网站有越来越多的流量时是有意义的。这也使得缓存变得很有必要。通过缓存 HTTP 请求中 的查询结果，计算结果，或者是渲染上下文，你将会避免在接下来的请求中巨大的开销。这使得服务端的响应时间和处理时间变短。

Django 配备有一个健硕的缓存系统，这使得你可以使用不同级别的颗粒度来缓存数据。你可以缓存单一的查询，一个特定的输出视图，部分渲染的模板上下文，或者整个网站。缓存系统中的内容会在默认时间内被储存。你可以指定缓存数据过期的时间。

这是当你的站点收到一个 HTTP 请求时将会通常使用的缓存框架的方法：

	1. 试着在缓存中寻找缓存数据
	2. 如果找到了，就返回缓存数据
	3. 如果没有找到，就执行下面的步骤：
		1. 执行查询或者处理请求来获得数据
		2. 在缓存中保存生成的数据
		3. 返回数据

你可以在这里找到更多关于 Django 缓存系统的细节信息：https://docs.djangoproject.com/en/1.8/topics/cache/ 。

##**激活缓存后端**
Django 配备有几个缓存后端，他们是：

* `backends.memcached.MemcachedCache` 或  `backends.memcached.PyLibMCCache`：一个内存缓存后端。内存缓存是一个快速、高效的基于内存的缓存服务器。后端的使用取决于你选择的 Python 绑定（bindings）。
* `backends.db.DatabaseCache`： 使用数据库作为缓存系统。
* `backends.filebased.FileBasedCache`：使用文件储存系统。把每个缓存值序列化和储存为单一的文件。
* `backends.locmem.LocMemCache`：本地内存缓存后端。这是默认的缓存后端
* `backends.dummy.DummyCache`：一个用于开发的虚拟缓存后端。它实现了缓存交互界面而不用真正的缓存任何东西。缓存是独立进程且是线程安全的

>对于可选的实现，使用内存的缓存后端吧，比如 `Memcached` 后端。

##**安装 Memcached**
我们将会使用 Memcached 缓存后端。内存缓存运行在内存中，它在 RAM 中分配了指定的数量。当分配的 RAM 满了时，Memcahed 就会移除最老的数据来保存新的数据。

在这个网址下载 Memcached： http://memcached.org/downloads 。如果你使用 Linux, 你可以使用下面的命令安装：

```
./configure && make && make test && sudo make install
```

如果你在使用 Mac OS X, 你可以使用命令 `brew install Memcached` 通过 Homebrew 包管理器来安装 Memcached 。你可以在这里下载 Homebrew `http://brew.sh`

如果你正在使用 Windwos ，你可以在这里找到一个 Windows 的 Memcached 二进制版本：http://code.jellycan.com/memcached/ 。

在安装 Memcached 之后，打开 shell ，使用下面的命令运行它：

```
memcached -l 127.0.0.1:11211
```

Memcached 将会默认地在 11211 运行。当然，你也可以通过 `-l` 选项指定一个特定的主机和端口。你可以在这里找到更多关于 Memcached 的信息：http://memcached.org 。

在安装 Memcached 之后，你需要安装它的 Python 绑定（bindings）。使用下面的命令安装：】

```
python install python3-memcached==1.51
```

##**缓存设置**
Django 提供了如下的缓存设置：

* `CACHES`：一个包含所有可用的项目缓存。
* `CACHE_MIDDLEWARE_ALIAS`：用于储存的缓存别名。
* `CACHE_MIDDLEWARE_KEY_PREFIX`：缓存键的前缀。设置一个缓存前缀来避免键的冲突，如果你在几个站点中分享相同的缓存的话。
* `CACHE_MIDDLEWARE_SECONDS`	：默认的缓存页面秒数

项目的缓存系统可以使用 `CACHES` 设置来配置。这个设置是一个字典，让你可以指定多个缓存的配置。每个 `CACHES` 字典中的缓存可以指定下列数据：

* `BACKEND`：使用的缓存后端。
* `KEY_FUNCTION`：包含一个指向回调函数的点路径的字符，这个函数以prefix(前缀)、verision(版本)、和 key (键) 作为参数并返回最终缓存键（cache key）。
* `KEY_PREFIX`：一个用于所有缓存键的字符串，避免冲突。
* `LOCATION`：缓存的位置。基于你的缓存后端，这可能是一个路径、一个主机和端口，或者是内存中后端的名字。
* `OPTIONS`：任何额外的传递向缓存后端的参数。
* `TIMEOUT`：默认的超时时间，以秒为单位，用于储存缓存键。默认设置是 300 秒，也就是五分钟。如果把它设置为 `None` ，缓存键将不会过期。
* `VERSION`：默认的缓存键的版本。对于缓存版本是很有用的。

##**把 memcached 添加进你的项目**
让我们为我们的项目配置缓存。编辑 `educa` 项目的 `settings.py` 文件，添加以下代码：

```python
CACHES = {
	'default': {
		'BACKEND': 'django.core.cache.backends.memcached.
MemcachedCache',
	'LOCATION': '127.0.0.1:11211',
	}
}
```

我们正在使用 `MemcachedCache` 后端。我们使用 `address:port` 标记指定了它的位置。如果你有多个 `memcached` 实例，你可以在 `LOCATION` 中使用列表。

##**监控缓存**
这里有一个第三方包叫做 django-memcached-status ，它可以在管理站点展示你的项目的 `memcached` 实例的统计数据。为了兼容 Python3**（译者夜夜月注：python3大法好。）** ，从下面的分支中安装它：

```
pip install git+git://github.com/zenx/django-memcached-status.git
```

编辑 `settings.py` ，然后把 `memcached_status` 添加进 `INSTALLED_APPS` 设置中。确保 memcached 正在运行，在另外一个 shell 中打开开发服务器，然后访问 http://127.0.0.1:8000/adim/ ，使用超级用户登录进管理站点，你就可以看到如下的区域：

![django-11-6](http://ohqrvqrlb.bkt.clouddn.com/django-11-6.png)

这张图片展示了缓存使用。绿色代表了空闲的缓存，红色的表示使用了的空间。如果你点击方框的标题，它展示了你的 memcached 实例的统计详情。

我们已经为项目安装好了 memcached 并且可以监控它。让我们开始缓存数据吧！

##**缓存级别**
Django 提供了以下几个级别按照颗粒度上升的缓存排列：

* `Low-level cache API`：提供了最高颗粒度。允许你缓存具体的查询或计算结果。
* `Per-view cache`：提供单一视图的缓存。
* `Template cache`：允许你缓存模板片段。
* `Per-site cache`：最高级的缓存。它缓存你的整个网站。

>在你执行缓存之请仔细考虑下缓存策略。首先要考虑那些不是单一用户为基础的查询和计算开销

##**使用 low-level cache API （低级缓存API）**
低级缓存 API 让你可以缓存任意颗粒度的对象。它位于 `django.core.cache` 。你可以像这样导入它：

```python
from django.core.cache import cache
```

这使用的是默认的缓存。它相当于 `caches['default']` 。通过它的别名来连接一个特定的缓存也是可能的：

```python
from django.core.cache import caches
my_cache = caches['alias']
```

让我们看看缓存 API 是如何工作的。使用命令 `python manage.py shell` 打开 shell 然后执行下面的代码：

```
>>> from django.core.cache import cache
>>> cache.set('musician', 'Django Reinhardt', 20)
```

我们连接的是默认的缓存后端，使用 `set{key,value, timeout}` 来保存一个名为 `musician` 的键和它的为字符串 `Django Reinhardt` 的值 20 秒钟。如果我们不指定过期时间，Django 会使在 `CACHES` 设置中缓存后端的默认过期时间。现在执行下面的代码：

```
>>> cache.get('musician')
'Django Reinhardt'
```

我们在缓存中检索键。等待 20 秒然后指定相同的代码：

```
>>> cache.get('musician')
None
```

`musician` 缓存键已经过期了，`get()` 方法返回了 None 因为键已经不在缓存中了。

>在缓存键中要避免储存 None 值，因为这样你就无法区分缓存值和缓存过期了


让我们缓存一个查询集：

```
>>> from courses.models import Subject
>>> subjects = Subject.objects.all()
>>> cache.set('all_subjects', subjects)
```

我们执行了一个在 `Subject` 模型上的查询集，然后把返回的对象储存在 `all_subjects` 键中。让我们检索一下缓存数据:

```
>>> cache.get('all_subjects')
[<Subject: Mathematics>, <Subject: Music>, <Subject: Physics>,
<Subject: Programming>]
```

我们将会在视图中缓存一些查询集。编辑 `courses` 应用的 `views.py` ，添加以下 导入：

```python
from django.core.cache import cache
```

在 `CourseListView` 的 `get()` 方法，把下面这几行：

```python
subjects = Subject.objects.annotate(
		total_courses=Count('courses'))
```

替换为：

```python
subjects = cache.get('all_subjects')
if not subjects:
	subjects = Subject.objects.annotate(
				total_courses=Count('courses'))
	cache.set('all_subjects', subjects)
```

在这段代码中，我们首先尝试使用 `cache.get()` 来从缓存中得到 `all_students` 键。如果所给的键没有找到，返回的是 None 。如果键没有被找到（没有被缓存，或者缓存了但是过期了），我们就执行查询来检索所有的 `Subject` 对象和它们课程的数量，我们使用 `cache.set()` 来缓存结果。

打开代发服务器，访问 http://127.0.0.1:8000 。当视图被执行的时候，缓存键没有被找到的话查询集就会被执行。访问 http://127.0.0.1:8000/adim/ 然后打开 memcached 统计。你可以看到类似于下面的缓存的使用数据：

![django-11-7](http://ohqrvqrlb.bkt.clouddn.com/django-11-7.png)

看一眼 **Curr Items** 应该是 1 。这表示当前有一个内容被缓存。**Get Hits** 表示有多少的 get 操作成功了，**Get Miss**表示有多少的请求丢失了。**Miss Ratio** 是使用它们俩来计算的。

现在导航回 http://127.0.0.1:8000/ ，重载页面几次。如果你现在看缓存统计的话，你就会看到更多的（**Get Hits**和**Cmd Get**被执行了）

##**基于动态数据的缓存**
有很多时候你都会想使用基于动态数据的缓存的。基于这样的情况，你必须要创建包含所有要求信息的动态键来特别定以缓存数据。编辑 `courses` 应用的 `views.py` ，修改 `CourseListView` ，让它看起来像这样：

```python
class CourseListView(TemplateResponseMixin, View):
	model = Course
	template_name = 'courses/course/list.html'
	
	def get(self, request, subject=None):
		subjects = cache.get('all_subjects')
		if not subjects:
			subjects = Subject.objects.annotate(
							total_courses=Count('courses'))
			cache.set('all_subjects', subjects)
		all_courses = Course.objects.annotate(
			total_modules=Count('modules'))
		if subject:
			subject = get_object_or_404(Subject, slug=subject)
			key = 'subject_{}_courses'.format(subject.id)
			courses = cache.get(key)
			if not courses:
				courses = all_courses.filter(subject=subject)
				cache.set(key, courses)
		else:
			courses = cache.get('all_courses')
			if not courses:
				courses = all_courses
				cache.set('all_courses', courses)
		return self.render_to_response({'subjects': subjects,
										'subject': subject,
										'courses': courses})
```

在这个场景中，我们把课程和根据科目筛选的课程都缓存了。我们使用 `all_courses` 缓存键来储存所有的课程，如果没有给科目的话。如果给了一个科目的话我们就用 `'subject_()_course'.format(subject.id)`动态的创建缓存键。

注意到我们不能用一个缓存查询集来创建另外的查询集是很重要的，因为我们已经缓存了当前的查询结果，所以我们不能这样做：

```python
courses = cache.get('all_courses')
courses.filter(subject=subject)
```

相反，我们需要创建基本的查询集 ` Course.objects.annotate(total_modules=Count('modules'))` ，它不会被执行除非你强制执行它，然后用它来更进一步的用 `all_courses.filter(subject=subject)` 限制查询集万一数据没有在缓存中找到的话。

##**缓存模板片段**
缓存模板片段是一个高级别的方法。你需要使用 `{% load cache %}` 在模板中载入缓存模板标签。然后你就可以使用 `{%  cache %}` 模板标签来缓存特定的模板片段了。你通常可以像下面这样使用缓存标签：

```html
{% cache 300 fragment_name %}
...
{% endcache %}
```

`{%  cache %}` 标签要求两个参数：过期时间，以秒为单位，和一个片段名称。如果你需要缓存基于动态数据的内容，你可以通过传递额外的参数给 `{% cache %}` 模板标签来特别的指定片段。

编辑 `students` 应用的 `/students/course/detail.html` 。在顶部添加以下代码，就在 `{% extends %}` 标签的后面：

```html
{% load cache %}
```

然后把下面几行：

```html
{% for content in module.contents.all %}
    {% with item=content.item %}
        <h2>{{ item.title }}</h2>
        {{ item.render }}
    {% endwith %}
{% endfor %}
```

替换为：

```html
{% cache 600 module_contents module %}
    {% for content in module.contents.all %}
        {% with item=content.item %}
            <h2>{{ item.title }}</h2>
            {{ item.render }}
        {% endwith %}
    {% endfor %}
{% endcache %}
```

我们使用名字 `module_contents` 和传递当前的 `Module` 对象来缓存模板片段。这对于当请求不同的模型是避免缓存一个模型的内容和服务错误的内容来说是很重要的。

>如果 `USE_I18N` 设置是为 True，单一站点的中间件缓存将会遵照当前激活的语言。如果你使用了 `{%  cache %}` 模板标签以及可用翻译特定的变量中的一个，那么他们的效果将会是一样的，比如：`{%  cache 600 name request.LANGUAGE_CODE %}`

##**缓存视图**
你可以使用位于 `django.views.decrators.cache` 的 `cache_page` 装饰器来焕春输出的单个视图。装饰器要求一个过期时间的参数（以秒为单位）。

让我们在我们的视图中使用它。编辑 `students` 应用的 `urls.py` ，添加以下 导入：

```python
from django.views.decorators.cache import cache_page
```

然后按照如下在 `student_course_detail_module` URL 模式上应用 `cache_page` 装饰器：

```python
url(r'^course/(?P<pk>\d+)/$',
	cache_page(60 * 15)(views.StudentCourseDetailView.as_view()),
	name='student_course_detail'),

url(r'^course/(?P<pk>\d+)/(?P<module_id>\d+)/$',
	cache_page(60 * 15)(views.StudentCourseDetailView.as_view()),
	name='student_course_detail_module'),
```

现在 `StudentCourseDetailView` 的结果就会被缓存 15 分钟了。

>单一的视图缓存使用 URL 来创建缓存键。多个指向同一个视图的 URLs 将会被分开储存

##**使用单一站点缓存**
这是最高级的缓存。他让你可以缓存你的整个站点。

为了使用单一站点缓存，你需要编辑项目中的 `settings.py` ，把 ` UpdateCacheMiddleware` 和  `FetchFromCacheMiddleware` 添加进 `MIDDLEWARE_CLASSES` 设置中：

```python
MIDDLEWARE_CLASSES = (
	'django.contrib.sessions.middleware.SessionMiddleware',
	'django.middleware.cache.UpdateCacheMiddleware',
	'django.middleware.common.CommonMiddleware',
	'django.middleware.cache.FetchFromCacheMiddleware',
	'django.middleware.csrf.CsrfViewMiddleware',
	# ...
)
```

记住在请求的过程中，中间件是按照所给的顺序来执行的，在相应过程中是逆序执行的。`UpdateCacheMiddleware` 被放在 `CommonMiddleware` 之前，因为它在相应时才执行，此时中间件是逆序执行的。`FetchFromCacheMiddleware` 被放在 `CommonMiddleware` 之后，是因为它需要连接后者的的请求数据集。

然后，把下列设置添加进 `settings.py` 文件：

```python
CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 60 * 15 # 15 minutes
CACHE_MIDDLEWARE_KEY_PREFIX = 'educa'
```

在这些设置中我们为中间件使用了默认的缓存，然后我们把全局缓存过期时间设置为 15 分钟。我们也指定了所有的缓存键前缀来避免冲突万一我们为多个项目使用了相同的 memcached 后端。我们的站点现在将会为所有的 GET 请求缓存以及返回缓存内容。

我们已经完成了这个来测试单一站点缓存功能。尽管，以站点的缓存对于我们来说是不怎么合适的，因为我们的课程管理视图需要展示更新数据来实时的给出任何的修改。我们项目中的最好的方法是缓存用于展示给学生的课程内容的模板或者视图数据。

我们已经大致体验过了 Django 提供的方法来缓存数据。你应合适的定义你自己的缓存策略，优先考虑开销最大的查询集或者计算。

##**总结**
在这一章中，我们创建了一个用于课程的公共视图，创建了一个用于学生注册和报名课程的系统。我们安装了 memcached 以及实现了不同级别的缓存。

在下一章中，我们将会为你的项目创建 RESTful API。

（译者 @夜夜月注：终于只剩下最后一章了！）

