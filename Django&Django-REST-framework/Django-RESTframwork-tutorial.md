

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

<span id=0>对于每个数据模型类，创建一个对应的Serializer类，命名为`XXXXSerializer`，如对User类，创建`UserSerializer`类，其继承自`serializers.HyperlinkedModelSerializer`类：</span>

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

这部分教程，我们要完成一个高亮显示代码的网页的api构建，并学习 django REST 有关的知识。

### 1 模型与序列化 Model & Serialization

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

- **多对一**

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

- **多对多**

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

- **带有属性的多对多**

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

- **一对一**

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

  如果要指定外键为该模型的主键，那么参数`primary_key=True`必须显示声明，否则django仍然会为该模型创建一个`AutoField`的id主键。

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

最后，别忘了要使用以下命令行，通过ORM初始化数据库，否则django将无法连接到对应的数据库。

```
python manage.py makemigrations

python manage.py migrate
```

到这里，高亮显示代码的网页的api就完成啦！接下来你可以使用命令行、rest_framework前端和apifox等第三方工具对其进行测试。

### 2 请求与响应  Requests & Responses

#### REST api支持

REST 框架下引入了一个新的`Request`类，其继承于python库的`HttpRequest`，但拥有更多功能。

比如在不使用REST framework时，request.POST只能让你处理POST方法的相关数据，

但现在，使用新`Request`类的request.data，你可以处理多种方法（POST、PUT、PATCH）的数据，所以更加的RESTful。

相应的，响应体也会由REST framework的`Response`代替，而不再使用`JSONResponse`等直接返回一种格式的相应。响应体需要渲染成何种格式可以由发起请求的一方（客户端）来决定 。

```python
return Response(data) # Renders to content type as requested by the client.
```

此外，不要再使用数字`status=400`做http返回状态码了，这样会降低代码的可读性，使用`status`的枚举来代替它们，比如`status.HTTP_400_BAD_REQUEST`.

要使用REST framework提供的`Request`和`Response`，需要导入以下命名空间：

```python
from rest_framework import status # 导入http状态码枚举
from rest_framework.decorators import api_view # 导入view请求处理
from rest_framework.response import Response # 导入返回体
```

然后在你的View层函数前，加上修饰词`@api_view([])`，列表里填写你的函数涉及到的请求方法，如`'GET','POST','PUT','DELETE'`等。

通常来说，一个函数对应一个URL的请求，但可以包含若干个不同的请求方法。加入`@api_view`修饰词的目的是告诉REST framework，在这个view函数里，其接受的request请求都必须是符合REST规范的。

#### 修改之前View代码

使用REST api，将`snippets/views.py`的代码完善如下：

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST']) # 这个方法对应的URL请求只接受 GET 和 POST
def snippet_list(request):
	# 如果请求方式为GET，返回所有snippet的列表
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        # 请求方法为GET时，可以不填写status，成功时Response会为我们自动返回200
        return Response(serializer.data)
    
	# 如果请求方式为POST，接收请求的参数并新建一个snippet
    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            # 使用Response返回响应体，并用枚举代替纯数字的http状态码
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE']) # 这个方法对应的URL请求只接受 GET、POST和DELETE
def snippet_detail(request, pk):
    # 先判断请求的数据实例是否存在
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

可以看到，使用`@api_view` 和`Response`，我们基本摆脱了使用某种特定格式（json）进行序列化。现在请求可以使用任何格式（只要RESTful支持），我们的代码就会返回相应格式的数据给客户端。我们也无需在响应的时候区分`HttpResponse`和`JSONResonse`等等，这样极大地提升了代码的健壮性。

#### 增加URL格式后缀支持

为了让“请求格式自由”在我们的api上显示的体现出来，不妨让使用不同格式的请求，在访问URL的时候，增加表明自己格式的后缀，比如：

```python
http http://127.0.0.1:8000/snippets.json  # .json表示这是一个json请求
http http://127.0.0.1:8000/snippets.api   # .api表示这是一个api可浏览式请求
```

