
# Flask官网一个案例的中文解析 — Flaskr

官方文档：http://flask.pocoo.org/docs/1.0/tutorial/

源代码：https://github.com/pallets/flask/tree/1.0.2/examples/tutorial




## Project Layout

创建文件目录，并进入

```
mkdir flask-tutorial
cd flask-tutorial
```

然后按照安装说明设置Python虚拟环境并为项目安装Flask。

本教程假设您从现在开始使用flask-tutorial目录。 每个代码块顶部的文件名都与该目录相关。

---
Flask应用程序可以像单个文件一样简单。



```python
# hello.py
from flask import Flask

app = Flask(__name__)


@app.route('/')
def hello():
    return 'Hello, World!'



```

但是，随着项目变得越来越大，将所有代码保存在一个文件中变得难以忍受。 Python项目使用 *packages（包）*将代码组织成多个模块，可以在需要的地方导入，教程也会这样做。

项目目录将包含：

- `flaskr/`，一个包含应用程序代码和文件的Python包。
- `tests/`，包含测试模块的目录。
- `venv/`，一个安装了Flask和其他依赖项的Python虚拟环境。
- 安装文件告诉Python如何安装项目。
- 版本控制配置，例如`git`。 您应养成为所有项目使用某种类型的版本控制的习惯，无论大小如何。
- 您将来可能添加的任何其他项目文件。

最后，您的项目布局将如下所示：
```
/home/user/Projects/flask-tutorial
├── flaskr/
│   ├── __init__.py
│   ├── db.py
│   ├── schema.sql
│   ├── auth.py
│   ├── blog.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── auth/
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── blog/
│   │       ├── create.html
│   │       ├── index.html
│   │       └── update.html
│   └── static/
│       └── style.css
├── tests/
│   ├── conftest.py
│   ├── data.sql
│   ├── test_factory.py
│   ├── test_db.py
│   ├── test_auth.py
│   └── test_blog.py
├── venv/
├── setup.py
└── MANIFEST.in
```

如果您正在使用版本控制，则应忽略在运行项目时生成的以下文件。 根据您使用的编辑器，可能还有其他文件。 通常，忽略您未编写的文件。 例如，使用git：

.gitignore
```
venv/

*.pyc
__pycache__/

instance/

.pytest_cache/
.coverage
htmlcov/

dist/
build/
*.egg-info/
```

## Application Setup
Flask应用程序是Flask类的一个实例。 有关应用程序的所有内容（例如配置和URL）都将在此类中注册。

创建Flask应用程序最直接的方法是直接在代码顶部创建一个全局Flask实例，就像“Hello，World！”示例在上一页中所做的那样。 虽然这在某些情况下很简单且有用，但随着项目的增长，它可能会导致一些棘手的问题。

**您可以在函数内创建Flask实例，而不是全局创建Flask实例**。 此功能称为应用程序工厂。 **应用程序需要的任何配置，注册和其他设置都将在函数内部进行，然后将返回应用程序**。



### The Application Factory
是时候开始编码了！ 创建`flaskr`目录并添加`__init__.py`文件。 `__init__.py`提供双重任务：它将包含应用程序工厂，它告诉Python应该将`flaskr`目录视为包。

```
mkdir flaskr
```


```python
# flaskr/__init__.py


import os

from flask import Flask


def create_app(test_config=None):
    # create and configure the app
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_mapping(
        SECRET_KEY='dev',
        DATABASE=os.path.join(app.instance_path, 'flaskr.sqlite'),
    )

    if test_config is None:
        # load the instance config, if it exists, when not testing
        app.config.from_pyfile('config.py', silent=True)
    else:
        # load the test config if passed in
        app.config.from_mapping(test_config)

    # ensure the instance folder exists
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    # a simple page that says hello
    @app.route('/hello')
    def hello():
        return 'Hello, World!'

    return app

```

`create_app` 是应用程序工厂函数。您将在本教程的后面添加它，但它已经做了很多。

1.`app = Flask(__name__,instance_relative_config=True)`创建`Flask`实例。  

- `__name__` 是当前Python模块的名称。应用程序需要知道它在哪里设置一些路径，而`__name__` 是一种方便的方式来告诉它。

- `instance_relative_config = True` 告诉应用程序配置文件是相对于instance文件夹的。instance文件夹位于flaskr包外部，可以保存不应提交到版本控制的本地数据，例如配置机密和数据库文件。 
  

2.`app.config.from_mapping()` 设置应用程序将使用的一些默认配置：  

- Flask和扩展使用 `SECRET_KEY` 来保证数据安全。它被设置为 `'dev'` 以在开发期间提供方便的值，但在部署时应该用随机值覆盖它。  

- `DATABASE` 是保存SQLite数据库文件的路径。它位于 `app.instance_path` 下，这是Flask为instance文件夹选择的路径。您将在下一节中了解有关数据库的更多信息。 

3.`app.config.from_pyfile()` 使用实例文件夹中`config.py`文件中的值（如果存在）覆盖默认配置。例如，在部署时，这可用于设置真正的`SECRET_KEY`。

