---
title: 基础设施邮件列表禁止注册
date: 2021-04-022
author: liuqi
categories:
- mailman
tags:
- mailman
---
# 背景
为响应护网行动，以及考虑到海外交付，现需要禁止社区的邮件列表注册功能，禁止新用户注册账号，同时禁用第三方登录。
# 禁止新用户注册账号
邮件列表的注册功能由allauth提供，首先考虑是否开启/关闭注册功能的配置能够直接禁止注册。很遗憾，目前allauth暂无这样的开关。在allauth的[高级用法](https://django-allauth.readthedocs.io/en/latest/advanced.html)中，我们可以看到`allauth.account.adapter.DefaulteAccountAdapter`类中有一个`is_open_for_signup`的方法，默认返回True，如果要禁用帐户注册，可以通过返回False来覆盖此方法。可以新建一个类继承`allauth.account.adapter.DefaulteAccountAdapter`，重写`is_open_for_signup`方法，并在`postorius`的项目`settings.py`中通过设置ACCOUNT_ADAPTER指向这个类。因为通过在`mailman-web-configmap`中重写`settings.py`会因为STATIC_URL='/static'，STATIC_URL未已`/`结束而引出一个ImproperlyConfigued的错误，而将STATIC_URL改为'/static/'后需要做更多修改，所以通过下面几个步骤来实现禁止注册与屏蔽第三方登录：

- 注释`postorius/templates/postorius/base.html`和`hyperkitty/templates/hyperkitty/base.html`中注册的标签，并将重写的base.html放入`mailman-web-configmap`中，以在页面上隐藏注册。
- 在`mailman-nginx-configmap`中`default.conf`的`server`内添加一个`location`，屏蔽注册路由
```
	server {
		listen 80 default_server;
		
		root /opt/mailman-web-data;
		index index.html;
		
		location /static {
			alias /opt/mailman-web-data/static;
		}

		location / {
			uwsgi_pass 0.0.0.0:8080;
			include uwsgi_params;
			uwsgi_read_timeout 300;
		}

		location /accounts/signup {
			return 403;
		}
	}
```
# 禁用第三方登录
在`mailman-web-configmap`的`settings_local.py`中，重写`settings.py`中的INSTALLED_APPS，去掉最后5项`allauth.socialaccount`相关的application，如下
```
  INSTALLED_APPS = [
	'hyperkitty',
	'postorius',
	'django_mailman3',
	'django.contrib.admin',
	'django.contrib.auth',
	'django.contrib.contenttypes',
	'django.contrib.sessions',
	'django.contrib.sites',
	'django.contrib.messages',
	'django.contrib.staticfiles',
	'rest_framework',
	'django_gravatar',
	'compressor',
	'haystack',
	'django_extensions',
	'django_q',
	'allauth',
	'allauth.account',
	'allauth.socialaccount',
  ]
```
# 后记
如果您有任何问题和改进建议，欢迎您随时联系[github issue](https://github.com/opensourceways/infra-landscape/issues)