这样前端在请求的时候，就无需再额外指定可接受的格式类型，只要在URL末端加上这么一个简单的后缀即可。

要添加这样的后缀支持，只需在View定义函数的时候，在参数后面加上`format=None`，比如我们将之前的两个函数定义改为：

```python
def snippet_list(request, format=None):
	
def snippet_detail(request, pk, format=None):
```

然后去到`snippets/urls.py`路由文件，将`urlpatterns`列表使用REST framework提供的`format_suffix_patterns`方法进行打包，即可大功告成：

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

现在进行测试，可以看到不论采取何种请求格式，都可以获得相同的结果：

```python
# 注：原文作者使用的是httpie

# POST using form data 
http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
  "id": 3,
  "title": "",
  "code": "print(123)",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# POST using JSON
http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

{
    "id": 4,
    "title": "",
    "code": "print(456)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```



### 3 基于类的视图实现 Class-based Views

这一部分我，我们介绍如何使用“基于类的视图”（Views）来代替之前写的“基于函数的视图”。这将有利于我们将部分有重复功能的方法复用，使得代码更加精炼。

#### 创建视图类

修改之前的`snippets/views.py`文件，这一次，我们要将每个处理请求URL的函数都改为类：

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer

# 使用APIView代替之前的api_view，这将是我们继承的基类
from rest_framework.views import APIView 
from rest_framework.response import Response
from rest_framework import status

# 这是django的一个异常类，用来抛出异常
from django.http import Http404

# 用来处理获取snippets列表的URL请求，记得继承APIView类
class SnippetList(APIView):
    
    # 类的函数名标识了对应处理的请求类型
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

# 用来处理单个snippets增删改查的URL请求，记得继承APIView类
class SnippetDetail(APIView):
    
    # 这是一个私有函数，使用时记得要加上前缀`self.`
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

我们将不同的请求方法分用不同的函数处理，这样代码就分的更开了，以便后续维护。

修改为类视图后，需要在路由文件`snippets/urls.py`中对视图的引用做同样的修改，方法如下：

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    # 调用视图类的`as_view()`函数即可
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

#### 使用混入模板(mixins)

对大多数的数据模型，**增删改查**都是最常用的数据操作。使用视图类的好处之一就是可以将这四种基本操作模板化。在REST framework中提供了一个*混入*（mixin）类，用来实现这些模板操作。

因此，我们可以进一步简化之前的`views.py`代码，套用mixin类的模板来实现：

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
# 导入mixins和generic模板类包
from rest_framework import mixins
from rest_framework import generics
"""
注意：使用模板包时，generics类会为我们自动生成APIView，所以无需导入APIView包
"""

# 使用mixins模板创建视图类，根据所需要的功能继承其相应的Mixin类
# `generics.GenericAPIView`用来生成APIView视图
class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    # 初始化Mixin类数据成员，包括所有数据模型对象(queryset)和对应的序列化类对象
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    # 在请求处理函数中，使用Self直接调用对应Mixin类继承而来的函数
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

