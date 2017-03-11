（译者注：第三章滚烫出炉，大家请不要吐槽文中图片比较模糊，毕竟都是从PDF中截图出来的，有点丢像素，大致能看就行- -，另外还是渣翻，但个人觉的比前两章翻译的稍微进步了那么一点点- -，希望后面几章翻译的越来越溜，就这样）

#第三章
##扩展你的blog应用
在上一章中我们学习了表单的基础和在你的项目集成第三方的应用。这一章将会覆盖以下内容：

* 创建定制的模板标签（template tags)和过滤器（filters）
* 添加一个站点地图和帖子反馈（post feed）
* 使用Solr和Haystack构建一个搜索引擎

###创建定制的模板标签（template tags)和过滤器（filters）
Django提供了很多内置的标签（tags），例如`{% if %}`或者`{% block %}`。你已经在你的模板（template）中使用过一些了。你可以在https://docs.djangoproject.com/en/1.8/ ref/templates/builtins/ 中找到更多关于内置模板标签（template tags）以及过滤器（filter）的参考。

当然，Django也允许你创建自己的模板标签（template tags）来执行定制的动作。当Django的内置模板标签（template tags)无法提供你需要在模板（template）执行的功能，你会发现能创建定制的模板标签（template tags）非常的有用。

###创建定制的模板标签（template tags）
Django提供了以下帮助函数（functions）来允许你通过一种简单的方式创建自己的模板标签（template tags）：

* simple_tag：处理数据并返回一个字符串（string）
* inclusion_tag：处理数据并返回一个渲染过的模板（template）
* assignment_tag：处理数据并在上下文（context）中放置一个变量（variable）

模板标签（template tags）必须存在Django的应用中。

进入你的blog应用目录，创建一个新的目录命令为*templatetags*然后在该目录下创建一个空的*__init__.py*文件。接着在该目录下继续创建一个文件并命名为*blog_tags.py*。到此，我们的blog应用文件结构应该如下所示：

    blog/
        __init__.py
        models.py
        ...
        templatetags/
            __init__.py
            blog_tags.py

文件的命令是非常重要的。你需要使用这些模块的命名在模板（template）中加载你的标签（tags)。

我们将要开始创建一个简单标签（simple tag）来获取blog中所有已发布的帖子总数。编辑你刚才创建的*blog_tags.py*文件，加入以下代码：
    
    from django import template
	
    register = template.Library()
	
    from ..models import Post
	
    @register.simple_tag
    def total_posts():
        return Post.published.count()

到目前为止，我们已经创建了一个简单的模板标签（template tag）用来取回所有已发布的帖子总数。每一个模板标签（template tags）都需要包含一个叫做*register*的变量来表明自己是一个有效的标签（tag）库。这个变量是*template.Library*的一个实例，它是用来注册你自己的模板标签（template tags）和过滤器（filter）的。我们定义了一个名为*total_posts*的标签，通过编写一个Python函数并且使用了`@register.simple-tag`装饰器使之成为一个简单标签（tag）并登记注册。

Django将会使用这个函数名作为标签（tag）名。如果你想使用别的名字来注册这个标签（tag），你可以指定装饰器的*name*属性，比如`@register.simple_tag(name='my_tag')`。

> 在添加了新的模板标签（template tags）模块后，你必须重启Django开发服务才能使用新的模板标签（template tags)和过滤器（filters)。

在使用定制的模板标签（template tags)之前，你必须在模板（template）中使用*{% load %}*来加载它们才能生效。就像之前提到的，你需要使用包含了你的模板标签（template tags)和过滤器（filter)的Python模块的名字。打开*blog/base.html*模板（template）在顶部添加*{% load blog_tags %}*加载你自己的模板标签（template tags)模块。之后使用你创建的标签（tag）来显示你的帖子总数。只需要在你的模板（template）中添加*{% total_posts %}*。最新的模板（template）看上去如下所示：

    {% load blog_tags %}
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
      <p>This is my blog. I've written {% total_posts %} posts so far.</p>
      </div>
    </body>
    </html>
    
我们需要重启服务来保证新的文件被加载到项目中。使用*Ctrl+C*停止服务然后通过以下命令再次启动：
    
    python manage.py runserver
    
在浏览器中打开 http://127.0.0.1:8000/blog/ 。你会看到帖子的总数展示在站点的侧边栏（sidebar），如下所示：

