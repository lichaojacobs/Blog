---
title: superset customization
date: 2017-10-13 14:35:12
tags: 
    - superset
    - 二次开发
    - python	
---

### 前言

- 由于数据组目前重度依赖kylin，然而kylin并没有官方开源的数据可视化工具。所幸kylin提供了丰富的查询API供我们直接传入SQL进行查询，与此同时发现superset有非官方对接kylin的开源插件，虽然两年没有维护了，对代码进行了部分重构也就成功将superset和kylin对接起来了。

- 通常一个开源框架在使用过程中，总是会有各种各样的针对具体场景的定制化需求，superset自然不例外。于是下面简单记录一下superset定制化的过程中干的一些事情，算是做一个小的总结。



### python2升级python3

虽然python2.7是superset官方推荐的版本，但是由于python2.7默认的Encoding的问题导致使用过程中一旦出现中文就出错。为了避免这种情况，我选择将python2.7升级到python3.5。当然升级还算顺利，碰到的最大的坑就是python2中的MysqlDB模块在python3中已经不维护了，于是修改superset源码如下

```
#init.py
#pip install pymysql

import pymysql
pymysql.install_as_MySQLdb()

```

### superset ldap配置

- 基本上各大开源组件都支持ldap的配置，superset也不例外。ldap的好处就是给企业用户提供统一的账户授权，方便权限管理。

- 由于google上基本上查不到相关的资料或者都是配置出错了没法解决的提问，这一块踩了不少的坑。然后发现superset其实是直接用的flask-appbuilder里提供的security组件支持ldap的，通过熟悉ldap的原理以及查看flask-appbuilder的官方文档和security组件的源代码最终解决了ldap的配置问题。

- 先在config.py中引入flask-appbuilder的相关依赖

	```
	#引入引用：
	from flask_appbuilder.security.manager import AUTH_OID,
	AUTH_REMOTE_USER,
	AUTH_DB, AUTH_LDAP,
	AUTH_OAUTH,
	AUTH_OAUTH
	```

- 再在config.py AUTH一块的区域中加入如下配置

	```	
	//最终配置（这里用到二次验证，即bind给定账户，然后根据权限去搜索输入的用户，拉	取用户的dn，最后用得到的密码和拉取到的dn去bind_user即可）
	#2表示使用ldap
	AUTH_TYPE = 2 
	#自己公司的ldap服务器
	AUTH_LDAP_SERVER = "ldap://ldap.example.com" 
	#ldap有两种认证方式，这里用到二次验证，需要配置一个admin账号
	AUTH_LDAP_BIND_USER = "cn=admin,dc=ldap,dc=mobvoi,dc=com"
	#admin账号的密码
	AUTH_LDAP_BIND_PASSWORD = "xxxx"
	AUTH_LDAP_USE_TLS = False
	#在哪个节点查找用户
	AUTH_LDAP_SEARCH = "ou=users,dc=ldap,dc=mobvoi,dc=com"
	#查找用户的字段
	AUTH_LDAP_UID_FIELD = "uid"
	#是否允许superset数据库中没有记录的ldap用户自动注册
	AUTH_USER_REGISTRATION = True
	#注册新用户时默认分配的权限
	AUTH_USER_REGISTRATION_ROLE = "Dashboard Readonly"
	
	```
	
	
### superset docker化

为了减轻运维服务器升级时服务的迁移成本，决定将统一修改的源码托管起来，每一次修改重新发布docker，这样能大大降低运维成本。

- 拉取ubuntu镜像，挂载本地磁盘，并进入后台运行模式

	```
	sudo docker run -it -v /home/:/home/ leemiracle/unbuntu /bin/bash
	
	```
	
- 在ubuntu中安装python3.5

	```
	wget https://www.python.org/ftp/python/3.5.4/Python-3.5.4rc1.tgz
	
	```
	
	
- 拖下superset、pylin源代码，安装pyklin 与ldap

	```
	git clonet git@github.com:lichaojacobs/superset.git
	git clone git@github.com:lichaojacobs/pykylin.git
	cd pykylin
	pip install -r ./requirements.txt
   python setup.py install
   pip install pyldap

	```
- 修改flask-appbuilder代码（重写了权限部分逻辑）

	```
	flask-appbuilder/security/manager.py  auth_user_ldap (method)
	try:
     user_db = self.auth_user_db(username,password)
     if user_db!=None:
        return user_db
  	except Exception:
     log.info("using ldap and first try db_auth failed")
	
	```

- 修改superset源码（增加status接口，主要是对服务状态进行监测）

	```
	
	from flask import jsonify
  class MyIndexView(IndexView):
    @expose('/')
    def index(self):
        return redirect('/superset/welcome')
    @expose('/status')
    def status(self):
        return jsonify(status='ok')
	
	```
	
- 修改superset config.py，加入缓存配置，ldap配置等

	```
	加上缓存：
  CACHE_DEFAULT_TIMEOUT = 7200
  CACHE_CONFIG = {
    'CACHE_TYPE':'redis',
    'CACHE_DEFAULT_TIMEOUT':7200,
    'CACHE_KEY_PREFIX':'superset_',
    'CACHE_REDIS_HOST':'xxxxxx.aliyuncs.com',
    'CACHE_REDIS_PORT':6379,
    'CACHE_REDIS_DB':'',
    'CACHE_REDIS_PASSWORD':'xxxxx'
  }
	
	```
	
- 加上GA脚本用于统计页面PV UV

	```
	<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');
	
  {% if g.user.get_full_name %}
  ga('set', 'userId', '{{g.user.get_full_name()}}');
  {% endif %}
  ga('create', 'UA-64695573-19', 'auto');
  ga('send', 'pageview');

  </script>
	
	```
	
- 自编译superset

	```
	cd $SUPERSET_HOME/superset/assets
	./js_build.sh
	#由于实际编译过程中出现了各种测试异常，我将npm run test,npm run cover去掉了
		
	```
	
- 安装superset

	```
	apt-get install build-essential libssl-dev libffi-dev python-dev python-pip libsasl2-dev libldap2-dev
   apt-get install python3.5-dev
	pip install --upgrade setuptools pip
	#直接运行上一步编译好的文件
	python superset/setup.py install
	
	```
	
- 打包镜像，重新commit修改，并打上tag

	```
	sudo docker commit -m "superset docker init" -a "author" <container id> <docker repository host>/superset:<version>
	
	```
- 运行docker

	```
	sudo docker run -m 1G --net=host <docker repository host>/superset:<version> superset runserver -p 8088 -t 500
	
	```
	
### 总结

虽然目前步子迈得有点小，但好在还在进步。在自己努力下，从之前痛苦的数据架构升级到现在，数据组的基础架构平台也算是稳定的在提供服务了。kylin也在啃了很久的源代码，做了一些优化之后从之前每天要挂三四次，数据主从同步问题每次执行任务都要重现到现在也能平稳运行。路还很长，想往底层钻的深一点，继续加油吧～