# `SnippetDetail`类的模板实现基本同理  
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)   
```

在`mixins`模板类中，提供了一系列基本数据操作的子类，但我们需要实现的视图类包含其中的哪些操作时，我们就可以直接继承该子类，然后使用REST framework为我们定义好的方法，简单的总结如下：

> python基础知识普及：
>
> `args` 是 arguments 的缩写，表示位置参数；`kwargs` 是 keyword arguments 的缩写，表示关键字参数（一个字典）；前面加入`*`和`**`表示这些参数**按照传入顺序**形成一个列表或字典，供函数使用。

| mixins类                    | 功能                                 | 返回函数                                  |
| --------------------------- | ------------------------------------ | ----------------------------------------- |
| `mixins.CreateModelMixin`   | 创建数据模型（增）                   | `self.create(request, *args, **kwargs)`   |
| `mixins.RetrieveModelMixin` | 查询单个数据（查）                   | `self.retrieve(request, *args, **kwargs)` |
| `minxins.ListModelMixin`    | 查询`queryset`中的所有数据组成的列表 | `self.list(request, *args, **kwargs)`     |
| `mixins.UpdateModelMixin`   | 更新单个数据模型（改）               | `self.update(request, *args, **kwargs)`   |
| `mixins.DestroyModelMixin`  | 删除单个数据模型（删）               | `self.destroy(request, *args, **kwargs)`  |

同时，不要忘了继承`generics.GenericAPIView`类，它相当于模板类APIView实现。

#### 使用通用视图类模板(generic class-based views)

如果你的视图类中*只涉及*以上增删改查的操作，而不涉及其他的自定义操作了，那么在`generics`中也给你提供了对应的子类，你只要传入你想要的操作对应的名称就行了！

最终，我们的视图类可以简化成这种样子：

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics

# 这表示，我们让generics帮我们实现`List`和`Create`操作
class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

# 同样，我们让generics帮我们实现`Retrieve`、`Update`和`Destroy`操作
class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

这属实是不能再简化了（只有9行了啊，各位！）

不过一旦你的视图函数中有对于数据有任何的特殊化操作，这种方法就不适用了，这时还是需要自行撰写视图类函数（或者，去重载一个你自己的APIView子类，详见[Creating custom mixins](https://www.django-rest-framework.org/api-guide/generic-views/#creating-custom-mixins)）。

### 4 授权与鉴权 Authentication & Permissions

到目前为止，我们的API还没有任何授权限制。也就是说，任何人都可以创建、修改和删除“代码高亮帖子”。为了安全性考虑，我们应该加入以下限制：

- 每个代码块必须标识其创作者（上传者）
- 只有经过授权的用户才能创建新的代码块
- 只有每个代码块的创作者本人才能更新或删除它
- 未经授权的用户可以用于所有代码块的*只读*权限

#### 修改模型

首先，我们需要在代码块`Snippet`类中标识每个代码块的创作者。因此我们要去修改这个model里面定义的内容。又因为创作者是一个用户，其也是一个数据模型，所以我们需要弄清楚这两个实体之间的关系。

显然，一个用户可以创作多个代码块，而每个代码块只有一个作者，因此我们将使用**多对一**关系：

修改`models.py`，加入如下内容：

```python
# 新增多对一外键
owner = models.ForeignKey('auth.User', /
                          related_name='snippets', on_delete=models.CASCADE)
# 哦对了，还有新增一个文本字段，用来存储高亮处理后的代码
highlighted = models.TextField()
```

既然增加了存储高亮代码块的部分，那么我们要在数据模型修改的时候，将高亮后的代码存进这个字段，所以修改`.save()`函数如下：

```python
...
# 先额外导入这些包
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight

