## Django REST framework 入门指南

### REST 框架基本概述 （QuickStart）

#### 项目配置

新建一个django项目，可以使用 `django-admin startproject` 或pycharm直接创建django项目

> 推荐新建虚拟python环境（以防项目污染主机使用的python环境），命令行使用
>
> ```python
> python3 -m venv env
> source env/bin/activate
> ```
>
> 来创建并激活；
>
> 使用pycharm创建项目时，可以使用conda创建虚拟环境。
>
> ![image-20210918193036763](C:\Users\86152\AppData\Roaming\Typora\typora-user-images\image-20210918193036763.png)

初始化数据库，使用命令行 `python manage.py migrate`

创建一个默认管理员账户，在django中，用户系统框架是默认帮你搭建好的：

```python
python manage.py createsuperuser --email admin@example.com --username admin
```

然后输入一个你的管理员密码即可，这个账户还需经过授权，之后我们会来做。



创建一个基本的app，在django中，每一个app实现一个特定的功能，这类似于Spring的Service，使用命令`django-admin startapp quickstart`创建一个名为quickstart的app

完成后项目的基本结构如下：

![image-20210918202132600](C:\Users\86152\AppData\Roaming\Typora\typora-user-images\image-20210918202132600.png)

#### Serializers

这对应着持久层，也就是包装数据模型的部分（dto）。

在`quickstart`app下新建一个名为`serializers.py`的代码文件，

由于我们还没有创建数据模型（对应数据库），这里使用django默认配置好的数据模型，其包括`User`和`Group`两个类，使用

```python
from django.contrib.auth.models import User, Group
```

导入两个数据模型类，使用

```python
from rest_framework import serializers
```

导入django REST framework的对应包。

对于每个数据模型类，创建一个对应的Serializer类，命名为`XXXXSerializer`，如对User类，创建`UserSerializer`类，其继承自`serializers.HyperlinkedModelSerializer`类：

```python
class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']
```

其中`HyperlinkedModelSerializer`表示我们采用**实体间超链接**的方法来处理实体间的关系，而不是使用主键的方法，所以serializer继承自这个类。

注意到Serializer类包含一个内部类：`Meta`，其用来存储模型的信息，`model`表示对应的数据模型类名称；`fields`是一个列表或元组，其定义要包装的属性的名称。

将Group也打包为Serializer，完整代码如下：

```python
from django.contrib.auth.models import User, Group
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```



#### Views

这对应着服务层，也就是实现功能的地方。

在创建app时，django会自动为每个app添加`views.py`文件，打开这个文件，然后开始撰写服务层：

导入REST包以及之前定义好的Serializer类：

```python
from django.contrib.auth.models import User, Group
from rest_framework import viewsets
from rest_framework import permissions
from tutorial.quickstart.serializers import UserSerializer, GroupSerializer
```

创建一系列ViewSet类，命名为`XXXXViewSet`，并继承`viewsets.ModelViewSet`类

```python
class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
    permission_classes = [permissions.IsAuthenticated]
```

每个ViewSet类 对应着 对一个数据流（Serializer类）的所有处理的方法。自然你可以对每个分开的方法创建对应的View类，但对同一类型的数据模型，采用同一个ViewSet会使得代码可读性增加。



#### URLs

这对应控制层，但在django中，控制层（路由及返回包控制）都是在项目根目录的`urls.py`中实现的，无需为每个数据模型创建控制层（不过如果数据规模过大，每个app内可以创建自己的子`urls.py`文件，从而与根目录的URL形成一个树状结构，方便检索）

在`urls.py`中导入view类和REST包：

```python
from django.urls import include, path # 这是默认写好的
from rest_framework import routers
from tutorial.quickstart import views
```

REST框架的router类提供了非常方便的路由生成和链接的方法，使用

```python
router = routers.DefaultRouter()
```

新建一个默认路由对象，然后使用其`.register()`方法，即可注册路由：

```python
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)
```

第一个参数允许正则表达式，其说明url的path名称，如"users" 表明url注册为"http://localhost/users"

第二个参数传入你的ViewSet或View类，表明注册的服务层

然后在urlpatterns映射列表中，我们只需传入`router.urls`即可：

