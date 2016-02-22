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