- `test_config`也可以传递给工厂，并将用于代替实例配置。这样您将在本教程后面编写的测试可以独立于您配置的任何开发值进行配置。 

4.`os.makedirs()` 确保`app.instance_path`存在。 Flask不会自动创建instance文件夹，但需要创建它，因为您的项目将在那里创建SQLite数据库文件。

5.**\@app.route()** 创建一个简单的路由，以便在进入本教程的其余部分之前可以看到应用程序正常工作。它在URL `/hello`和返回响应的函数之间创建了一个连接，字符串`'Hello，World！'`在这种情况下。




### Run The Application
现在，您可以使用flask命令运行应用程序。 从终端，告诉Flask在哪里找到您的应用程序，然后在开发模式下运行它。

只要页面引发异常，开发模式就会显示交互式调试器，并在您更改代码时重新启动服务器。 您可以让它保持运行，只需按照教程重新加载浏览器页面即可。  

For Linux and Mac:
```
export FLASK_APP=flaskr
export FLASK_ENV=development
flask run
```
For Windows cmd, use set instead of export:
```
set FLASK_APP=flaskr
set FLASK_ENV=development 
flask run
```
For Windows PowerShell, use \$env: instead of export: 

```
$env:FLASK_APP = "flaskr"
$env:FLASK_ENV = "development"
flask run
```

You’ll see output similar to this:
```
* Serving Flask app "flaskr"
* Environment: development
* Debug mode: on
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger PIN: 855-212-761
```

Visit http://127.0.0.1:5000/hello in a browser and you should see the “Hello, World!” message. Congratulations, you’re now running your Flask web application!

## Define and Access the Database

该应用程序将使用SQLite数据库来存储用户和帖子。 Python在`sqlite3`模块中内置了对SQLite的支持。

SQLite很方便，因为它不需要设置单独的数据库服务器并且内置于Python。 但是，如果并发请求尝试同时写入数据库，则随着每次写入顺序发生，它们将变慢。 小应用程序不会注意到这一点。 一旦变大，您可能希望切换到其他数据库。

本教程不会详细介绍SQL。 如果您不熟悉它，SQLite文档会描述该语言。

### Connect to the Database
使用SQLite数据库（以及大多数其他Python数据库库）时要做的第一件事就是创建一个连接。 使用连接执行任何查询和操作，该连接在工作完成后关闭。

在Web应用程序中，此连接通常与请求相关联。 它在处理请求时在某个时刻创建，并在发送响应之前关闭。



```python
# flaskr/db.py

import sqlite3

import click
from flask import current_app, g
from flask.cli import with_appcontext


def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(
            current_app.config['DATABASE'],
            detect_types=sqlite3.PARSE_DECLTYPES
        )
        g.db.row_factory = sqlite3.Row

    return g.db


def close_db(e=None):
    db = g.pop('db', None)

    if db is not None:
        db.close()
        
        
```

`g`是一个特殊对象，对每个请求都是唯一的。它用于存储请求期间可能由多个函数访问的数据。如果在同一请求中第二次调用`get_db`，则会存储并重用连接，而不是创建新连接。

`current_app`是另一个指向处理请求的Flask应用程序的特殊对象。由于您使用的是应用程序工厂，因此在编写其余代码时没有应用程序对象。创建应用程序并处理请求时将调用`get_db`，因此可以使用`current_app`。

`sqlite3.connect()` 建立与`DATABASE`配置键指向的文件的连接。此文件尚不存在，并且在您稍后初始化数据库之前不会存在。

`sqlite3.Row` 告诉连接返回行为类似于dicts的行。这允许按名称访问列。

`close_db`通过检查是否已设置`g.db`来检查是否创建了连接。如果连接存在，则关闭。再往下，您将告诉您的应用程序有关应用程序工厂中的`close_db`函数，以便在每次请求后调用它。

### Create the Tables
在SQLite中，数据存储在表和列中。 这些需要在存储和检索数据之前创建。 Flaskr将用户存储在`user`表中，并在`post`表中发布。 使用创建空表所需的SQL命令创建一个文件：


```python
# flaskr/schema.sql

DROP TABLE IF EXISTS user;
DROP TABLE IF EXISTS post;

CREATE TABLE user (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);

CREATE TABLE post (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  author_id INTEGER NOT NULL,
  created TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES user (id)
);
```

将运行这些SQL命令的Python函数添加到`db.py`文件中：


```python
# flaskr/db.py

def init_db():
    db = get_db()

    with current_app.open_resource('schema.sql') as f:
        db.executescript(f.read().decode('utf8'))


@click.command('init-db')
@with_appcontext
def init_db_command():
    """Clear the existing data and create new tables."""
    init_db()
    click.echo('Initialized the database.')
    
    
    
```

`open_resource()` 打开一个相对于`flaskr`包的文件，这很有用，因为您以后在部署应用程序时不一定知道该位置的位置。 `get_db`返回数据库连接，用于执行从文件读取的命令。

