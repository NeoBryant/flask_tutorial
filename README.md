# flask_tutorial -- flaskr 博客

[源代码](https://github.com/pallets/flask/tree/1.0.2/examples/tutorial)


## run app
```bash

bash:
for mac/linux:
    # 初始化数据库
    export FLASK_APP=flaskr
    export FLASK_ENV=development
    flask init-db
    
    # 运行服务器
    flask run

    visit:  
        http://127.0.0.1:5000/hello 

    visit:  
        http://127.0.0.1:5000/auth/register # 注册
        http://127.0.0.1:5000/auth/login # 登录

    flask run
    flaskr

    pip install -e .
```

## tree of app
```bash
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