```python
urlpatterns = [
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

注意，使不使用router类对所有其他的View层类没有影响，你依然可以在urlpatterns下加入手动定义的path和include相应的服务层类与函数，只不过使用router是自动生成URLs罢了。

这里第二行我们加入了一个REST framework内置的View，名称是`'rest_framework.urls'`，对应的url是`'api-auth/'`，这是REST framework写好的登陆鉴权页面，很适合在使用浏览器测试api而非自动化测试时快速生成一个登陆页面，所以我们加上了这个路由，以方便接下来测试。

完事之后，我们要把模型名称`'rest_framework'`加到`settings.py`中的`INSTALLED_APPS`列表中，以便其能正常运行：

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

接下来，你就可以登录http://localhost/users/，django会为你准备好一个后台的网页，里面你可以进行api测试，注意要先在右上角登录之前创建的超级管理员用户。

![image-20210919104101832](C:\Users\86152\AppData\Roaming\Typora\typora-user-images\image-20210919104101832.png)

如果你使用postman或apifox等工具，测试接口前，也要先POST一个登陆请求。

## REST 教程详述 

### 1 模型与序列化 Model&Serialization

这部分教程，我们要完成一个高亮显示代码的网页的api构建，并学习 django REST 有关序列化的知识。

新建一个django app （你也可以直接新建项目，再创建app），名为“snippets”：

```shell
python manage.py startapp snippets
```

并将新的app的命名空间，加入到`settings.py`中的`INSTALLED_APPS`列表中，即：

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
]
```

安装包pygments，这是一个用来生成高亮的html格式代码的插件：

```shell
pip install pygments # 如果是conda环境，使用conda install pygments
```

准备工作就完成了。

#### 建立数据模型

为了实现代码高亮功能，首先要建立一个模型，存储与高亮的代码有关的数据。

进入`snippets/models.py`文件，这里用来定义该app所需的数据模型。

```python
# 导入django的models类，用来创建模型
from django.db import models
# 导入pygments的相关类，用来生成高亮字符串
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

# 这是用来初始化 pygments 的静态常量
LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


# 建立模型类，继承Model类
class Snippet(models.Model):
    # 为模型添加属性
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100) # 这里使用choice参数，将之前的定义的pygments接受的常量列表放入进去
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    # 定义元数据，这不是必须的，通常用来标识一些默认选项
    class Meta:
        ordering = ['created']  # 比如这个表示默认排序方式为“按创建时间排序”

```

你的数据模型类应该继承自django的`models.Model`模板类，然后使用`models`域下的各种函数，就可以轻松创建各种类型的数据，做一个简单的总结：

##### 字段类型