`click.command()` 定义了一个名为`init-db`的命令行命令，该命令调用`init_db`函数并向用户显示成功消息。 您可以阅读命令行界面以了解有关编写命令的更多信息。

@with_appcontext 是什么意思？？？


### Register with the Application
需要向应用程序实例注册`close_db`和`init_db_command`函数，否则应用程序将不会使用它们。 但是，由于您使用的是工厂函数，因此在编写函数时该实例不可用。 相反，编写一个接受应用程序并进行注册的函数。


```python
# flaskr/db.py
def init_app(app):
    app.teardown_appcontext(close_db)
    app.cli.add_command(init_db_command)
    
    
```

`app.teardown_appcontext()` 告诉Flask在返回响应后清理时调用该函数。

`app.cli.add_command()` 添加了一个可以使用`flask` 命令调用的新命令。

从工厂导入并调用此功能。 在返回应用程序之前，将新代码放在工厂函数的末尾。


```python
#flaskr/__init__.py

def create_app():
    app = ...
    # existing code omitted

    from . import db
    db.init_app(app)

    return app


```

### Initialize the Database File
现在已经在应用程序中注册了`init-db`，可以使用`flask`命令调用它，类似于上一页的`run`命令。

> 注意：
> 如果您仍在从上一页运行服务器，则可以停止服务器，也可以在新终端中运行此命令。 如果您使用新终端，请记住更改到项目目录并激活env，如激活环境中所述。 您还需要设置`FLASK_APP`和`FLASK_ENV`，如上一页所示。

运行`init-db`命令：
```
flask init-db
Initialized the database.
```
现在，项目中的实例文件夹中将有一个`flaskr.sqlite`文件。

## Blueprints and Views
视图函数是您编写的代码，用于响应对您的应用程序的请求。 Flask使用模式将传入的请求URL与应该处理它的视图相匹配。 该视图返回Flask变为传出响应的数据。 Flask也可以转向另一个方向，并根据其名称和参数生成视图的URL。

### Create a Blueprint
蓝图是一种组织一组相关视图和其他代码的方法。 它们不是直接在应用程序中注册视图和其他代码，而是使用蓝图进行注册。 然后，当工具在工厂功能中可用时，蓝图就会在应用程序中注册。

Flaskr将有两个蓝图，一个用于身份验证功能，另一个用于博客帖子功能。 **每个蓝图的代码都将在一个单独的模块中**。 由于博客需要了解身份验证，因此您首先要编写身份验证。




```python
# flaskr/auth.py
import functools

from flask import (
    Blueprint, flash, g, redirect, render_template, request, session, url_for
)
from werkzeug.security import check_password_hash, generate_password_hash

from flaskr.db import get_db

bp = Blueprint('auth', __name__, url_prefix='/auth')


```

这将创建一个名为`'auth'`的蓝图。 与应用程序对象一样，蓝图需要知道它的定义位置，因此`__name__`作为第二个参数传递。 **`url_prefix`将添加到与蓝图关联的所有URL之前**。

使用`app.register_blueprint()`从工厂导入并注册蓝图。 在返回应用程序之前，将新代码放在工厂函数的末尾。


**每一个蓝图都要在create_app里register后才能使用！！！**




```python
# flaskr/__init__.py
def create_app():
    app = ...
    # existing code omitted

    from . import auth
    app.register_blueprint(auth.bp)

    return app


```

身份验证蓝图将具有**注册新用户**以及**登录**和**注销**的视图。

### The First View: Register
当用户访问`/auth/register`URL时，`register`视图将返回带有表单的`HTML`以供填写。 当他们提交表单时，它将验证他们的输入并再次显示表单并显示错误消息或创建新用户并转到登录页面。

现在您只需编写视图代码。 在下一页中，您将编写模板以生成`HTML`表单。


```python
# flaskr/auth.py

@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None

        if not username:
            error = 'Username is required.'
        elif not password:
            error = 'Password is required.'
        elif db.execute(
            'SELECT id FROM user WHERE username = ?', (username,)
        ).fetchone() is not None:
            error = 'User {} is already registered.'.format(username)

        if error is None:
            db.execute(
                'INSERT INTO user (username, password) VALUES (?, ?)',
                (username, generate_password_hash(password))
            )
            db.commit()
            return redirect(url_for('auth.login'))

        flash(error)

    return render_template('auth/register.html')
```

这是`register`视图函数正在执行的操作：

1.**\@bp.route** 将URL `/register`与`register`视图功能相关联。当Flask收到对`/auth/register`的请求时，它将调用`register`视图并使用返回值作为响应。

2.如果用户提交了表单，`request.method`将为`'POST'`。在这种情况下，请开始验证输入。

3.`request.form`是一种特殊类型的`dict`映射，提交了表单键和值。用户将输入他们的用户名和密码。

4.验证`username`和`password`不为空。

5.通过查询数据库并检查是否返回结果来验证尚未注册`username`。 `db.execute`采用SQL查询 `?` 任何用户输入的占位符，以及用于替换占位符的值元组。数据库库将负责转义值，因此您不容易受到SQL注入攻击。