![django-3-1](http://ohqrvqrlb.bkt.clouddn.com/django-3-1.png)


定制模板标签（template tags）的作用是你可以处理任何的数据并且在模板（template）中添加它而不用去担心视图（view）是否已经执行过。你可以运行查询集（QuerySets）或者处理任何数据在你的模板（template）中展示结果。

现在，我们要创建另一个标签（tag）可以在我们blog的侧边栏（sidebar）展示最新的几个帖子。这一次我们要使用一个包含标签（inclusion tag）。通过使用一个包含标签（inclusion tag），你可以通过你的模板标签（template tags）返回的上下文变量（context variables）来渲染模板（template）。编辑*blog_tags.py*文件，添加如下代码：

    @register.inclusion_tag('blog/post/latest_posts.html')
    def show_latest_posts(count=5):
        latest_posts = Post.published.order_by('-publish')[:count]
        return {'latest_posts': latest_posts}

在以上代码中，我们通过装饰器*@register.inclusion_tag*注册模板标签（template tag），然后我们指定模板（template）必须被*blog/post/latest_posts.html*返回的值渲染。我们的模板标签（template tag)将会接受一个可选的*count*参数（默认是5）允许我们指定我们想要显示的帖子数量。我们使用这个变量来限制`Post.published.order_by('-publish')[:count]`查询返回的结果。请注意，这个函数返回了一个变量字典替代了一个简单的值。包含标签（inclusion tags）必须返回一个包含值的字典用作上下文（context）来渲染指定的模板（template）。包含标签（inclusion tags）返回一个字典。这个我们刚创建的模板标签（template tag）可以接受的想要显示的帖子数（可选），类似*{% show_latest_posts 3 %}。

