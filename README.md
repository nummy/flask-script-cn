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

Flask-Script提供了很多方法来帮助用户从命令行中获取输入信息，例如：
```Python
from flask.ext.script import Manager, prompt_bool

from myapp import app
from myapp.models import db

manager = Manager(app)

@manager.command
def dropdb():
    if prompt_bool(
        "Are you sure you want to lose all your data"):
        db.drop_all()
```

调用方式如下：
```Python
> python manage.py dropdb
Are you sure you want to lose all your data ? [N]
```

### 默认命令

** runserver **

Flask-Script自身提供了很多定义好的命令，例如`Server`和`Shell`。
`Server`命令用于运行Flask应用服务器：
```Python
from flask.ext.script import Server, Manager
from myapp import create_app

manager = Manager(create_app)
manager.add_command("runserver", Server())

if __name__ == "__main__":
    manager.run()
```

调用方式如下：
```Python
python manage.py runserver
```

`Server`命令有许多参数，可以通过`python manager.py runserver -?`来获取详细帮助信息，也可以在构造函数中重定义默认值：
```Python
server = Server(host="0.0.0.0", port=9000)
```

大多数情况下，`runserver`命令用于开启调试模型运行服务器，以便查找bug，因此，如果没有在配置文件中特别声明的话，`runserver`默认是开启调试模式的，当修改代码的时候，会自动重载服务器。

**shell**

`Shell`命令用于打开一个Python终端，你可以给它传递一个`make_context`参数，这个参数必须是一个可调用对象，并且返回一个字典。
```Python
from flask.ext.script import Shell, Manager

from myapp import app
from myapp import models
from myapp.models import db

def _make_context():
    return dict(app=app, db=db, models=models)

manager = Manager(create_app)
manager.add_command("shell", Shell(make_context=_make_context))
```

这个命令非常有帮助，尤其是当你需要从shell中引入很多模块的时候，它可以节省很多时间。

`Shell`命令默认使用IPython，如果已经安装了的话，否则使用默认的Python终端，当然那你也可以禁用这个功能，给`Shell`构造函数传递`use_ipython`参数，或者在命令中使用`--no-ipython`。

```Python
shell = Shell(use_ipython=False)
```

还可以使用`@manager.shell`修饰器:
```Python
@manager.shell
def make_shell_context():
    return dict(app=app, db=db, models=models)
```

调用方式跟默认的一样：
```Python
> python manage.py shell
```

`shell`命令以及`runserver`命令默认被加载，如果你想要覆盖这些命令，可以使用`add_command（）`或者修饰器，如果你传递`with_default_commands=Flase`给Manager构造函数，这些命令就不会加载了。

```Python
manager = Manager(app, with_default_commands=False)
```

### Sub-Managers

Sub-Manager是Manager的实例对象，它用于给另外一个Manager添加命令。

```Python
def sub_opts(app, **kwargs):
    pass
sub_manager = Manager(sub_opts)

manager = Manager(self.app)
manager.add_command("sub_manager", sub_manager)
```

如果给`sub_manager`添加选项，`sub_opts`会接收相应选项值。应用通过app传递给`sub_manager`。

如果`sub_opts`返回值非空，那么这个值会覆盖app传递的值，这样你就可以创建一个sub-manager来替换整个应用。其一个用途就是创建一个独立的管理应用，如下所示：
```Python
def gen_admin(app, **kwargs):
    from myweb.admin import MyAdminApp
    ## easiest but possibly incomplete way to copy your settings
    return MyAdminApp(config=app.config, **kwargs)
sub_manager = Manager(gen_admin)

manager = Manager(MyApp)
manager.add_command("admin", sub_manager)

> python manage.py runserver
[ starts your normal server ]
> python manage.py admin runserver
[ starts an administrative server ]
```

### 扩展开发

使用sub-manager可以很方便的创建扩展，下面是创建一个数据库扩展的例子:
```Python
manager = Manager(usage="Perform database operations")

@manager.command
def drop():
    "Drops database tables"
    if prompt_bool("Are you sure you want to lose all your data"):
        db.drop_all()


@manager.command
def create(default_data=True, sample_data=False):
    "Creates database tables from sqlalchemy models"
    db.create_all()
    populate(default_data, sample_data)


@manager.command
def recreate(default_data=True, sample_data=False):
    "Recreates database tables (same as issuing 'drop' and then 'create')"
    drop()
    create(default_data, sample_data)


@manager.command
def populate(default_data=False, sample_data=False):
    "Populate database with default data"
    from fixtures import dbfixture

    if default_data:
        from fixtures.default_data import all
        default_data = dbfixture.data(*all)
        default_data.setup()

    if sample_data:
        from fixtures.sample_data import all
        sample_data = dbfixture.data(*all)
        sample_data.setup()
```
然后用户就可以将sub-manager注册到Manager上。
```Python
manager = Manager(app)

from flask.ext.database import manager as database_manager
manager.add_command("database", database_manager)
```

调用方式如下：
```Python
> python manage.py database

 Please provide a command:

 Perform database operations
  create    Creates database tables from sqlalchemy models
  drop      Drops database tables
  populate  Populate database with default data
  recreate  Recreates database tables (same as issuing 'drop' and then 'create')
```

### 异常处理

虽然用户不希望看到异常信息，但是开发者希望可以-看到以便调试。因此，`flask.ext.script.commands`提供了一个异常类`InvalidCommand`来输出异常信息。

命令处理函数：
```Python
from flask.ext.script.commands import InvalidCommand

[… if some command verification fails …]
class MyCommand(Command):
    def run(self, foo=None,bar=None):
        if foo and bar:
                raise InvalidCommand("Options foo and bar are incompatible")
```

在主循环中：
```Python
try:
    MyManager().run()
except InvalidCommand as err:
    print(err, file=sys.stderr)
    sys.exit(1)
```

### 访问本地代理

`Manager`运行在`Flask test context`上下文环境中，也就是说你可以访问本地请求代理，如`current_app`。