`fetchone()` 从查询中返回一行。如果查询未返回任何结果，则返回None。稍后，使用`fetchall()`，它返回所有结果的列表。

6.如果验证成功，请将新用户数据插入数据库。为安全起见，密码不应直接存储在数据库中。相反，`generate_password_hash()`用于安全地散列密码，并存储该散列。**由于此查询修改数据，因此需要在之后调用`db.commit()`以保存更改**。

7.存储用户后，它们将被重定向到登录页面。 **`url_for()`根据其名称生成登录视图的URL**。这比直接编写URL更好，因为它允许您稍后更改URL而不更改链接到它的所有代码。 `redirect()`生成对生成的URL的重定向响应。

8.如果验证失败，则向用户显示错误。 `flash()`存储在呈现模板时可以检索的消息。

9.当用户最初导航到`auth/register`，或者存在验证错误时，应显示带有注册表单的HTML页面。 `render_template()`将呈现包含HTML的模板，您将在本教程的下一步中编写该模板。



### Login
该视图遵循与上面的`register`视图相同的模式。




```python
# flaskr/auth.py
@bp.route('/login', methods=('GET', 'POST'))
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None
        user = db.execute(
            'SELECT * FROM user WHERE username = ?', (username,)
        ).fetchone()

        if user is None:
            error = 'Incorrect username.'
        elif not check_password_hash(user['password'], password):
            error = 'Incorrect password.'

        if error is None:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('index'))

        flash(error)

    return render_template('auth/login.html')


```

这里与`register`视图有一些不同之处：

1. 首先查询用户并将其存储在变量中以供以后使用。 

2. `check_password_hash()`以与存储的哈希相同的方式哈希提交的密码并安全地比较它们。 如果匹配，则密码有效。  

3. `session`是一个跨请求存储数据的`dict`。 验证成功后，用户的`id`将存储在新会话中。 数据存储在发送到浏览器的cookie中，然后浏览器将其与后续请求一起发回。 Flask安全地对数据进行签名，以便不会被篡改。

既然用户的`id`存储在`session`中，它将在后续请求中可用。 在每个请求开始时，如果用户已登录，则应加载其信息并使其可供其他视图使用。


```python
# flaskr/auth.py
@bp.before_app_request
def load_logged_in_user():
    user_id = session.get('user_id')

    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute(
            'SELECT * FROM user WHERE id = ?', (user_id,)
        ).fetchone()

```

`bp.before_app_request()`注册一个在视图函数之前运行的函数，无论请求什么URL。 `load_logged_in_user`检查用户id是否存储在`session`中，并从数据库中获取该用户的数据，并将其存储在`g.user`上，该持续时间为请求的长度。 如果没有用户id，或者id不存在，则`g.user`将为`None`。

### Logout
要注销，您需要从`session`中删除用户id。 然后`load_logged_in_user`将不会在后续请求中加载用户。


```python
# flaskr/auth.py
@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))

```

### Require Authentication in Other Views
创建，编辑和删除博客帖子将需要用户登录。装饰器可用于检查它应用于的每个视图。


```python
#flaskr/auth.py
def login_required(view):
    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            return redirect(url_for('auth.login'))

        return view(**kwargs)

    return wrapped_view

```

此装饰器返回一个新的视图函数，该函数包装它应用的原始视图。 新函数检查用户是否已加载，否则重定向到登录页面。 如果加载了用户，则调用原始视图并继续正常。 在编写博客视图时，您将使用此装饰器。

### Endpoints and URLs
`url_for()`函数根据名称和参数生成视图的URL。 与视图关联的名称也称为端点，默认情况下，它与视图函数的名称相同。

例如，在本教程前面添加到app工厂的`hello()`视图名为`'hello'`，可以与`url_for('hello')`链接。 如果它接受了一个参数，你将在后面看到它，它将与使用`url_for('hello'，who ='World')`相关联。

使用蓝图时，蓝图的名称将附加到函数的名称前面，因此上面写入的登录函数的端点是`“auth.login”`，因为您将其添加到`“auth”`蓝图中。

## Templates
您已为应用程序编写了身份验证视图，但如果您正在运行服务器并尝试转到任何URL，则会看到`TemplateNotFound`错误。那是因为视图调用了`render_template()`，但是你还没有编写模板。模板文件将存储在`flaskr`包内的`templates`目录中。

模板是包含静态数据的文件以及动态数据的占位符。使用特定数据呈现模板以生成最终文档。 Flask使用`Jinja`模板库来渲染模板。

在您的应用程序中，您将使用模板呈现`HTML`，该HTML将显示在用户的浏览器中。在Flask中，**Jinja配置为自动显示在HTML模板中呈现的任何数据**。这意味着呈现用户输入是安全的;他们输入的任何字符可能会弄乱HTML，例如`<`和`>`将使用安全值进行转义，这些值在浏览器中看起来相同但不会产生不良影响。

