# 更新记录

### 2021-07-23

- 重新使用 Django 3.2.4 构建项目
- 增加 API 文档库 coreapi

## 需要安装的 Python 库

- django
- djangorestframework
- django-cors-headers
- python-jose
- coreapi
- pymysql
- django-filter

## 初始化 Django 工程

```shell
django-admin startproject backend
```

创建好工程后，我们要对目录和配置进行一些调整，首先在根目录下创建两个目录：apps 和 settings，将所有的 app 都存放到 apps 目录里面，把 settings 配置存放在 settings 目录下，这样我们的根目录就更加清晰了

![image-20200906165728250](https://tva1.sinaimg.cn/large/007S8ZIlly1gih1a24xxyj306z07adg0.jpg)

- 调整 settings 配置

  首先将 backend 目录下的 settings.py 文件拷贝到 settings 目录下，创建 dev.py 和 pro.py 两个文件，主要用于开发配置和部署配置，将 settings.py 文件中的数据库配置和 DEBUG 移到这两个文件中，内容如下

  ![开发环境配置](https://tva1.sinaimg.cn/large/007S8ZIlly1gih1j0t8hpj30ex07u74s.jpg)

  ![发布环境](https://tva1.sinaimg.cn/large/007S8ZIlly1gih1jrxwzuj30ei081aak.jpg)

  在 settings.py 文件中把 apps 添加到环境变量中

  ![image-20200906170827998](https://tva1.sinaimg.cn/large/007S8ZIlly1gih1lhq8acj30gw04fdg5.jpg)

  修改语言和时区

  ![image-20200906171003795](https://tva1.sinaimg.cn/large/007S8ZIlly1gih1n5ltc1j306z02dt8m.jpg)

## 增加多数据库配置

- 在 backend 目录下增加 router.py 文件

  路由配置文件当中的返回值是我们在 DATABASES 中配置的键，默认是 default，按照一定的条件返回不同的键，每个键内配置不同的数据库连接，就可以实现 Django 项目连接多个数据库

  ```python
  class CustomRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label in settings.DATABASE_APPS_MAPPING:
            return settings.DATABASE_APPS_MAPPING[model._meta.app_label]

    def db_for_write(self, model, **hints):
        if model._meta.app_label in settings.DATABASE_APPS_MAPPING:
            return settings.DATABASE_APPS_MAPPING[model._meta.app_label]

    def allow_relation(self, obj1, obj2, **hints):
        db_obj1 = settings.DATABASE_APPS_MAPPING.get(obj1._meta.app_label)
        db_obj2 = settings.DATABASE_APPS_MAPPING.get(obj2._meta.app_label)
        if db_obj1 and db_obj2:
            if db_obj1 == db_obj2:
                return True
            else:
                return False

    def allow_syncdb(self, db, model):
        if db in settings.DATABASE_APPS_MAPPING.values():
            return settings.DATABASE_APPS_MAPPING.get(model._meta.app_label) == db
        elif model._meta.app_label in settings.DATABASE_APPS_MAPPING:
            return False

    def allow_migrate(self, db, app_label, model=None, **hints):
        """
        Make sure the auth app only appears in the 'auth_db'
        database.
        """
        if db in settings.DATABASE_APPS_MAPPING.values():
            return settings.DATABASE_APPS_MAPPING.get(app_label) == db
        elif app_label in settings.DATABASE_APPS_MAPPING:
            return False
        return None
  ```

- 在 settings.py 文件中增加路由配置

  ```python
  DATABASE_ROUTERS = ['backend.router.CustomRouter']
  ```

## 设置自定义用户模型

- 在 apps 下增加 users 应用

  在 models.py 下增加如下内容

  ```python
  from django.contrib.auth.models import AbstractUser
  from django.db import models
  from uuid import uuid4


  # Create your models here.

  class UserInfo(AbstractUser):
      uid = models.CharField(max_length=36, default=uuid4, primary_key=True)
      nickname = models.CharField(max_length=32, verbose_name='昵称')
      telephone = models.CharField(max_length=11, blank=True, null=True, unique=True, verbose_name='手机号码')
      create_time = models.DateTimeField(auto_now_add=True, verbose_name='创建时间')

      class Meta:
          db_table = 'tbl_user'
  ```

- 在 settings.py 中增加如下内容

  ```python
  INSTALLED_APPS = [
    ...
    'users'
  ]

  AUTH_USER_MODEL = 'users.UserInfo'
  ```

## 解决跨域问题

为什么会有跨域问题，这里就不做详细解释了，可以看一下两篇文章

- [前后端分离 djangorestframework——解决跨域请求](https://www.cnblogs.com/Eeyhan/p/10440444.html)
- [Django 跨域验证及 OPTIONS 请求](https://w.url.cn/s/AoDZ2R0)

在 settings.py 文件中做如下配置

```python
INSTALLED_APPS = [
  ...
  'corsheaders'
  ...
]

MIDDLEWARE = [
  ...
  'corsheaders.middleware.CorsMiddleware', # 注意必须放在CommonMiddleware前面
  ...
]

CORS_ALLOW_CREDENTIALS = True
CORS_ORIGIN_ALLOW_ALL = True
CORS_ALLOW_METHODS = ['*']

CORS_ALLOW_HEADERS = ['*']
```

## jwt 登录认证

我们使用 rest api 接口，一般就很少使用用户名和密码认真，jwt 认证是比较常用的，因此这也是项目初始化必须做的。要注意

- 在根目录下增加 utils 目录，增加两个文件 authentication.py 和 jwt_util.py

- authentication.py 文件

  ```python
  import time
  from django.db.models import Q
  from rest_framework.authentication import BaseAuthentication
  from rest_framework.exceptions import AuthenticationFailed
  from users.models import UserInfo
  from utils.auth.jwt_util import JwtUtil
  from threading import local

  _thread_local = local()


  class JwtAuthentication(BaseAuthentication):
      def authenticate(self, request):
          access_token = request.META.get('HTTP_ACCESS_TOKEN', None)
          if access_token:
              jwt = JwtUtil()
              data = jwt.check_jwt_token(access_token)
              if data:
                  username = data.get('username')
                  telephone = data.get('telephone')
                  exp = data.get('exp')
                  if time.time() > exp:
                      raise AuthenticationFailed('authentication time out')

                  try:
                      user = UserInfo.objects.get(Q(username=username) | Q(telephone=telephone))
                      _thread_local.user = user
                  except (UserInfo.DoesNotExist, UserInfo.MultipleObjectsReturned) as e:
                      return (None, None)
                  else:
                      return (user, None)

          raise AuthenticationFailed('authentication failed')


  def get_current_user():
      return getattr(_thread_local, 'user', None)
  ```

- jwt_util.py 文件

  ```python
  import time
  from jose import jwt
  from django.conf import settings


  class JwtUtil:
      @staticmethod
    def gen_jwt_token(user):
        to_encode = {
            'username': user.username,
            'telephone': user.telephone,
            'exp': time.time() + 24 * 3600,
        }

        encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
        return encoded_jwt

    @staticmethod
    def check_jwt_token(value):
        playload = {}
        try:
            playload = jwt.decode(value, settings.SECRET_KEY, algorithms=[settings.ALGORITHM]
                                  )
        except Exception as e:
            logger.exception(e)

        return playload
  ```

- 在 settings.py 中增加跨域认证的字段

  ```python

  REST_FRAMEWORK = {
      'DEFAULT_AUTHENTICATION_CLASSES': [
          'utils.authentications.JwtAuthentication'
      ],
      'DATETIME_FORMAT': '%Y-%m-%d %H:%M:%S',
      'DATETIME_INPUT_FORMATS': '%Y-%m-%d %H:%M:%S'
  }
  ```

## 修改登录认证为 JWT 方式

- 在 utils 目录创建 user_backend.py 文件

  ```python
  from django.contrib.auth import backends
  from django.db.models import Q
  from users.models import UserInfo
  from utils.jwt_util import JwtUtil


  class UserBackend(backends.ModelBackend):
      def authenticate(self, request, username=None, password=None, **kwargs):
          user = UserInfo.objects.get(Q(username=username) | Q(telephone=username))
          if user.check_password(password):
              jwt = JwtUtil()
              token = jwt.gen_jwt_token(user)
              user.token = token
              return user
          raise UserInfo.DoesNotExist(
              "UserInfo matching query does not exist."
          )
  ```

- 在 settings 中设置自定义认证方式

  ```python
  AUTHENTICATION_BACKENDS = ['utils.user_backend.UserBackend']
  ```

## Django 日志记录

使用 loguru 模块，一个非常优秀的日志记录开源库，在 utils 目录中增加 logger.py 文件，配置内容如下：

```python
# 日志配置
import time
from pathlib import Path
from loguru import logger

root_path = Path(__file__).parent.parent

# 定位到log日志文件
log_path = root_path.joinpath('logs')
log_path.mkdir(parents=True, exist_ok=True)

log_path_info = log_path.joinpath(f'{time.strftime("%Y-%m-%d")}_info.log')
log_path_warning = log_path.joinpath(f'{time.strftime("%Y-%m-%d")}_warning.log')
log_path_error = log_path.joinpath(f'{time.strftime("%Y-%m-%d")}_error.log')

# 日志简单配置 文件区分不同级别的日志
logger.add(log_path_info,
           rotation="10 MB",
           encoding='utf-8',
           enqueue=True,
           level='INFO',
           retention=5,
           compression='zip')

logger.add(log_path_warning,
           rotation="10 MB",
           encoding='utf-8',
           enqueue=True,
           level='WARNING',
           retention=5,
           compression='zip')

logger.add(log_path_error,
           rotation="10 MB",
           encoding='utf-8',
           enqueue=True,
           level='ERROR',
           retention=5,
           compression='zip')

__all__ = ["logger"]
```

在所有需要记录日志的文件中采用如下方式使用

```python
from utils.logger import logger
...


class RegisterView(APIView):
    authentication_classes = []

    def post(self, request):
        ...
        logger.info('用户: %s 注册成功', username)
        ...
```