| 名称            | 数据类型             | 常用参数（所有字段都有的默认参数[见下](#1)）                 | 备注                                                         |
| --------------- | -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| AutoField       | 整数，自增序列       |                                                              | 所有*没有指定主键*的模型，主键都是该类型                     |
| BigAutoField    | 64位正整数，自增序列 |                                                              | 保证适合 `1` 到 `9223372036854775807` 的数字                 |
| BigIntegerField | 64位整数             |                                                              | 保证适合从 `-9223372036854775808` 到 `9223372036854775807` 的数字 |
| BooleanField    | 布尔类型             |                                                              |                                                              |
| CharField       | 字符串类型           | max_length -> 最大长度<br>db_collation -> 定义该字段的数据库字符序名称 | 不适用于大段的文本                                           |
| DateField       | 日期类型             | auto_now -> 设置为true时，**每次**保存对象时自动自动将字段设置为当前时间<br>auto_now_add -> 设置为true时，**第一次**保存对象时自动自动将字段设置为当前时间；<br>以上选项会与default选项相排斥 | 在 Python 中用一个 `datetime.date` 实例表示其值              |
| DateTimeField   | 日期时间类型         | 与DateField相同                                              | 在 Python 中用一个 `datetime.datetime` 实例表示其值          |
| DecimalField    | 十进制小数           | **必须**：decimal_places -> 小数位数<br>**必须**：max_digits  -> 总位数，必须大于小数位数 | 在 Python 中用一个 `Decimal` 实例来表示其值                  |
| DurationField   | 时间段类型           |                                                              | 在 Python 中用 [`timedelta`](https://docs.python.org/3/library/datetime.html#datetime.timedelta) 建模。<br>当在 PostgreSQL 上使用时，使用的数据类型是 `interval`<br>在 Oracle 上使用的数据类型是 `INTERVAL DAY(9) TO SECOND(6)`<br>否则使用微秒的 `bigint`。 |
| EmailField      | 邮件类型             | max_length -> 最大长度（默认值为254）                        | 是一个CharField，不过会自动验证字段值是否为有效邮件地址      |
| FileField       | 文件上传字段         | 这部分内容很长，参考[FileField](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#filefield) |                                                              |
| FilePathField   | 文件路径字段         | **必须**：path  -> 文件系统的绝对路径                        | 是一个CharField，用来存储文件路径，参考[FilePathField](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#filepathfield) |
| FloatField      | 浮点数类型           |                                                              | 使用 Python 的 `float` 类型表示其值                          |
| ImageField      | 图像上传字段         | 使用该字段需要`Pillow`库，参考[ImageField](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#imagefield) | 继承自FileField，不过会验证图像有效性                        |
| IntegerField    | 整数类型             |                                                              | 保证适合`-2147483648` 到 `2147483647` 的值                   |
| JSONField       | json类型             | 参考[JSONField](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#jsonfield) | 在 Python 中，数据以其 Python 数据类型表示：字典、列表、字符串、数字、布尔值和 None。 |
| TextField       | 大段文本类型         | max_length -> 最大长度<br/>db_collation -> 定义该字段的数据库字符序名称 | 可以用来存储大段文本信息                                     |
| TimeField       | 时间类型             | 与DateField相同                                              | 在 Python 中用 datetime.time 实例表示                        |
| URLField        | url类型              | max_length -> 最大长度（默认值为200）                        | 是一个CharField，不过会自动验证url的有效性                   |

##### <span id=1>字段默认选项</span>

- [`null`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.null)

如果设置为 `True`，当该字段为空时，Django 会将数据库中该字段设置为 `NULL`。默认为 `False` 。

- [`default`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#default)

设置字段的默认值。

- [`blank`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.blank)

如果设置为 `True`，该字段允许为空。默认为 `False`。

注意该选项与 `null` 不同， [`null`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.null) 选项仅仅是数据库层面的设置，而 [`blank`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.blank) 是涉及表单验证方面。如果一个字段设置为 [`blank=True`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.blank) ，在进行表单验证时，接收的数据该字段值允许为空，而设置为 [`blank=False`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.blank) 时，不允许为空。

- [`choices`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.choices)（详情点击链接查看）

一系列二元组，用作此字段的选项。如果提供了二元组，默认表单小部件是一个选择框，而不是标准文本字段，并将限制给出的选项。

- [`primary_key`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#primary-key)（详情点击链接查看）

如果设置为 `True` ，将该字段设置为该模型的主键。

如果你没有为模型中的任何字段指定 `primary_key=True`，Django 会自动添加一个字段来保存主键。

- [`unique`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#unique)（详情点击链接查看）

如果设置为 `True`，这个字段必须在整个表中保持值唯一。

这是在数据库级别和模型验证中强制执行的。如果你试图保存一个在 [`unique`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.Field.unique) 字段中存在重复值的模型，模型的 [`save()`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/instances/#django.db.models.Model.save) 方法将引发 [`django.db.IntegrityError`](https://docs.djangoproject.com/zh-hans/3.2/ref/exceptions/#django.db.IntegrityError)。

除了 [`ManyToManyField`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.ManyToManyField) 和 [`OneToOneField`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.OneToOneField) 之外，该选项对所有字段类型有效。

##### 数据模型的关联关系

当你的数据模型直接有关联关系时，需要以下额外定义保证不同数据库之间的数据更新正确。

- 多对一

  在“多”方的数据模型类中，增加“一”方类型的对象，使用`models.ForeignKey`：

  ````python
  class Car(models.Model): # 一个manufacturer可以生产多个car
      manufacturer = models.ForeignKey(
          Manufacturer,
          on_delete=models.CASCADE, # on_delete 参数说明了删除时的级联关系
      )
  ````

> 要创建一个递归关系——即一个与自己有多对一关系的对象，
>
> 使用 `models.ForeignKey('self', on_delete=models.CASCADE)`，
>
> 这对于一对一的关系也适用。

- 多对多

  在任意一个“多”方的数据模型中，使用`models.ManyToManyField`：

  ```python
  class Author(models.Model):
      # ...
      
  class Article(models.Model): # 一篇article可有多个author，反之亦然
      # ...
      authors = models.ManyToManyField(Author)
  ```

  不要在两个模型中都添加ManyToMany字段，并且一般来讲，应该把 `ManyToManyField`
  实例放到**需要在表单中被编辑的对象**中。

- 带有属性的多对多

  在定义多对多字段时，添加参数 `through=` ，值为一个中间模型的名称，其包含该多对多关系的属性

  ```python
  class Person(models.Model): 
      name = models.CharField(max_length=128)
  
      def __str__(self):
          return self.name
  
  class Group(models.Model):
      name = models.CharField(max_length=128)
      members = models.ManyToManyField(Person, through='Membership')
  
      def __str__(self):
          return self.name
  
  class Membership(models.Model): # 多个person加入多个Group，Membership记录额外信息
      person = models.ForeignKey(Person, on_delete=models.CASCADE)
      group = models.ForeignKey(Group, on_delete=models.CASCADE)
      date_joined = models.DateField() # 加入时间
      invite_reason = models.CharField(max_length=64) # 邀请人
  ```

  关于中间模型的限制条件，参考[ManyToManyField](https://docs.djangoproject.com/zh-hans/3.2/topics/db/models/#extra-fields-on-many-to-many-relationships)

- 一对一

  在子级的“一”方数据模型中，使用`models.OneToOneField`引用父级的“一”方：

  ```python
  class Place(models.Model):
      name = models.CharField(max_length=50)
      address = models.CharField(max_length=80)
  
  class Restaurant(models.Model): # 一个restaurant一定是一个place
      place = models.OneToOneField(
          Place,
          on_delete=models.CASCADE,
          primary_key=True, # 这将指定外键place为该模型的主键
      )
      serves_hot_dogs = models.BooleanField(default=False)
      serves_pizza = models.BooleanField(default=False)
  
  ```

  如果指定外键为该模型的主键，那么参数`primary_key=True`必须显示声明，否则django仍然会为该模型创建一个`AutoField`的id主键。

  因此可以在单个模型当中指定多个 [`OneToOneField`](https://docs.djangoproject.com/zh-hans/3.2/ref/models/fields/#django.db.models.OneToOneField) 的字段。

#### 建立序列化类

为了实现Web API的传输，数据类型需要进行 序列化 和 反序列化，将其转化为一种特定的格式（通常是`json`），因此django REST framework提供序列化类，方便序列化工具的建立。

在app`snippets`文件夹下新建立一个`serializers.py`（因为REST framework并不属于django本体，因此创建app时不会为你新建这些代码文件）

##### 生成序列化类的两种方式

1、自行创建类：（优点：可控制性，序列化过程中可以通过代码进行数据处理）

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES

# 定义名称具有关联系的 序列化类
class SnippetSerializer(serializers.Serializer):
    # 需要序列化的数据，手动通过serializers重新定义
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    # style参数只针对 使用django模板渲染到html时 的数据显示格式
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    # create函数在序列化的对象第一次生成的时候调用，因此它应该初始化数据模型类的对象，并将其返回
    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    # update函数在序列化对象更新（serializer.save()）的时候调用，因此应该修改instance的属性值
    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

使用继承`serializers.Serializer`类的方式创建序列化类，需要重载`create()`和`update()`两个函数，并在其中实现对对应数据模型类instance的处理。因此你可以在这部分内容中自行进行数据的处理。

2、使用`ModelSerializers`自动生成（优点：自动定义类型，代码量极少）

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
```

继承`serializers.ModelSerializer`，只需要在内部类`Meta`中传入数据模型类和所需序列化的字段（注意，id是django隐式添加的主键，但在这里要显示声明），即可自动判定类型，生成序列化类。使用这种方法，序列化类的`create()`和`update()`函数都是默认的。

#### 在服务层中使用序列化类

写好了序列化类，接下来我们就可以在服务层（每个app的Views文件）中方便的使用它进行数据操作。

> **【注意】**我们现在使用的依然是传统django的View层语法，暂不涉及到RESTful的特性。

在`snippets/views.py`文件中添加代码：

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer

@csrf_exempt # 该注解标识一个视图可以被跨域访问

# 这个函数用来回应对 数据列表 查找和新建的请求
def snippet_list(request):
    # 如果请求方式为GET，返回所有snippet的列表
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)
    
	# 如果请求方式为POST，接收请求的参数并新建一个snippet
    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
    
@csrf_exempt
# 这个函数用来回应对 单个数据 增删改查的请求（pk代表主键）
def snippet_detail(request, pk):
    # 先判断请求的数据实例是否存在
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

数据操作总结：

- `instance = model.object.get(pk)`通过主关键字获取对象，`model.object.all()`获取所有对象的列表

- `serializer = XXXXSerializer(instance,**option)`创建数据模型对象 对应的 序列化类的对象，如传入的是列表，在参数上加入`many=True`可以创建一个序列化的列表

- `serializer = XXXXSerializer(instance, data)`通过请求数据新建或修改的`serializer `，调用其`.save()`函数即可将数据通过序列化方式存入模型数据库，记得在save之前使用`serializer.is_valid()`校验数据有效性

- 删除对象直接使用`instance.delete()`

#### 测试API

以上都完成之后，我们新建文件`snippets/urls.py`，这是一个子路由文件，只用来处理这一个app的路由信息，在其中我们添加：

```python
from django.urls import path
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
    # <int:pk> 表示path路径中的值会被以int类型作为参数传入到函数的pk参数中去
]
```

由此，在项目配置目录的`tutorial/urls.py`中，我们只需引用：

```python
urlpatterns = [
    ...
    path('', include('snippets.urls')),
]
```

即可完成对`snippets/urls.py`文件中所有路由的应用。

最后，别忘了要使用命令行，使用ORM初始化数据库，否则django将无法连接到对应的数据库。

```
python manage.py makemigrations

python manage.py migrate
```

到这里，高亮显示代码的网页的api就完成啦！接下来你可以使用命令行、rest_framework前端和apifox等第三方工具对其进行测试。

### 2 请求与响应  Requests and Responses