Jinja看起来和行为大多像Python。特殊分隔符用于区分Jinja语法与模板中的静态数据。 **`{{`和`}}`之间的任何内容都是将输出到最终文档的表达式。 `{％`和`％}`表示控制流语句，如`if`和`for`**。与Python不同，块由开始和结束标记而不是缩进表示，因为块内的静态文本可能会更改缩进。

### The Base Layout
应用程序中的每个页面将围绕不同的主体具有相同的基本布局。 **每个模板不是在每个模板中编写整个HTML结构，而是扩展基本模板并覆盖特定部分**。


```python
# flaskr/templates/base.html
<!doctype html>
<title>{% block title %}{% endblock %} - Flaskr</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<nav>
  <h1>Flaskr</h1>
  <ul>
    {% if g.user %}
      <li><span>{{ g.user['username'] }}</span>
      <li><a href="{{ url_for('auth.logout') }}">Log Out</a>
    {% else %}
      <li><a href="{{ url_for('auth.register') }}">Register</a>
      <li><a href="{{ url_for('auth.login') }}">Log In</a>
    {% endif %}
  </ul>
</nav>
<section class="content">
  <header>
    {% block header %}{% endblock %}
  </header>
  {% for message in get_flashed_messages() %}
    <div class="flash">{{ message }}</div>
  {% endfor %}
  {% block content %}{% endblock %}
</section>
```

`g`在模板中自动可用。根据是否设置了`g.user`（来自`load_logged_in_user`），显示用户名和注销链接，否则显示注册和登录的链接。 `url_for()`也可自动使用，用于生成视图的URL，而不是手动写出来。

在页面标题之后，在内容之前，模板循环遍历`get_flashed_messages()`返回的每条消息。您在视图中使用`flash()`来显示错误消息，这是将显示它们的代码。

此处定义的三个块将在其他模板中被覆盖：

1. `{％block title％}`将更改浏览器标签和窗口标题中显示的标题。
2. `{％block header％}`与`title`类似，但会更改页面上显示的标题。
3. `{％block content％}`是每个页面的内容所在的位置，例如登录表单或博客文章。  

基本模板直接位于`templates`目录中。**为了保持其他组织的有序性，蓝图的模板将放置在与蓝图*同名*的目录中。**

### Register



```python
# flaskr/templates/auth/register.html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Register{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="username">Username</label>
    <input name="username" id="username" required>
    <label for="password">Password</label>
    <input type="password" name="password" id="password" required>
    <input type="submit" value="Register">
  </form>
{% endblock %}
```

**`{％extends'base.html'％}`告诉Jinja该模板应该替换基本模板中的块**。 所有呈现的内容必须出现在覆盖基本模板中的块的`{％block％}`标记内。

这里使用的有用模式是将`{％block title％}`放在`{％block header％}`中。 这将设置标题栏，然后将其值输出到标题栏中，这样窗口和页面共享相同的标题而不写两次。

**`input`标记在此处使用`required`属性。 这告诉浏览器在填写这些字段之前不要提交表单。**如果用户使用的旧浏览器不支持该属性，或者他们使用浏览器之外的其他内容来发出请求，那么您仍然需要验证 Flask视图中的数据。 始终完全验证服务器上的数据非常重要，即使客户端也进行了一些验证。


### Log in
除了标题和提交按钮之外，这与register模板相同。


```python
# flaskr/templates/auth/login.html¶
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Log In{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="username">Username</label>
    <input name="username" id="username" required>
    <label for="password">Password</label>
    <input type="password" name="password" id="password" required>
    <input type="submit" value="Log In">
  </form>
{% endblock %}
```

### Register A User
现在已经编写了身份验证模板，您可以注册用户。 确保服务器仍在运行（如果不是，则`run flask`），然后转到http://127.0.0.1:5000/auth/register。

尝试单击“注册”按钮而不填写表单，并看到浏览器显示错误消息。 尝试从`register.html`模板中删除`required`的属性，然后再次单击“注册”。 页面将重新加载，而不是浏览器显示错误，将显示视图中`flash()`的错误。

填写用户名和密码，您将被重定向到登录页面。 尝试输入错误的用户名，或正确的用户名和错误的密码。 如果您登录，您将收到错误，因为没有`index`视图可以重定向到。

## Static Files
身份验证视图和模板可以正常工作，但它们现在看起来很简单。 可以添加一些CSS来为您构建的HTML布局添加样式。 样式不会改变，因此它是静态文件而不是模板。

Flask会自动添加一个`static`视图，该视图采用相对于`flaskr/static`目录的路径并为其提供服务。 `base.html`模板已经有一个指向`style.css`文件的链接：
```
{{ url_for('static', filename='style.css') }}
```
除了CSS之外，其他类型的静态文件可能是具有JavaScript功能的文件或logo图像。 它们都放在`flaskr/static`目录下，并用`url_for('static'，filename ='...')`引用。

本教程不关注如何编写CSS，因此您只需将以下内容复制到`flaskr/static/style.css`文件中：


