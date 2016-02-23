## Flask-Script参考手册

Flask-Script 扩展为Flask应用添加了一个命令行解析器，它使Flask应用可以通过命令行来运行服务器，自定义Python shell，以及通过脚本来设置数据库、周期性任务以及其它Flask应用本身不提供的功能。

Flask-Script跟Flask工作的方式很相似，通过给Manage实例对象定义和添加命令，然后就可以在命令行中调用这些，命令了。

```Python
# manage.py

from flask.ext.script import Manager

from myapp import app

manager = Manager(app)

@manager.command
def hello():
    print "hello"

if __name__ == "__main__":
    manager.run()
```

运行下面的代码，结果如下：
```Python
python manage.py hello
> hello
```

### 安装Flask-Script

通过pip来安装Flask-Script:
```Python
pip install Flask-Script
```

或者直接从github上下载最新的源代码进行安装：
```Python
git clone https://github.com/smurfix/flask-script.git
cd flask-script
python setup.py develop
```

### 定义并运行命令

首先创建一个Python模块来运行脚本命令，文件名为manage.py。

如果是命令比较多的话，可以将它们分开放在几个文件中存放。

在manage.py中必须创建一个Manager对象，Manager类会记录所有的命令，并处理如何调用这些命令。

```Python
from flask.ext.script import Manager

app = Flask(__name__)
# configure your app

manager = Manager(app)

if __name__ == "__main__":
    manager.run()
```

调用manager.run()方法之后，Manager对象就准备好接收命令行传递的命令了。

Manager实例的时候需要传递一个Flask对象给它，这个参数也可以是一个函数或者可调用对象，只要它们能够返回一个Flask对象就可以。

下一步就是创建和添加命令，可以通过以下三种方式来创建命令：
- 继承Command类
- 使用@command修饰器
- 使用@option修饰器

比方说，我们创建一个简单的hello命令，它会输出"hello, world"信息，它不需要传递任何参数进去，所以比较简单。

##### 方式1

```Python
from flask.ext.script import Command

class Hello(Command):
    "prints hello world"

    def run(self):
        print "hello world"
```

然后给manager对象添加命令：
```Python
manager.add_command('hello', Hello())
```

这条语句必须在`manage.run()`之前执行，接下来就可以执行命令了：
```Python 
python manage.py hello
> hello world
```

也可以直接将`Command`对象传递给`manager.run()`：
```Python
manager.run({'hello' : Hello()})
```

Command类必须定义一个`run()`方法，方法中的位置参数以及可选参数来由命令行中输入的参数来决定。

使用下面的命令获取可用命令以及详细说明：

```Python
python manage.py
```

使用下面的命令获取指定命令的帮助信息：

```Python
python manage.py runserver -？
```

上面的代码会输出`runserver`命令的用法以及Command类中的文档注释。


这种方法比较灵活，但是也很冗长，对于简单的命令，我们可以使用@command修饰器：

##### 方法2

```Python
@manager.command
def hello():
    "Just say hello"
    print "hello"
```

这种方法得到的结果与方法一的结果一致:
```Python
python manage.py hello
> hello
```

一样的，使用下面的命令获取帮助：
```Python
python manage.py -?
> Just say hello
```

##### 方法3

最后，使用@option修饰器，如果想对命令进行更加复杂的控制，可以使用这种方法：
```Python
@manager.option('-n', '--name', help='Your name')
def hello(name):
    print "hello", name
```

在后面，我们会详细讲解@option的用法。

帮助信息可以通过`--help`或者`-h`实现查询，但是预期的结果可能不太理想，因为`-h`也可以用作`--host`的简写，导致Flask无法进行区分。

如果你想恢复`-h`的原始用法，可以给manager对象的	`help_args`设置你想用的参数，例如：
```Python
manager = Manager()
manager.help_args = ('-h','-?','--help')
```

可以在`sub-command`或者`manager`中重写列表：
```Python
def talker(host='localhost'):
    pass
ccmd = ConnectCmd(talker)
ccmd.help_args = ('-?','--help')
manager.add_command("connect", ccmd)
manager.run()
```

这样的话，`manager -h` 将输出帮助信息，而`manager connect -h www.example.com ` 将连接远程主机。

### 给命令添加参数

大多数命令都可以接收可选参数。以上面的hello命令为例，我们可以给它增加一个输出名字的功能：
```Python
python manage.py hello --name=Joe
hello Joe
```