def save(self, *args, **kwargs):
    """
    使用`pygments` 库将code字段转化为高亮代码字段，存储到highlighted字段
    """
    lexer = get_lexer_by_name(self.language)
    linenos = 'table' if self.linenos else False
    options = {'title': self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

完成对数据模型的改动后，我们使用命令行刷新数据库，并再额外创建几个用户用来测试：

```python
# 先drop，再创建数据库（windows下请使用windows命令 或 在IDE中drop数据库）
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate

# 创建另一个超级用户
python manage.py createsuperuser
```

#### 创建用户序列化类

既然我们的数据模型现在需要用户的模型外键，那么我们也需要提供一些用户数据操作的API，所以需要建立一个新的序列化类代表用户（这部分内容和我们在[Qucikstart](#0)中做的相似）。在`serializers.py`我们增加：

```python
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ['id', 'username', 'snippets']
```

注意我们使用了`serializers.PrimaryKeyRelatedField()`，这是因为snippets是关系中“多”的一方，它是不会被`ModelSerializer`自动处理的，因此需要我们手动处理。（后面我们会说到如何解决这个问题，详见[这里](#5)）

同时更新`views.py`，增加处理用户信息和用户列表的视图层：

```python
from django.contrib.auth.models import User
from snippets.serializers import UserSerializer

class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

将对User的访问加入路由：

```python
# 加入path到文件`snippets/urls.py`的urlpatterns中：
path('users/', views.UserList.as_view()),
path('users/<int:pk>/', views.UserDetail.as_view()),
```

#### 传递用户数据

到现在为止，我们在创建”代码块“实例的时候，还没有包含任何有关用户的数据。接下来我们要做的就是在创建的时候，把当前登录用户的数据，通过视图层和序列化层写进我们修改过的数据模型里。

在视图层，我们增加一个名为`.perform_create()`的函数，重载这个函数允许我们插手实例创建的步骤，从而将请求中的数据（用户）写进序列化对象中：

```python
# 在SnippetList类中添加如下函数
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```

现在，我们的序列化对象在调用`create()`函数时，一个新的名为`owner`的字段就会被传入，其他请求中的有效数据不受影响。

于是乎，我们需要继续修改序列化层，打开`serializers.py`，修改定义如下：

```python
class SnippetSerializer(serializers.ModelSerializer):
	# 从视图层的owner字段中，提取用户名（只读）存储到数据库的owner字段中
	owner = serializers.ReadOnlyField(source='owner.username')
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style', 'owner']
```

别忘了在Meta子类中，也加入'owner'这个字段。

在这里我们使用了`ReadOnlyField`的类型，这个类型的字段永远是只读的，也就是说在仅在序列化（创建、读数据库）到时候使用，而反序列化（写入、修改数据库的时候）不允许使用，从而保证了任何用户无权篡改作者信息。对于这里的用户名类型，也可以用`CharField(read_only=True)`代替只读类型。

#### 增加访问权限控制

现在，所有的代码块对象都会存储一个有关联的用户对象了，接下来的步骤就是保证“只有授权用户才可以创建、更新和删除对应的代码块”。

在REST framework中，有一系列授权模块，用来限制用户有什么权限来访问特定的视图。

在视图层`views.py`导入授权模块，名为`permissions`：

```python
from rest_framework import permissions
```

然后，添加以下授权语句到每个视图类，包括`SnippetList` 和 `SnippetDetail`:

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

`IsAuthenticatedOrReadOnly`值表示有授权的用户（已登录）才可以有完全访问权限，否则均为只读权限。

#### 增加登陆模块

既然现在API需要授权才能访问，所以我们还需要一个登陆接口，能让授权用户得以登陆。

你大可以自己实现自己所需要的登陆接口。不过REST framework提供了一个简易的，用于后台测试的登陆接口，你只要做的是在**项目配置文件夹**的路由文件`urls.py`中加入REST framework 提供的登陆视图：

```python
urlpatterns += [
    # url的名称`api-auth/`可以自定义
    path('api-auth/', include('rest_framework.urls')),
]
```

现在访问本地浏览器，你可以在右上角看到一个登陆按钮，那么调用接口就成功了。

#### 对象级别鉴权

到目前为止，还有一个问题：现在已登录的用户即可访问所有代码块对象，但我们要求只有**创建者本人**，才可以更新或删除自己的代码块，这就涉及到单个对象级别的鉴权。

为了实现这一功能，我们在`snippet/`文件夹下新建一个`permissions.py`代码文件：

```python
from rest_framework import permissions

# 新建授权类，继承授权基类
class IsOwnerOrReadOnly(permissions.BasePermission):
    
    # 自定义一个授权函数，其只允许创建者本人更新对象
    def has_object_permission(self, request, view, obj):
        # 读取代码块无需鉴权，所以我们直接允许以下类型的请求，
        # permissions.SAFE_METHODS，即： GET, HEAD 或 OPTIONS
        if request.method in permissions.SAFE_METHODS:
            return True

        # 除去以上类型的“安全请求”，返回对象的作者与请求的用户的判断 以确定权限
        return obj.owner == request.user
```

这样我们的自定义鉴权类就实现了。这里判定函数的参数分别为

`request`：传入的请求类

`view`：使用此鉴权的视图类

`obj`：访问的序列化模型类

现在我们可以在视图类中加入我们的自定义鉴权：

```python
# from snippets.permissions import IsOwnerOrReadOnly
# 注意我们定义的文件名`permissions`和库中的类文件`permissions`同名，
# 因此建议使用时采取完全引用或创建别名

permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                      snippets.permissions.IsOwnerOrReadOnly]
```

现在登陆浏览器，你会发现，只有在登陆了相应作者的账户时，DELETE和PUT选项才会在前端后台展示出来。

#### 用户鉴权方式

在这个例子中，我们使用的是django REST framework的默认鉴权模式，即

`SessionAuthentication`和 `BasicAuthentication`。这两种模式都不需要单独创建*授权类 [authentication classes](https://www.django-rest-framework.org/api-guide/authentication/)*，但会存在一些缺点：

比如使用`SessionAuthentication`，会让权限判断全部在后端服务器上，那么当用户很多的时候，重复鉴权就会拖慢后端服务器的性能。因此现在多采用`TokenAuthentication`等方式，在[附录部分](#10)我们会解释如何使用其他的鉴权方式。

### 5 使用超链接API处理实体关系 Relationships & Hyperlinked APIs

<span id=5>到现在为止，我们的API依然采用主键的方式。现在我们需要是增强我们API的内聚性。因此这一部分我们将使用超链接来处理实体间的关系。</span>

#### 创建API_root

如同我们对待'snippets' 和 'users'模型类一样，现在我们也为API单独创建一个根入口：使用基于函数的形式，在`snippets/views.py`中创建一个名为api_root的函数：

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

创建这个“根入口”的目的，是为了让任意请求可以方便的获取**所有**数据模型类的**超链接**，生成这个超链接的方法，即使用REST framework提供的`reverse()`函数，它可以返回一个数据模型了所有符合要求的URL，即我们在`snippets/urls.py`起过名字的相应URL。

将创建的api根入口加到`urlpattens`中：

```python
path('', views.api_root),
```

#### 创建高亮代码View视图类

为了实现用户直接在浏览器观看高亮代码的功能，我们还需要一个返回先前创建的高亮文本字段的视图类，给它起名为`SnippetHighlight`：

```python
# 增加导入该渲染包
from rest_framework import renderers

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = [renderers.StaticHTMLRenderer]

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

在这个视图类中，我们没有使用JSON返回格式，而是使用了静态HTML的格式（先进行预渲染），因此使用了` renderer_classes = [renderers.StaticHTMLRenderer]`，并且我们重载了get函数，只返回snippet对象的高亮文本字段。

将创建的高亮视图类加入到`urlpatterns`中：

```python
path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view()),
```

#### 增加超链接

超链接实体关系，简言之，就是通过查询URL，找到*关系的一方*所引用的*关系的另一方*的序列化类，从而获取实体间关系信息。

为了使用超链接关系，我们需要修改序列化类的定义，将之前使用的`serializers.`修改为`HyperlinkedModelSerializer`

使用`HyperlinkedModelSerializer`，它将**不会**为序列化对象生成名为`id`的主键。取而代之的是一个名为`url`的字段，其存储*关系的外键*所对应的视图类/函数的URL。

将`snippets/serializers.py`的代码修改为：

```python
# 使用serializers.HyperlinkedModelSerializer替代serializers.serializers
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    # 使用HyperlinkedIdentityField代替PrimaryKeyRelatedField
    # 加入一对一的highlight字段，序列化高亮代码，注意这里将格式设置为'html'
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ['url', 'id', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style']


class UserSerializer(serializers.HyperlinkedModelSerializer):
    # 相同的，如果是一对多关系，将参数many设置为True
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ['url', 'id', 'username', 'snippets']
```

#### 配置URL路径命名

在之前加入超链接时，我们发现其传入的参数为`view_name = '【url-pattens】'`

这里传入的即为视图层的URL路径名，为了让这个命名正确引用，必须在`url.py`文件中正确命名。

对应的命名规则如下：

- API 根路径命名：例如 `'user-list'` 和 `'snippet-list'`（这是在`api_root()`的视图根函数定义的）
- 单个序列化类外键的路径命名：例如 `'snippet-highlight'`和`'snippet-detail'`（分别是在`SnippetSerializer`和`UserSerializer`序列化类的`HyperlinkedRelatedField`中定义的）
- 单个序列化类默认主键的路径命名：会被自动以`'【model_name】-detail'`，命名，例如 `'snippet-detail'` 和 `'user-detail'`

因此，修改`urlpatterns`如下：

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    path('', views.api_root),
    # snippets根目录
    path('snippets/',
        views.SnippetList.as_view(),
        name='snippet-list'),
    # 单个snippet具体页面
    path('snippets/<int:pk>/',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    # 但snippet对应高亮页面
    path('snippets/<int:pk>/highlight/',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    # 用户根目录
    path('users/',
        views.UserList.as_view(),
        name='user-list'),
    # 单个用户具体页面
    path('users/<int:pk>/',
        views.UserDetail.as_view(),
        name='user-detail')
])
```

#### 增加分页

在项目配置文件的`settings.py`中加入如下配置，即可让REST framework在默认返回列表类型的对象时，使用分页方式返回：

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

有关自定义分页格式，参见[附录](#11)

最终我们的api返回格式如下：



### 6 视图集和路由器 ViewSets & Routers

虽然我们已经学会了用最简洁的方式，创建一系列视图类，但似乎在视图很多很复杂的方法，一堆视图类仍然不是最好的方案。在REST framework中，提供了一个抽象类`ViewSet`，它允许开发者在构建URL前构建视图，把配置URL交给对应的`Routers`完成。只要实例化一个视图集类对象，即可创建一系列通用的视图类集合。

ViewSet类在使用上几乎和View一致，但可用的操作从`get`,`put`之类的变为了`retrieve`和`update`。

#### 使用视图集更新视图类

回到`views.py`，现在我们来用`ViewSet`替代之前写的所有视图类。

我们先将有关用户的两个视图：`UserList` 和 `UserDetail`合并到一个视图集，起名为`UserViewSet`.

```python
from rest_framework import viewsets

# `viewsets.ReadOnlyModelViewSet` 会默认将所有可读的方法都设置为只读
class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    该viewset会自动提供`list`和`retrieve`操作，无需你撰写代码
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

按照这个模式，我们再把  `SnippetList`, `SnippetDetail` 和 `SnippetHighlight` 修改为一个视图集类：

```python
class SnippetViewSet(viewsets.ModelViewSet):
    """
    该viewset 会自动提供`list`, `create`, `retrieve`,
    `update` 和`destroy` 操作.

    我们自定义的 `highlight`和`perform_create` 操作，需要额外实现.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

注意到我们仍然使用了`@action`修饰词，这是为了创建不属于`ModelViewSet`定义的基本操作（增删改查和列表查）的操作函数，它的参数如下：

| 参数             | 含义                           | 备注                                                         |
| ---------------- | ------------------------------ | ------------------------------------------------------------ |
| detail           | 函数是否采用详细定义           | True或False，决定生成URL的格式，具体见下表                   |
| method           | 函数接受的HTTP请求方式         | 若不填，默认值为GET<br/>若填入其他请求方式，请传入列表和字符串，如method=['post'] |
| url_path         | 函数对应的URL路径名            | 若不填，默认值参考下表生成                                   |
| url_name         | 函数对应的URL配置路径参数名    | 若不填，默认值参考下表生成                                   |
| renderer_classes | 函数的返回包采用何种渲染器渲染 | 应传入列表                                                   |

自动路由产生的URL格式说明：

- {basename}：视图集类的前缀名，即`XXXXXViewSet`中的`XXXX`（忽略首字母大写）

- {prefix}：前缀，即该视图集下已有相应请求方法时，将其url作为前缀，否则前缀为{basename}

- {url_path}：一个其他的的url连接

- {lookup}：列表数据的主键值，如`<int:pk>`

| 生成的URL格式                 | HTTP 请求方法                              | Action操作类型或相应参数                                     | 生成的URL Name        |
| ----------------------------- | ------------------------------------------ | ------------------------------------------------------------ | --------------------- |
| {prefix}/                     | GET<br>POST                                | list<br/>create 【默认生成】                                 | {basename}-list       |
| {prefix}/{url_path}/          | GET, 或其他由 `methods` 参数指定的请求方法 | `@action(detail=False)` 将detail参数设置为False              | {basename}-{url_name} |
| {prefix}/{lookup}/            | GET<br>PUT<br>PATCH<br>DELETE              | retrieve<br/>update<br/>partial_update<br/>destroy【默认生成】 | {basename}-detail     |
| {prefix}/{lookup}/{url_path}/ | GET, 或其他由 `methods` 参数指定的请求方法 | `@action(detail=True)`   将detail参数设置为True              | {basename}-{url_name} |

#### 手动将视图集和URL绑定

> 注意：这不是必须的，后面我们会使用`Router`路由器来简化相应步骤

为了让视图集正确工作（能够按照规则生成对应的URL索引），需要在对应app的是url配置文件里定义不同方法对应的视图集操作索引。

修改 `snippets/urls.py` 如下：

```python
from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

# 为了代码可读性，我们为每个增加索引的api_view对象重新赋值到新的变量
snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})