```python
#flaskr/static/style.css
html { font-family: sans-serif; background: #eee; padding: 1rem; }
body { max-width: 960px; margin: 0 auto; background: white; }
h1 { font-family: serif; color: #377ba8; margin: 1rem 0; }
a { color: #377ba8; }
hr { border: none; border-top: 1px solid lightgray; }
nav { background: lightgray; display: flex; align-items: center; padding: 0 0.5rem; }
nav h1 { flex: auto; margin: 0; }
nav h1 a { text-decoration: none; padding: 0.25rem 0.5rem; }
nav ul  { display: flex; list-style: none; margin: 0; padding: 0; }
nav ul li a, nav ul li span, header .action { display: block; padding: 0.5rem; }
.content { padding: 0 1rem 1rem; }
.content > header { border-bottom: 1px solid lightgray; display: flex; align-items: flex-end; }
.content > header h1 { flex: auto; margin: 1rem 0 0.25rem 0; }
.flash { margin: 1em 0; padding: 1em; background: #cae6f6; border: 1px solid #377ba8; }
.post > header { display: flex; align-items: flex-end; font-size: 0.85em; }
.post > header > div:first-of-type { flex: auto; }
.post > header h1 { font-size: 1.5em; margin-bottom: 0; }
.post .about { color: slategray; font-style: italic; }
.post .body { white-space: pre-line; }
.content:last-child { margin-bottom: 0; }
.content form { margin: 1em 0; display: flex; flex-direction: column; }
.content label { font-weight: bold; margin-bottom: 0.5em; }
.content input, .content textarea { margin-bottom: 1em; }
.content textarea { min-height: 12em; resize: vertical; }
input.danger { color: #cc2f2e; }
input[type=submit] { align-self: start; min-width: 10em; }
```

您可以在示例代码中找到一个不太紧凑的style.css版本。
```
html {
  font-family: sans-serif;
  background: #eee;
  padding: 1rem;
}

body {
  max-width: 960px;
  margin: 0 auto;
  background: white;
}

h1, h2, h3, h4, h5, h6 {
  font-family: serif;
  color: #377ba8;
  margin: 1rem 0;
}

a {
  color: #377ba8;
}

hr {
  border: none;
  border-top: 1px solid lightgray;
}

nav {
  background: lightgray;
  display: flex;
  align-items: center;
  padding: 0 0.5rem;
}

nav h1 {
  flex: auto;
  margin: 0;
}

nav h1 a {
  text-decoration: none;
  padding: 0.25rem 0.5rem;
}

nav ul  {
  display: flex;
  list-style: none;
  margin: 0;
  padding: 0;
}

nav ul li a, nav ul li span, header .action {
  display: block;
  padding: 0.5rem;
}

.content {
  padding: 0 1rem 1rem;
}

.content > header {
  border-bottom: 1px solid lightgray;
  display: flex;
  align-items: flex-end;
}

.content > header h1 {
  flex: auto;
  margin: 1rem 0 0.25rem 0;
}

.flash {
  margin: 1em 0;
  padding: 1em;
  background: #cae6f6;
  border: 1px solid #377ba8;
}

.post > header {
  display: flex;
  align-items: flex-end;
  font-size: 0.85em;
}

.post > header > div:first-of-type {
  flex: auto;
}

.post > header h1 {
  font-size: 1.5em;
  margin-bottom: 0;
}

.post .about {
  color: slategray;
  font-style: italic;
}

.post .body {
  white-space: pre-line;
}

.content:last-child {
  margin-bottom: 0;
}

.content form {
  margin: 1em 0;
  display: flex;
  flex-direction: column;
}

.content label {
  font-weight: bold;
  margin-bottom: 0.5em;
}

.content input, .content textarea {
  margin-bottom: 1em;
}

.content textarea {
  min-height: 12em;
  resize: vertical;
}

input.danger {
  color: #cc2f2e;
}

input[type=submit] {
  align-self: start;
  min-width: 10em;
}
```

转到http://127.0.0.1:5000/auth/login
该页面应如下面的屏幕截图所示。

您可以从Mozilla的文档中阅读有关CSS的更多信息。 如果更改静态文件，请刷新浏览器页面。 如果更改未显示，请尝试清除浏览器的缓存。

## Blog Blueprint
在编写身份验证蓝图以编写博客蓝图时，您将使用与学到的相同的技术。 博客应列出所有帖子，允许登录用户创建帖子，并允许帖子的作者编辑或删除帖子。

在实现每个视图时，请保持开发服务器正常运行。 在保存更改时，请尝试访问浏览器中的URL并进行测试。

### The Blueprint
定义一个蓝图并在应用程序工厂里注册。


```python
# flaskr/blog.py
from flask import (
    Blueprint, flash, g, redirect, render_template, request, url_for
)
from werkzeug.exceptions import abort

from flaskr.auth import login_required
from flaskr.db import get_db

bp = Blueprint('blog', __name__)
```

使用`app.register_blueprint()`从工厂导入并注册蓝图。 在返回应用程序之前，将新代码放在工厂函数的末尾。


```python
#flaskr/__init__.py
def create_app():
    app = ...
    # existing code omitted

    from . import blog
    app.register_blueprint(blog.bp)
    app.add_url_rule('/', endpoint='index')

    return app
```