或者：
```Python
python manage.py hello -n Joe
```

为了实现这个功能，可以使用Command类的option_list属性：
```Python
from flask.ext.script import Command, Manager, Option

class Hello(Command):

    option_list = (
        Option('--name', '-n', dest='name'),
    )

    def run(self, name):
        print "hello %s" % name
```

可选参数保存在Option中。

或者，可以在Command类中定义一个get_option()方法：
```Python
class Hello(Command):

    def __init__(self, default_name='Joe'):
        self.default_name=default_name

    def get_options(self):
        return [
            Option('-n', '--name', dest='name', default=self.default_name),
        ]

    def run(self, name):
        print "hello",  name
```

如果你使用@command修饰器，那就更简单了，options参数可以从函数参数中自动提取出来。
```Python
@manager.command
def hello(name):
    print "hello", name
```

然后这样调用命令：
```Python
> python manage.py hello Joe
hello Joe
```

也可以传递格外的参数：
```Python
@manager.command
def hello(name="Fred")
    print "hello", name
```

调用方式如下：
```Python
> python manage.py hello --name=Joe
hello Joe
```

或者:
```Python
> python manage.py hello -n Joe
hello Joe
```

缩写格式是参数的第一个字母，所以`name`对应为`-n`,为了避免混淆，不同的参数首字母最好不同。

注意可选参数为布尔值类型的话：
```Python
@manager.command
def verify(verified=False):
    """
    Checks if verified
    """
    print "VERIFIED?", "YES" if verified else "NO"
```

那就可以这样调用：
```Python
> python manage.py verify
VERIFIED? NO

> python manage.py verify -v
VERIFIED? YES

> python manage.py verify --verified
VERIFIED? YES
```

`@command`修饰器对于大多数简单的命令来说已经足够用了，但是如果你想更加的灵活的进行配置，推荐使用`@option`修饰器。
```Python
@manager.option('-n', '--name', dest='name', default='joe')
def hello(name):
    print "hello", name
```
还可以添加多个配置参数：
```Python
@manager.option('-n', '--name', dest='name', default='joe')
@manager.option('-u', '--url', dest='url', default=None)
def hello(name, url):
    if url is None:
        print "hello", name
    else:
        print "hello", name, "from", url
```

调用方式如下：
```Python
> python manage.py hello -n Joe -u reddit.com
hello Joe from reddit.com
```

或者：
```Python
> python manage.py hello --name=Joe --url=reddit.com
hello Joe from reddit.com
```

### 给manager添加配置参数

配置参数也可以直接传递给Manager对象，这样你就可以直接传递配置参数给应用，而不是单独的命令。例如，你可能需要通过一个标志来设置应用的配置文件，假设你使用工程模式创建应用：
```Python
def create_app(config=None):

    app = Flask(__name__)
    if config is not None:
        app.config.from_pyfile(config)
    # configure your app...
    return app
```

你想直接在命令行中指定配置文件。为了传递config参数，使用Manager对象的`add_option()`方法，它的参数与Option一样。
```Python
manager.add_option('-c', '--config', dest='config', required=False)
```

注意这个方法必须在`manager.run()`方法之前调用。
假设你定义了以下命令：
```Python
@manager.command
def hello(name):
    uppercase = app.config.get('USE_UPPERCASE', False)
    if uppercase:
        name = name.upper()
    print "hello", name
```

那么可以这样调用：
```Python
> python manage.py -c dev.cfg hello joe
hello JOE
```

前提你在dev.cfg文件中设置了`USE_UPPERCASE=True`。

注意`config`参数不是传递给`hello`，如果这样调用的话，会报错：
```Python
> python manage.py hello joe -c dev.cfg
```

因为`-c`参数不属于`hello`命令。

可以给命令和manager添加相同名字的参数，而不会产生冲突，例如：
```Python
@manager.option('-n', '--name', dest='name', default='joe')
@manager.option('-c', '--clue', dest='clue', default='clue')
def hello(name,clue):
    uppercase = app.config.get('USE_UPPERCASE', False)
    if uppercase:
        name = name.upper()
        clue = clue.upper()
    print "hello {0}, get a {1}!".format(name,clue)

> python manage.py -c dev.cfg hello -c cookie -n frank
hello FRANK, get a COOKIE!
```

注意`dest`的名字必须不同。

为了让manager的参数奏效，必须使用Flask的工厂模式创建应用。

### 获取用户输入