urlpatterns = format_suffix_patterns([
    path('', api_root),
    path('snippets/', snippet_list, name='snippet-list'),
    path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail')
])
```

增加索引的方法，即在调用对应`ViewSet`的`.as_view()`函数中，将HTTP请求名和ViewSet操作名按照字典格式一一对应传入即可，注意`list`, `create`, `retrieve`,   `update` 和`destroy`都是默认的ViewSet操作名。

但我们还可以使用更简单的方法。

#### 使用路由器

既然在之前的定义视图集的时候，我们已经说明了操作对应的HTTP请求方式（自定义操作默认为GET），那为何还要在url配置文件中再写一遍呢。因此我们可以直接调用REST framework的`Router`路由器类，来为我们自动路由，生成对应的URL配置参数。

因此，最终 `snippets/urls.py` 简化如下：

```python
from django.urls import path, include
# 导入路由器类
from rest_framework.routers import DefaultRouter
from snippets import views

# 实例化一个路由器对象，将我们写好的视图集作为参数传入.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# 只需`include`路由器对象，即无需再写所有的url配置参数.
urlpatterns = [
    path('', include(router.urls)),
]
```

使用`Router`的注意事项：

- 观察到注册视图到路由器的时候，传入的参数是正则字符串，因此一个路由器类对象可以实例化若干个视图集，可通过视图集名称进行正则索引
- `Router`最终生成的路由URL由你在视图集定义的时候所采用的命名和序列化的数据有关
  - 比如你为名为`SnippetsViewSet`的视图集，并传入了一个`snippet`的列表序列化类作为参数，则其增删改查对应的即为`snippets/<int:pk>/`的POST,DELETE,PUT和GET请求对应的URL，而返回列表的操作为`snippets/`的GET操作
  - 我们自定义的操作，如`snippets/<int:pk>/highlight/`，其生成的URL地址与你在`@action`修饰符中的填写的参数和定义的函数名有关（详见上）

虽然使用视图集和路由器类，能**最大程度的抽象API路由和简化代码**；但其不代表这一定是你的最佳选择。有的时候，**单独来写视图、URL索引能增加代码可扩展性，后期维护更加方便**。

因此，权衡利弊，选择权在你自己的手里。

## 附录

这些是你在使用django REST framework中可能遇到的各种其他问题和解决方案。

### Token方式鉴权实现

### 自定义分页格式

### 链接其他的数据库