与auth蓝图不同，blog蓝图没有`url_prefix`。 因此`index`视图将位于`/`，`create`视图位于`/create`，依此类推。 **博客是Flaskr的主要特色，因此博客索引将成为主要索引。**

但是，下面定义的`index`视图的端点将是`blog.index`。 某些auth视图引用了普通`index`端点。 `app.add_url_rule()` 将端点名称`'index'`与`/` url相关联，以便`url_for('index')`或`url_for('blog.index')`都可以工作，从而生成相同的`/` URL。

在另一个应用程序中，您可以为博客蓝图提供`url_prefix`，并在应用程序工厂中定义单独的`index`视图，类似于`hello`视图。 然后`index`和`blog.index`端点和URL会有所不同。

### Index
索引将显示所有帖子，最近的帖子。 使用`JOIN`以便结果中可以使用`user`表中的作者信息。


```python
# flaskr/blog.py
@bp.route('/')
def index():
    db = get_db()
    posts = db.execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' ORDER BY created DESC'
    ).fetchall()
    return render_template('blog/index.html', posts=posts)
```


```python
# flaskr/templates/blog/index.html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Posts{% endblock %}</h1>
  {% if g.user %}
    <a class="action" href="{{ url_for('blog.create') }}">New</a>
  {% endif %}
{% endblock %}

{% block content %}
  {% for post in posts %}
    <article class="post">
      <header>
        <div>
          <h1>{{ post['title'] }}</h1>
          <div class="about">by {{ post['username'] }} on {{ post['created'].strftime('%Y-%m-%d') }}</div>
        </div>
        {% if g.user['id'] == post['author_id'] %}
          <a class="action" href="{{ url_for('blog.update', id=post['id']) }}">Edit</a>
        {% endif %}
      </header>
      <p class="body">{{ post['body'] }}</p>
    </article>
    {% if not loop.last %}
      <hr>
    {% endif %}
  {% endfor %}
{% endblock %}
```

当用户登录时，`header`块会添加指向`create`视图的链接。 当用户是帖子的作者时，他们会看到指向该帖子的`update`视图的“Edit”链接。 `loop.last`是Jinja for loop中可用的特殊变量。 它用于在每个帖子之后显示除最后一个之外的一行，以便在视觉上将它们分开。

### Create
`create`视图与auth `register`视图的工作方式相同。 显示表单，或验证发布的数据，并将帖子添加到数据库或显示错误。

您之前编写的`login_required`装饰器用于博客视图。 用户必须登录才能访问这些视图，否则他们将被重定向到登录页面。


```python
# flaskr/blog.py
@bp.route('/create', methods=('GET', 'POST'))
@login_required
def create():
    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        if error is not None:
            flash(error)
        else:
            db = get_db()
            db.execute(
                'INSERT INTO post (title, body, author_id)'
                ' VALUES (?, ?, ?)',
                (title, body, g.user['id'])
            )
            db.commit()
            return redirect(url_for('blog.index'))

    return render_template('blog/create.html')
```


```python
#flaskr/templates/blog/create.html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}New Post{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="title">Title</label>
    <input name="title" id="title" value="{{ request.form['title'] }}" required>
    <label for="body">Body</label>
    <textarea name="body" id="body">{{ request.form['body'] }}</textarea>
    <input type="submit" value="Save">
  </form>
{% endblock %}
```

### Update
`update`和`delete`视图都需要通过`id`获取`post`并检查作者是否与登录用户匹配。 为避免重复代码，您可以编写一个函数来获取帖子并从每个视图中调用它。


```python
# flaskr/blog.py
def get_post(id, check_author=True):
    post = get_db().execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' WHERE p.id = ?',
        (id,)
    ).fetchone()

    if post is None:
        abort(404, "Post id {0} doesn't exist.".format(id))

    if check_author and post['author_id'] != g.user['id']:
        abort(403)

    return post
```

`abort()`将引发一个返回HTTP状态代码的特殊异常。 它需要一个可选的消息来显示错误，否则使用默认消息。 `404`表示“未找到”，`403`表示“禁止”。 （`401`表示“未授权”，但您重定向到登录页面而不是返回该状态。）

定义了`check_author`参数，以便该函数可用于获取`post`而无需检查作者。 如果您编写了一个视图来显示页面上的单个帖子，用户无关紧要，因为他们没有修改帖子，这将非常有用。


```python
# flaskr/blog.py
@bp.route('/<int:id>/update', methods=('GET', 'POST'))
@login_required
def update(id):
    post = get_post(id)

    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        if error is not None:
            flash(error)
        else:
            db = get_db()
            db.execute(
                'UPDATE post SET title = ?, body = ?'
                ' WHERE id = ?',
                (title, body, id)
            )
            db.commit()
            return redirect(url_for('blog.index'))

    return render_template('blog/update.html', post=post)
```