现在，在*blog/post/*下创建一个新的模板（template）文件并且命名为*latest_posts.html*。在该文件中添加如下代码：

    <ul>
    {% for post in latest_posts %}
      <li>
        <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
    </ul>

在这里，我们通过使用模板标签（template tag）返回的*latest_posts*变量展示了一个无序的帖子列。现在，编辑*blog/base.hmtl*模板（template）添加这个新的模板标签（template tag）来展示最新的3条帖子。侧边栏（sidebar）区块（block）看上去应该如下所示：
    
    <div id="sidebar">
      <h2>My blog</h2>
      <p>This is my blog. I've written {% total_posts %} posts so far.</p>
      <h3>Latest posts</h3>
      {% show_latest_posts 3 %}
    </div>
    
这个模板标签（template tag）被调用而且传递了需要展示的帖子数（原文此处 number of comments，应该是写错了)。当前模板（template）在给予上下文（context）的位置会被渲染。

现在，回到浏览器刷新这个页面，你会看到如下图所示：

![django-3-2](http://ohqrvqrlb.bkt.clouddn.com/django-3-2.png)

最后，我们来创建一个分配标签（assignment tag）。分配标签（assignment tag）类似简单标签（simple tags）但是他们能将变量存储在给予的变量中。我们将会创建一个分配标签（assignment tag）来展示拥有最多评论的帖子。编辑*blog_tags.py*文件，添加如下代码：

    from django.db.models import Count
	
    @register.assignment_tag
    def get_most_commented_posts(count=5):
        return Post.published.annotate(
                    total_comments=Count('comments')
                ).order_by('-total_comments')[:count]
        
这个查询集（QuerySet）使用*annotate()*函数来进行聚合查询，使用了*Count*聚合函数。我们构建了一个查询集（QuerySet）聚合了每一个帖子的评论总数保存在*total_comments*字段中，接着我们通过这个字段对查询集（QuerySet）进行排序。我们还提供了一个可选的*count*变量通过给予的值来限制返回的帖子数量。

除了*Count*以外，Django还提供了不少聚合函数，例如*Avg,Max,Min,Sum*.你可以在 https://docs.djangoproject.com/en/1.8/topics/db/aggregation/ 页面获取到更多的聚合方法。

编辑*blog/base.html*模板（template），在侧边栏（sidebar）`<div>`元素中添加如下代码：

    <h3>Most commented posts</h3>
    {% get_most_commented_posts as most_commented_posts %}
    <ul>
    {% for post in most_commented_posts %}
      <li>
        <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
    </ul>
    
使用分配模板标签（assignment template tags）的方法是*{% template_tag as variable %}*。例如我们的模板标签（template tag），我们使用*{% get_most_commented_posts as most_commented_posts %}*。 使用这样的方式，我们可以存储这个模板标签（template tag）返回的结果到一个新的*most_commented_posts*变量中。这样，我们就可以通过一个无序列表(unordered list)显示返回的帖子。

现在，打开浏览器刷新页面来看下最终的结果，应该如下图所示：

![django-3-3](http://ohqrvqrlb.bkt.clouddn.com/django-3-3.png)

你可以在 https://docs.djangoproject.com/en/1.8/howto/custom-template-tags/ 页面得到更多的关于定制模板标签（template tags）的信息。

###创建定制的模板过滤器（template filters）
Django拥有多种内置的模板过滤器（template filters）允许你对模板（template）中的变量进行修饰。这些过滤器其实就是Python函数提供了一个或两个参数————第一个就是需要处理的变量值，第二个是可选的参数。它们返回的值可以被展示或者被别的过滤器（filters）继续处理。一个过滤器（filter）类似*{{ variable|my_filter }}*或者再带上一个参数，类似*{{ variable|my_filter:"foo" }}*。你可以对一个变量一次性调用多个过滤器（filter），类似*{{ variable|filter1|filter2 }}*, 每一个过滤器（filter）都会对上一个过滤器（filter）输出的结果进行过滤。

我们这就创建一个定制的过滤器（filter）使我们的blog帖子内容可以使用markdown语法然后在模板（template）中将帖子内容转变为HTML。Markdown是一种非常容易使用的文本格式化语法并且它可以转变为HTML。你可以在 http://daringfireball.net/projects/markdown/basics 页面学习这方面的知识。

首先，通过pip渠道安装Python的markdown模板：

    pip install Markdown==2.6.2
    
之后编辑*blog_tags.py*文件，包含如下代码：

    from django.utils.safestring import mark_safe
    import markdown
	
    @register.filter(name='markdown')
    def markdown_format(text):
        return mark_safe(markdown.markdown(text))

我们使用和模板标签（template tags）一样的方法来注册我们自己的模板过滤器（template filter）。为了避免我们的函数名和*markdown*模板名起冲突，我们将我们的函数命名为*markdown_format*，然后将过滤器（filter）命名为*markdown*，在模板（template）中的使用方法类似*{{ variable|markdown }}*。Django会转义过滤器（filter）生成的HTML代码。我们使用Django提供的*mark_safe*方法用来标记结果确保在模板（template）中渲染成安全的HTML。默认的，Django不会信赖任何HTML代码并且在输出之前会进行转义。唯一的例外就是被标记为安全转义的变量。这样的操作可以阻止Django从输出中执行潜在的危险的HTML，并且允许你创造一些例外情况只要你知道你正在运行的是安全的HTML。

现在，在帖子列表和详情模板（template）中加载你的模板标签（template tags）模块。在*post/list.html*和*post/detail.html*模板（template）的顶部*{% extends %}*的后方添加如下代码：

    {% load blog_tags %}
    
在*post/detail.thml*模板中，替换以下内容：

    {{ post.body|linebreaks }}

替换成：

    {{ post.body|markdown }}
    
之后，在*post/list.html*文件中，替换以下内容：

    {{ post.body|truncatewords:30|linebreaks }}
    
替换成：

    {{ post.body|markdown|truncatewords_html:30 }}
    
*truncateword_html*过滤器（filter）根据后方给予的单词数量来缩短字符串，避免没有关闭的HTML标签（tags）。

现在，打开 http://127.0.0.1:8000/admin/blog/post/add/ ,添加一个帖子，内容如下所示：

    This is a post formatted with markdown
    --------------------------------------
    *This is emphasized* and **this is more emphasized**.
    Here is a list:
    * One
    * Two
    * Three
    And a [link to the Django website](https://www.djangoproject.com/)

在浏览器中查看帖子的渲染情况，你会看到如下图所示：

![django-3-4](http://ohqrvqrlb.bkt.clouddn.com/django-3-4.png)

就像你所看到的，定制的模板过滤器（template filters）对于自定义的格式化是非常有用的。你可以在 https://docs.djangoproject.com/en/1.8/howto/custom-template-tags/#writing-custom-templatefilters 页面获取更多关于定制过滤器（filter）的信息。

###为你的站点添加一个站点地图（sitemap）
Django自带一个站点地图（sitemap）框架，允许你为你的站点动态生成站点地图（sitemap）。一个站点地图（sitemap）是一个xml文件，它会告诉搜索引擎你的网站中存在的页面，它们的关联和它们更新的有多频繁。通过使用站点地图（sitemap），你可以帮助网络爬虫（crawlers）来对你的网站内容进行索引和标记。

Django站点地图（sitemap）框架依赖*django.contrib.sites*模块，这个模块允许你对对象们进行关联来详细说明你的项目中正在运行的网站。这为你需要使用一个单独的Django项目来运行多个站点提供了便利。为了安装站点地图（sitemap）框架，我们需要在项目中激活*sites*和*sitemap*这两个应用。编辑项目中的*settings.py*文件在*INSTALLED_APPS*中添加*django.contrib.sites*和*django.contrib.sitemaps*。之后为站点ID定义一个新的设置，如下所示：

    SITE_ID = 1
    # Application definition
    INSTALLED_APPS = (
    # ...
    'django.contrib.sites',
    'django.contrib.sitemaps',
    )
    
现在，运行以下命令在数据库中为Django的站点应用创建所需的表：

    python manage.py migrate
    
你会看到如下的输出内容：

    Applying sites.0001_initial... OK

*sites*应用现在已经在数据库中进行了同步。现在，在你的*blog*应用目录下创建一个新的文件命名为*sitemaps.py*。打开这个文件，输入以下代码：

    from django.contrib.sitemaps import Sitemap
    from .models import Post
	
    class PostSitemap(Sitemap):
        changefreq = 'weekly'
        priority = 0.9
        def items(self):
            return Post.published.all()
        def lastmod(self, obj):
            return obj.publish
            
通过继承*sitemaps*模块提供的*Sitemap*类我们创建一个定制的站点地图（sitemap）。*changefreq*和*priority*属性表明了帖子页面修改的频率和它们在网站中的关联性（最大值是1）。*items()*方法返回了对象的查询集（QuerySet）用来包含在这个站点地图（sitemap）中。默认的，Django在每个对象中调用*get_absolute_url()*方法来获取它的URL。请记住，这个方法是我们在第一章（创建一个blog应用）中创建的，用来返回每个帖子的标准URL。如果你想为每个对象指定URL，你可以在你的站点地图（sitemap）类中添加一个*location*方法。*lastmode*方法接收*items()*返回的每一个对象并且返回最新的一个被修改过的对象。*changefreq*和*priority*方法都能成为另一个方法或者属性。你可以在Django的官方文档 https://docs.djangoproject.com/en/1.8/ref/contrib/sitemaps/ 页面中获取更多的站点地图（sitemap）参考。

最后，我们还需要添加我们的站点地图（sitemap）URL。编辑项目中的主*urls.py文件，如下所示添加站点地图（sitemap）：

    from django.conf.urls import include, url
    from django.contrib import admin
    from django.contrib.sitemaps.views import sitemap
    from blog.sitemaps import PostSitemap
    
	sitemaps = {
        'posts': PostSitemap,
    }
	
    urlpatterns = [
        url(r'^admin/', include(admin.site.urls)),
        url(r'^blog/',
            include('blog.urls'namespace='blog', app_name='blog')),
        url(r'^sitemap\.xml$', sitemap, {'sitemaps': sitemaps},
            name='django.contrib.sitemaps.views.sitemap'),
    ]
    
在这里，我们加入了必须的导入并定义了一个*sitemaps*的字典。我们定义了一个URL模式来匹配*sitemap.xml*并使用*sitemap*视图（view）。*sitemaps*字典会被传递到*sitemap*视图（view）中。现在，在浏览器中打开 http://127.0.0.1:8000/sitemap.xml 。你会看到类似下方的XML代码：

    <?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
        <url>
            <loc>http://example.com/blog/2015/09/20/another-post/</loc>
            <lastmod>2015-09-29</lastmod>
            <changefreq>weekly</changefreq>
            <priority>0.9</priority>
        </url>
        <url>
            <loc>http://example.com/blog/2015/09/20/who-was-djangoreinhardt/</loc>
            <lastmod>2015-09-20</lastmod>
            <changefreq>weekly</changefreq>
            <priority>0.9</priority>
        </url>
    </urlset>

每一个帖子都会根据它的*get_absolute_url()*方法构建URL。如同我们之前在站点地图(sitemap)中所指定的，*lastmode*属性对应该帖子的*publish*日期字段，*changefreq*和*priority*属性也从我们的*PostSitemap*类中带入。你能看到被用来构建URL的域（domain）是*example.com*。这个域（domain）来自一个存储在数据库中的*Site*对象。这个默认的对象是在我们之前同步*sites*框架数据库时创建的。打开  http://127.0.0.1:8000/admin/sites/site/ ，你会看到如下图所示：

![django-3-5](http://ohqrvqrlb.bkt.clouddn.com/django-3-5.png)

这个列表是为*sites*框架显示的管理视图（admin view）。在这里，你可以设置域（domain）或者主机（host）给*sites*框架使用，而且*sites*应用也依赖它们。为了生成能在我们本地环境可用的URL，更改域（domain）名为*127.0.0.1:8000*，如下图所示：

![django-3-6](http://ohqrvqrlb.bkt.clouddn.com/django-3-6.png)

为了开发需要我们指向了我们本地主机。在生产环境中，你必须使用你自己的域（domain)名给*sites*框架。

###为你的blog帖子创建feeds
>译者注：这节中有不少英文单词，例如feed，syndication，Atom等，没有比较好的翻译，很多地方也都是直接不翻译保留原文，所以我也保留原文）

Django有一个内置的syndication feed框架，你可以用类似的方式（manner）通过使用*sites*框架创建站点地图（sitemap）来动态（dynamically）生成RSS或者Atom feeds。

在blog应用的目录下创建一个新文件命名为*feeds.py*。添加如下代码：

    from django.contrib.syndication.views import Feed
	from django.template.defaultfilters import truncatewords
	from .models import Post
	
	class LatestPostsFeed(Feed):
		title = 'My blog'
		link = '/blog/'
		description = 'New posts of my blog.'
		
		def items(self):
			return Post.published.all()[:5]
			
		def item_title(self, item):
			return item.title
			
		def item_description(self, item):
			return truncatewords(item.body, 30)
			

首先，我们继承了syndication框架的*Feed*类创建了一个子类。其中的*title，link，description*属性各自对应RSS中的`<title>`,`<link>`,`<description>`元素。

*items()*方法取回的对象们会被包含在feed中。我们只给这个feed取回最新五个已发布的帖子。*item_title()*和*item_description()*方法取回*items()*返回的每个对象然后返回各自的标题和描述信息给每个item。我们使用内置的*truncatewords*模板过滤器（template filter）构建帖子的描述信息并只保留最前面的30个单词。

现在，编辑blog应用下的*urls.py*文件，导入你刚创建的*LatestPostsFeed*，在新的URL模式（pattern）中实例化feed：

    from .feeds import LatestPostsFeed
	
	urlpatterns = [
		# ...
		url(r'^feed/$', LatestPostsFeed(), name='post_feed'),
	]
    
在浏览器中转到 http://127.0.0.1:8000/blog/feed/ 。你会看到最新的5个blog帖子的RSS feedincluding：

    <?xml version="1.0" encoding="utf-8"?>
	
	<rss xmlns:atom="http://www.w3.org/2005/Atom" version="2.0">
		<channel>
			<title>My blog</title>
			<link>http://127.0.0.1:8000/blog/</link>
			<description>New posts of my blog.</description>
			<atom:link href="http://127.0.0.1:8000/blog/feed/" rel="self"/>
			<language>en-us</language>
			<lastBuildDate>Sun, 20 Sep 2015 20:40:55 -0000</lastBuildDate>
			<item>
				<title>Who was Django Reinhardt?</title>
				<link>http://127.0.0.1:8000/blog/2015/09/20/who-was-django-reinhardt/</link>
				<description>The Django web framework was named after the amazing jazz guitarist Django Reinhardt.</description>
				<guid>http://127.0.0.1:8000/blog/2015/09/20/who-was-django-reinhardt/</guid>
			</item> 
			...
		</channel>
	</rss>
    
如果你在一个RSS客户端中打开相同的URL，你会通过一个非常人性化的接口看到你的feed。

最后一步是在blog的侧边栏（sitebar）添加一个feed订阅（subscription）里链接。打开*blog/base.html*模板（template）在侧边栏（sitebar）的*div*中的帖子总数下添加如下代码：

    <p><a href="{% url "blog:post_feed" %}">Subscribe to my RSS feed</a></p>
    
现在，在浏览器中打开 http://127.0.0.1:8000/blog/ 看下侧边栏（sitebar）。这个新链接将会带你去blog的feed：

![django-3-7](http://ohqrvqrlb.bkt.clouddn.com/django-3-7.png)

###使用Solr和Haystack添加一个搜索引擎
>译者注:终于到了这一节，之前自己按照本节一步步操作的走下来添加了一个搜索引擎但是并没有达到像本节中所说的效果，期间还出现了很多莫名其妙的错误，可以说添加的搜索引擎功能失败了，而且本节中的提到的工具版本过低，官网最新版本的操作步骤已经和本节中描述的不一样，本节的翻译就怕会带来更多的错误，大家有需要尽量去阅读下英文原版。另外，一些单词没有好的翻译我还是保留原文。

现在，我们要为我们的blog添加搜索功能。Django ORM允许你通过使用*icontains*过滤器（filter）执行对大小写不敏感的查询操作。举个例子，你可以使用以下的查询方式来找到内容中包含单词*framework*的帖子：
    
    Post.objects.filter(body__icontains='framework')
    
但是，如果你想要更加强大的搜索功能，你需要使用合适的搜索引擎。我们准备使用*Solr*结合Django为我们的blog构建一个搜索引擎。*Solr*是一个非常流行的开源搜索平台，它提供了全文检索（full text search），term boosting，hit highlighting，分类搜索（faceted search）以及动态聚集（clustering），还有其他更多的高级搜索特性。

为了在我们的项目中集成*Solr*，我们需要使用*Haystack*。*Haystack*是一个为多个搜索引擎提供抽象层的工作 的Django应用。它提供了一个非常类似与Django查询集（QuerySets）的简单的API。让我们开始安装和配置*Solr*和*Haystack*。

###安装Solr
你需要1.7或更高的Java运行环境来安装Solr。你可以在终端中输入`java -version`来检查你的java版本。下方的输出和你的输出可能有所出入，但是你必须保证安装的版本至少也要是1.7的：

    java version "1.7.0_25"
	Java(TM) SE Runtime Environment (build 1.7.0_25-b15)
	Java HotSpot(TM) 64-Bit Server VM (build 23.25-b01, mixed mode)
    
如果你没有安装过Java或者版本过低没有达到要求，请到 http://www.oracle.com/technetwork/java/javase/downloads/index.html 下载。

确定Java版本后，从 http://archive.apache.org/dist/lucene/solr/ 下载**4.10.4**版本的Solr（译者注：请一定要下载这个版本，不然下面的操作无法进行！！）。解压下载的文件进入*Solr-4.10.4*文件夹中的*example*目录下（`cd solr-4.10.4/example/`)。这个目录下包含了一个*Solr*的现成配置。在这个目录下，运行*Solr*的内置Jetty web服务，通过以下命令：

    java -jar start.jar
    
在浏览器中打开 http://127.0.0.1:8983/solr/ 。你会看到如下图所示：

![django-3-8](http://ohqrvqrlb.bkt.clouddn.com/django-3-8.png)

以上是*Solr*的管理控制台。这个控制台给你展示了数据统计，允许你管理你的搜索后端，检测索引数据，并且执行查询操作。

###创建一个Solr core
Solr允许你隔离每一个core实例。每个Solr **core**是一个**全文搜索引擎**实例，包含了一个Solr配置，一个数据架构（schema），以及其他必须的配置才能使用。Slor允许你在页面上创建和管理cores。一个已经存在的参考例子配置中包含了一个core，名字叫*collection1*。如果你点击**Core Admin**菜单栏， 你可以看到这个core的信息，如下图所示：

![django-3-9](http://ohqrvqrlb.bkt.clouddn.com/django-3-9.png)

我们要为我们的blog应用创建一个core。首先，我们需要为我们的core创建文件结构。进入*solr-4.10.4/example/*目录下，创建一个新的目录命名为*blog*。然后在*blog*目录下创建空文件和目录，如下所示：

    blog/ 
        data/
		conf/
		protwords.txt
		schema.xml
		solrconfig.xml
		stopwords.txt
		synonyms.txt
		lang/
		stopwords_en.txt
                
在*solrconfig.xml*文件中添加如下XML代码：

    <?xml version="1.0" encoding="utf-8" ?>
	<config>
		<luceneMatchVersion>LUCENE_36</luceneMatchVersion>
		<requestHandler name="/select" class="solr.StandardRequestHandler" default="true" />
		<requestHandler name="/update" class="solr.UpdateRequestHandler" />
		<requestHandler name="/admin" class="solr.admin.AdminHandlers" />
		<requestHandler name="/admin/ping" class="solr.PingRequestHandler">
			<lst name="invariants">
				<str name="qt">search</str>
				<str name="q">*:*</str>
			</lst>
		</requestHandler>
	</config>

你还可以从本章的示例代码中拷贝该文件。这是一个最小的Solr配置。编辑*schema.xml*文件，加入如下XML代码：

    <?xml version="1.0" ?>
	<schema name="default" version="1.5">
	</schema>
    
这是一个空的**架构（schema）**。这个架构（schema）定义了一些被这个搜索引擎编入索引中字段以及它们的数据类型。我们之后要使用一个定制的架构（schema）。

现在，点击**Core Admin**菜单栏再点击**Add core**按钮。你会看到如下图所示：

![django-3-10](http://ohqrvqrlb.bkt.clouddn.com/django-3-10.png)

你需要在表单中填写一下数据

* name: blog
* instanceDir: blog
* dataDir: data
* config: solrconfig.xml
* schema: schema.xml

*name*字段是这个core的命名。*instanceDir*字段表明你的core的目录。*dataDir*是被编入索引的数据将要存放的目录。*config*字段是你的*Solr* XML配置文件名。*schema*字段是你的*Solr* XML 数据架构（schema)文件名。

现在，点击**Add Core**按钮。如果你看到下图所示，说明你的新core已经成功的添加到Solr中：

![django-3-11](http://ohqrvqrlb.bkt.clouddn.com/django-3-11.png)

###安装Haystack
为了在Django中使用Solr，我们还需要Haystack。通过pip渠道安装Haystack：

    pip install django-haystack==2.4.0
    
Haystack能和一些搜索引擎后台交互。要使用Solr后端，你还需要安装*pysolr*模块。运行如下命令：

    pip install pysolr==3.3.2
    
在*django-haystack*和*pysolr*完成安装后，你还需要在你的项目中激活*Haystack*。打开*settings.py*文件，在*INSTALLED_APPS*设置中添加*haystack*：

    INSTALLED_APPS = (
		# ...
		haystack', 
    )
    
你还需要为haystack定义搜索引擎后端。为此你要添加一个*HAYSTACK_CONNECTIONS*设置。在*settings.py*文件中添加如下内容：

    HAYSTACK_CONNECTIONS = {
		'default': {
			'ENGINE': 'haystack.backends.solr_backend.SolrEngine',
			'URL': 'http://127.0.0.1:8983/solr/blog'
		},
	}
    
要注意URL要指向我们的blog core。到此为止，Haystack已经安装好并且已经为使用Solr做好了准备。

###创建索引（indexex）
现在，我们必须将我们想要存储在搜索引擎中的模型进行注册。在Haystack中的惯例是在你的应用中创建一个*search_indexes.py*文件，然后在该文件中注册你的模型（models）。在你的blog应用目录下创建一个新的文件命名为*search_indexes.py*，添加如下代码：

    from haystack import indexes
	from .models import Post
	
	class PostIndex(indexes.SearchIndex, indexes.Indexable):
		text = indexes.CharField(document=True, use_template=True)
		publish = indexes.DateTimeField(model_attr='publish')
		
		def get_model(self):
			return Post
			
		def index_queryset(self, using=None):
			return self.get_model().published.all()

这是一个为*Post*模型（model）定制的*SearchIndex*。通过这个索引（index），我们告诉Haystack这个模型（model）中的数据必须被搜索引擎编入索引。这个索引（index）是通过继承*indexes.SearchIndex*和*indexes.Indexable*构建的。每一个*SearchIndex*都需要它的一个字段拥有`document=True`。按照惯例，这个字段命名为*text*。这个字段是一个主要的搜索字段。通过使用`use_template=True`，我们告诉Haystack这个字段将会被渲染成一个数据模板（template）来构建document，它会被搜索引擎编入索引（index）。*publish*字段是一个日期字段也会被编入索引。我们通过*model_attr*参数来表明这个字段对应*Post*模型（model）的*publish*字段。这个字段会被编入索引包含被编入索引的*Post*对象的*publish*字段的内容。

额外的字段例如我们接下来会提到的能给搜索提供非常有用的额外的过滤器（filters）。*get_model()*方法必须返回模型（model）给documents将会储存在这个索引中。*index_queryset()*方法给对象返回的查询集（QuerySet）也会被编入索引。请注意，我们只包含了发布状态的帖子。

现在，在blog应用的模板（templates）目录下创建目录和文件*search/indexes/blog/post_text.txt*，然后添加如下代码：

    {{ object.title }}
	{{ object.tags.all|join:", " }}
	{{ object.body }}
    
这是document模板（template）的默认路径，是给索引中的*text*字段使用的。Haystack使用应用名和模型（model）名来动态构建这个路径。每一次我们要对一个对象进行索引，Haystack都会基于这个模板（template）构建一个document，并且之后在Solr的搜索引擎中对这个document进行索引。

现在，我们已经有了一个定制的搜索索引（index），我们需要创建合适的Solr架构（schema）。Solr的配置基于XML，所以我们必须为我们即将编入索引（index）的数据生成一个XML架构（schema）。非常幸运，haystack提供了一个方法可以动态的生成架构（schema），基于我们的搜索索引（indexes）。打开终端，输入以下命令：

    python manage.py build_solr_schema

你会看到一个XML的输出内容。如果你看下生成的XML代码的地步，你会看到Haystack为你的*PostIndex*动态生成了字段：

    <field name="text" type="text_en" indexed="true" stored="true" multiValued="false" />
	<field name="publish" type="date" indexed="true" stored="true" multiValued="false" />
     
从`<?xml version="1.0"？>`开始拷贝所有输出的XML内容直到最后的标签（tag）`</schema>`，需要包含所有的标签（tags）。

这个XML架构（schema）是用来给数据做索引（index）到到Solr中。粘贴这个新的架构（schema）到你的Solr文件夹下的*example*目录下的*blog/conf/schema.xml*文件中。*schema.xml*文件也被包含在本章的示例代码中，所以你可以直接从示例代码中复制出来使用。

在浏览器中打开 http://127.0.0.1:8983/solr/ 然后点击**Core Admin**菜单栏，再点击**blog** core，然后再点击**Reload**按钮：

![django-3-12](http://ohqrvqrlb.bkt.clouddn.com/django-3-12.png)

我们重新载入这个core确保*schema.xml*的改变生效。当core重新载入完毕，新的架构（schema）才对新数据进行索引（index）做好准备。

###对数据进行索引（Indexing data）
让我们blog中的帖子编辑索引（index）到Solr中。打开终端，执行以下命令：

    python manage.py rebuild_index
    
你会看到如下警告：

    WARNING: This will irreparably remove EVERYTHING from your search index in connection 'default'. Your choices after this are to restore from backups or rebuild via the ‘rebuild_index’ command.
    Are you sure you wish to continue? [y/N]
    
输入y。Haystack将会整理搜索索引并且插入所有的发布状态的blog帖子中。你会看到如下输出：

    Removing all documents from your index because you said so. All documents removed. Indexing 4 posts
    
在浏览器中打开 http://127.0.0.1:8983/solr/#/blog 。在**Statistics*下方，你会看到有多少个documents被编入索引（indexed），如下所示：

![django-3-13](http://ohqrvqrlb.bkt.clouddn.com/django-3-13.png)

现在，在浏览器中打开 http://127.0.0.1:8983/solr/#/blog/query 。这是一个Solr提供的查询接口。点击*Execute query*按钮。默认的会请求在你的core中所有被编入索引（indexde）的documents。你会看到一串带有这个查询结果的*JSON*输出。输出的documents如下所示：

    {
		"id": "blog.post.1",
		"text": "Who was Django Reinhardt?\njazz, music\nThe Django web framework was named after the amazing jazz guitarist Django Reinhardt.",
		"django_id": "1",
		"publish": "2015-09-20T12:49:52Z",
		"django_ct": "blog.post"
	},
    
这是每个帖子在搜索索引（index）中存储的数据。*text*字段包含了标题，通过逗号分隔的标签（tags），还有帖子的内容，这个字段是在我们之前定义的模板（template）上构建的。

你已经使用过`python manage.py rebuild_index`来移除索引（index）中的任何信息然后再次对documents进行索引（index）。要更新你的索引（index）不需要移除所有对象，你可以使用`python manage.py update_index`。另外一种方法，你可以使用参数`--age=<num_hours>`来更新少量的对象。你可以设置一个定时任务（Cron job）来执行这些操作为了保证你的Solr索引不断进行自动更新。

###创建一个搜索视图（view）
现在，我们要开始创建一个定制视图（view）来允许我们的用户搜索帖子。首先，我们需要一个搜索表单（form）。编辑blog应用下的*forms.py*文件，加入以下代码：

    class SearchForm(forms.Form):
		query = forms.CharField()

我们会使用*query*来让用户引入搜索条件（terms）。编辑blog应用下的*views.py*文件，加入以下代码：

    from .forms import EmailPostForm, CommentForm, SearchForm
	from haystack.query import SearchQuerySet
	
	def post_search(request):
		form = SearchForm()
		if 'query' in request.GET:
			form = SearchForm(request.GET)
			if form.is_valid():
                cd = form.cleaned_data
				results = SearchQuerySet().models(Post).filter(content=cd['query']).load_all()
				# count total results
				total_results = results.count()
				
		return render(request, 'blog/post/search.html',
				{'form': form,
				'cd': cd,
				'results': results,
				'total_results': total_results})
                      
在这个视图（view）中，我们首先实例化了我们刚才创建的*SearchForm*.我们准备使用*GET*方法来提交这个表单（form），这样可以使URL结果中包含查询的参数。假设这个表单（form）已经被提交，我们将在*request.GET*字典中查找*query*参数。当表单（form）被提交后，我们通过提交的*GET*数据来实例化它，然后我们要检查传入的数据是否有效（valid）。如果这个表单是有效（valid）的，我们使用*SearchQuerySet*针对所有被编入索引的并且主要内容中包含给予的查询内容的*Post*对象来执行一次搜索。*load_all()*方法会立刻加载所有在数据库中有关联的*Post*对象。通过这个方法，我们使用数据库对象填充搜索结果，避开了每次迭代存取数据对象的时候对每一个对象都进行存取（译者注：这话不太好翻译，看不懂的话可以看下原文）。最后，我们在*total_results*变量中存储返回结果的总数然后传递本地的变量为上下文(context)来渲染一个模板（template）。

搜索视图（view）已经准备好了。我们还需要创建一个模板（template）来展示表单（form）和用户执行搜索后返回的结果。在*templates/blog/post/*目录下创建一个新的文件命名为*search.html*，添加如下代码：

    {% extends "blog/base.html" %}
	{% block title %}Search{% endblock %}
	{% block content %}
		{% if "query" in request.GET %}
			<h1>Posts containing "{{ cd.query }}"</h1>
			<h3>Found {{ total_results }} result{{ total_results|pluralize }}</h3>
			{% for result in results %}
				{% with post=result.object %}
					<h4><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></h4>           
					{{ post.body|truncatewords:5 }}
				{% endwith %}
				{% empty %}
           			<p>There are no results for your query.</p>
			{% endfor %}	
			<p><a href="{% url "blog:post_search" %}">Search again</a></p>
		{% else %}
			<h1>Search for posts</h1>
			<form action="." method="get">
				{{ form.as_p }}
				<input type="submit" value="Search">
			</form>
		{% endif %}
	{% endblock %}

就像在搜索视图（view）中，我们做了区分如果这个表单（form）是基于*query*参数存在的情况下提交。在这个post提交前，我们展示了这个表单和一个提交按钮。当这个post被提交，我们就展示查询的操作结果，包含返回结果的总数和结果列。每一个结果都是Solr和Haystack封装处理后返回的document。我们需要使用*result.object*来存取真实的有关联的*Post*对象在这个结果中。

最后，编辑blog应用下的*urls.py*文件，添加以下代码：

    url(r'^search/$', views.post_search, name='post_search'),
    
现在，在浏览器中打开 http://127.0.0.1:8000/blog/search/。你会看到如下图所示的搜索表单（form）：

![django-3-14](http://ohqrvqrlb.bkt.clouddn.com/django-3-14.png)

现在，输入一个查询条件然后点击**Search**按钮。你会看到查询搜索的结果，如下图所示：

![django-3-15](http://ohqrvqrlb.bkt.clouddn.com/django-3-15.png)

如今，在你的项目中你已经构建了一个强大的搜索引擎，但这仅仅只是开始，还有更多丰富的功能可以通过*Solr*和Haystack做到。Haystack包含视图（views），表单（forms）以及更多高级功能可以提供给搜索引擎。你可以在 http://django-haystack.readthedocs.org/en/latest/ 页面上获取更多Haystack的信息。

Solr搜索引擎可以通过定制你的架构（schema）来适配各种需求。你可以进行combine analyzers，tokenizers，以及对过滤器（filters）进行标记可以在索引或搜索的时候执行为你的站点内容提供更加精确的搜索。你可以在 https://wiki.apache.org/solr/AnalyzersTokenizersTokenFilters 页面获取到所有的信息。

###总结
在这一章中，你学习了如何创建定制的Django模板标签（template tags）和过滤器（filters）提供给模板（template）一些定制的功能。你还创建了一个站点地图（sitemap）给搜索引擎来爬取你的站点以及一个RSS给用户来订阅。你还通过在项目中集成Slor和Haystack为blog应用构建了一个搜索引擎。

在下一章中，你将会学习到如果构建一个社交网站，通过使用Django认证框架，创建定制的用户概况，以及社交认证。

###译者总结
终于将第三章也翻译完成了，隔了一个星期没翻译，相对于前两章，发现这章的翻译速度又加快了那么一点点，两天内完成翻译。本章中创建自己的模板标签和过滤器个人认为非常实用，我已经打算这段时间将手头上上线的几个项目都使用本章中提供的方法进行部分重构。本章最后部分的搜索引擎我倒是用不到，看官们可以也可以选择不看，毕竟书中提供的版本太老了。。。。。。前三章都是围绕博客应用展开（为什么大家都喜欢用博客应用做初始教程- -|||)，下一章开始将开启新的项目应用，我们下周或下下周或下个月继续- -|||


