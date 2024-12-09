
## 前言


本来标题是想叫“生成不重复的四位数”的，不过单纯数字有点局限，推广一下变成不重复 ID 吧\~


这个功能是在做下面图片里这个小项目时遇到的，有点像微信的面对面建群，生成一个随机且不重复的密码，其他人输入这个密码就能加入教室。


[![LearnMates](https://img2024.cnblogs.com/blog/866942/202412/866942-20241208214201169-1930276892.png)](https://github.com)


实现这个功能有不少方法，本文简单记录一下。


## 不依赖第三方库


首先单纯基于 Django ORM 来实现这个功能


先定义一个模型



```
from django.db import models

class MyModel(models.Model):
    unique_code = models.CharField(max_length=4, unique=True)

```

### 单任务


最简单粗暴的方法，写一个死循环



```
import random

def generate_unique_code():
    while True:
        code = str(random.randint(1000, 9999))
        if not MyModel.objects.filter(unique_code=code).exists():
            return code

```

这个实现在单线程测试环境下肯定是没问题的，不过这个操作并不是原子化的，并发环境下可能会生成重复的数字。


### 考虑并发


高并发情况下，可以使用数据库事务或乐观锁。



```
from django.db import transaction, IntegrityError

def create_instance():
	  # 尝试次数
  	retry = 10
    for _ in range(retry):
        code = generate_unique_code()
        try:
            with transaction.atomic():
                instance = MyModel(unique_code=code)
                instance.save()
            return instance
        except IntegrityError:
            # 如果出现唯一性冲突，重新尝试
            continue
    raise Exception("无法生成唯一的四位数")

```

### 预先生成


前面两种方法都要频繁读取数据库，性能比较差。


还可以用空间换时间的方式，因为只是四位数，0000\-9999 这个范围的数字也不多，预先把这一万行存入数据库，加个 `available` 字段


当需要生成唯一 ID 的时候，就先筛选 `available == True` 的数据，然后随机抽取一个；并且把这个字段设置为 `False`


大概思路就是这样


## 使用第三方库


在这个项目里，我搭配使用了这三个库（这也是我写这篇文章的主要目的，记录一下这几个库）


* shortuuid
* hashids
* django\-autoslug


### shortuuid


`shortuuid` 是一个轻量级的库，可以生成比较短的 UUID


使用这个库来实现这个功能的话很简单



```
import shortuuid
shortuuid.ShortUUID(alphabet="0123456789").random(length=4)

```

不过这个项目中，我并没有使用这个库来做这个


事实上，这个库顾名思义有个 uuid，自然是用来做与 python 内置 UUID 有关工作


我用这个库把 `Client` 模型的 ID 简化到 7 位



```
class Client(ModelExt):
    client_id = models.UUIDField(default=uuid.uuid4, editable=False)
    client_key = models.CharField(max_length=100, default=uuid.uuid4)
    user = models.OneToOneField(
        User, on_delete=models.SET_NULL, db_constraint=False, null=True, blank=True, unique=True,
    )
    consumer_name = models.CharField(max_length=100, null=True, blank=True)
    is_online = models.BooleanField(default=False)

    def short_client_id(self):
        short = shortuuid.encode(self.client_id)
        return short[:7]

```

使用 `shortuuid.encode` 方法可以把 32 位的 UUID 变成 22 位，并且还能使用 `decode` 方法复原


### hashids


这个是将数字转为短字符串的库


虽然名字里带个 hash ，但这个库的编码是可逆的



```
import hashids
h = hashids.Hashids()
h.encode(123, 456)
# Out[15]: 'X68fkp'
h.decode('X68fkp')
# Out[16]: (123, 456)

```

我用来根据时间戳生成教室名称



```
def get_timestamp_hashid():
    hashids = Hashids(salt='hahaha salt lala')
    t = timezone.now().timestamp()
    result = tuple(map(int, str(t).split('.')))
    return hashids.encode(*result)

```

因为 `encode` 方法接收的是数字（也可以是包含数字的 tuple），所以这里把时间戳的整数部分和小数部分转换为 tuple 然后传入 hashids


### django\-autoslug


这个库用于生成基于字段的唯一 slug，同时可以自定义生成逻辑。


我就是用这个库来实现生成唯一的教室密码功能（但并不是很推荐这种方式）



```
from autoslug import AutoSlugField


def populate_classroom_number(instance):
    return str(random.randint(1000, 9999))
  
class ClassroomIdiom(ModelExt):
    name = models.CharField(max_length=100)
    number = AutoSlugField(populate_from=populate_classroom_number, unique=True)

```

这个库的原理很简单，根据用户定义的规则生成 slug，然后检查数据库是否重复，遇到重复的话就在后面追加数字，这样有可能导致生成出来的数字超过 4 位数


最好的还是我前面说的 **不依赖第三方库** 的第三种方式。


这里使用 django\-autoslug 单纯是为了偷懒，把复杂的判断逻辑交给第三方库，毕竟这只是个玩具项目。


## 小结


在生成唯一 ID 这件事上，Django 和其他后端框架没啥不同的，思路都是类似的，只不过可以借助 Python 生态偷懒一下…


 本博客参考[PodHub豆荚加速器官方网站](https://rikeduke.com)。转载请注明出处！