与您到目前为止所编写的视图不同，`update`函数采用参数`id`。 这对应于路径中的`<int:id>`。 真正的URL看起来像`/1/update`。 Flask将捕获`1`，确保它是一个`int`，并将其作为`id`参数传递。 如果你没有指定`int:`而是做`<id>`，它将是一个字符串。 要生成更新页面的URL，需要传递`url_for()`，以便知道要填写的内容：`url_for('blog.update', id = post ['id'])`。 这也在上面的`index.html`文件中。

`create`和`update`视图看起来非常相似。 主要区别在于`update`视图使用`post`对象和`UPDATE`查询而不是`INSERT`。 通过一些巧妙的重构，您可以为这两个操作使用一个视图和模板，但是对于本教程，将它们分开是更清楚的。


```python
# flaskr/templates/blog/update.html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Edit "{{ post['title'] }}"{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="title">Title</label>
    <input name="title" id="title"
      value="{{ request.form['title'] or post['title'] }}" required>
    <label for="body">Body</label>
    <textarea name="body" id="body">{{ request.form['body'] or post['body'] }}</textarea>
    <input type="submit" value="Save">
  </form>
  <hr>
  <form action="{{ url_for('blog.delete', id=post['id']) }}" method="post">
    <input class="danger" type="submit" value="Delete" onclick="return confirm('Are you sure?');">
  </form>
{% endblock %}
```

此模板有两种形式。 第一个将编辑的数据发布到当前页面（`/<id>/update`）。 另一个表单只包含一个按钮，并指定一个发布到删除视图的`action`属性。 该按钮使用一些JavaScript在提交之前显示确认对话框。

模式`{{request.form ['title'] 或 post ['title']}}`用于选择表单中显示的数据。 当表单尚未提交时，将显示原始`post`数据，但如果发布了无效表单数据，则您希望显示该表单，以便用户可以修复错误，因此请使用`request.form`。 `request`是另一个在模板中自动提供的变量。

### Delete
delete视图没有自己的模板，删除按钮是`update.html`的一部分，并发布到`/<id>/delete` URL。 由于没有模板，它只会处理POST方法，然后重定向到`index`视图。


```python
# flaskr/blog.py¶
@bp.route('/<int:id>/delete', methods=('POST',))
@login_required
def delete(id):
    get_post(id)
    db = get_db()
    db.execute('DELETE FROM post WHERE id = ?', (id,))
    db.commit()
    return redirect(url_for('blog.index'))
```

恭喜，您现在已经完成了申请！ 花一些时间在浏览器中试用一切。 但是，在项目完成之前还有更多工作要做。

## Make the Project Installable
使项目可安装意味着您可以构建分发文件并将其安装在另一个环境中，就像在项目环境中安装Flask一样。 这使得部署项目与安装任何其他库相同，因此您使用所有标准Python工具来管理所有内容。

安装还带来了其他一些好处，这些好处可能在教程中或作为新的Python用户不明显，包括：

- 目前，Python和Flask只了解如何使用flaskr包，因为您正在从项目的目录中运行。 安装意味着无论您从何处运行，都可以导入它。
- 您可以像管理其他软件包一样管理项目的依赖项，因此请安装yourproject.whl安装它们。
- 测试工具可以将您的测试环境与开发环境隔离开来。

> 注意:
> 这是在本教程后期介绍的，但是在您将来的项目中，您应该始终从这开始。

### Describe the Project
`setup.py`文件描述了您的项目以及属于它的文件。


```python
# setup.py
from setuptools import find_packages, setup

setup(
    name='flaskr',
    version='1.0.0',
    packages=find_packages(),
    include_package_data=True,
    zip_safe=False,
    install_requires=[
        'flask',
    ],
)
```

`package`告诉Python要包含哪些包目录（以及它们包含的Python文件）。 `find_packages()`自动查找这些目录，因此您不必键入它们。 要包含其他文件，例如`static`和`templates`目录，请设置`include_package_data`。 Python需要另一个名为`MANIFEST.in`的文件来告诉其他数据是什么。


```python
# MANIFEST.in
include flaskr/schema.sql
graft flaskr/static
graft flaskr/templates
global-exclude *.pyc
```

这告诉Python复制`static`和`template`目录以及`schema.sql`文件中的所有内容，但要排除所有字节码文件。

有关所用文件和选项的其他说明，请参阅官方包装指南。


### Install the Project
使用pip在虚拟环境中安装项目。

pip install -e。
这告诉pip在当前目录中找到setup.py并将其安装在可编辑或开发模式下。 可编辑模式意味着当您对本地代码进行更改时，如果更改有关项目的元数据（例如其依赖项），则只需重新安装。

您可以观察到项目现在已安装了pip list。
```
pip list

Package        Version   Location
-------------- --------- ----------------------------------
click          6.7
Flask          1.0
flaskr         1.0.0     /home/user/Projects/flask-tutorial
itsdangerous   0.24
Jinja2         2.10
MarkupSafe     1.0
pip            9.0.3
setuptools     39.0.1
Werkzeug       0.14.1
wheel          0.30.0
```

到目前为止，您运行项目的方式没有任何变化。 `FLASK_APP`仍然设置为`flaskr`并且`flask run`仍然运行应用程序。


```python

```
