title: "Ubuntu14下搭建Nginx+PHP+Moodle简单指南"
date: 2014-08-14 19:48:09
tags: [moodle,php,nginx]
---

Moodle是一个开源课程管理系统（CMS），也被称为学习管理系统（LMS）或虚拟学习环境（VLE）。它已成为深受世界各地教育工作者喜爱的一种为学生建立网上动态网站的工具。相信在教育技术不断信息化的今天，moodle的应用范围将越来越广泛，今天就将快速配置moodle的心得记录下来。

<!-- more -->

# 懒人安装模式

基于阿里云`Ubuntu 14.04`系统  

更新源
```bash
	sudo apt-get update
	sudo apt-get upgrade
```
安装`nginx`  `php5-fpm`及扩展
```bash
	sudo apt-get install nginx php5-fpm
	sudo apt-get install php5-curl
	sudo apt-get install php5-mysql
```
配置`nginx`
```bash
	sudo vi /etc/nginx/sites-available/default
```
修改为以下内容
```bash
	root /www/php; //配置根目录
```
及
```bash
	location ~ \.php$ {
		fastcgi_pass unix:/var/run/php5-fpm.sock;  
		fastcgi_index index.php;  
		include fastcgi_params;  
	}  
```
开启 或 重载`nginx`
```bash
	service nginx start
	service nginx reload
```
安装`mysql`
```bash
	sudo apt-get install mysql-server
```
安装`moodle`
```bash
	cd /www/php
	wget download.moodle.org/download.php/direct/stable27/moodle-latest-27.tgz
	tar -zxvf moodle-latest-27.tgz
	mkdir moodle
```
接下来就可以通过公网ip访问配置`moodle`

### 问题1
>Parent directory (/www/php) is not writeable. Data directory (/www/php/moodledata) cannot be created by the installer.  

[社区回答](http://stackoverflow.com/questions/18749927/error-installing-moodle-dataroot-location-is-not-secure-and-parent-)  
	
	sudo chown -R www-data:www-data /www/php/moodle
	sudo chown -R www-data:www-data /www/php/moodledata


### 问题2
>Moodle会尝试将配置存储在您的Moodle根目录中。安装脚本无法自动创建一个包含您设置的config.php文件，极可能是由于Moodle>目录是不能写的。您可以复制如下的代码到Moodle根目录下的config.php文件中。

```bash
	vi /www/php/mododle/config.php
```
复制
```php
	<?php  // Moodle configuration file

	unset($CFG);
	global $CFG;
	$CFG = new stdClass();

	$CFG->dbtype    = 'mysqli';
	$CFG->dblibrary = 'native';
	$CFG->dbhost    = 'localhost';
	$CFG->dbname    = 'moodle';
	$CFG->dbuser    = '<name>';
	$CFG->dbpass    = '<password>';
	$CFG->prefix    = 'mdl_';
	$CFG->dboptions = array (
	  'dbpersist' => 0,
	  'dbport' => 3306,
	  'dbsocket' => '',
	);
	
	$CFG->wwwroot   = 'http://<ip>';
	$CFG->dataroot  = '/www/php/moodledata';
	$CFG->admin     = 'admin';

	$CFG->directorypermissions = 0777;

	require_once(dirname(__FILE__) . '/lib/setup.php');

	// There is no php closing tag in this file,
	// it is intentional because it prevents trailing whitespace problems!
```

### 问题3
>![检查服务器](http://daysv.qiniudn.com/o_18v7tklup9441rm51joi1h5u1gp39.jpg)

空缺的扩展逐一安装
```
	sudo apt-get install php5-gd
	sudo apt-get install php5-xmlrpc 
	sudo apt-get install php5-intl
```
### 问题4
当进去后发现css样式似乎不起作用(**这设定略坑**)  
[官方doc](https://docs.moodle.org/dev/Install_Moodle_On_Ubuntu_with_Nginx/PHP-fpm)

>In the Moodle administration, disable 'slash arguments' (http://YOURMOODLESITE/admin/search.php?query=slashargument). Without disabling the 'slash arguments', you may notice that the admin setup page is missing the images and css styling. However, if you turn off slash arguments, then other things won't work, so really someone ought to work out what is wrong here and fix it.

>If you cannot access the Admin interface, you can edit the database using a tool like phpMyAdmin.

>1. Go to the table labeled 'mdl_config'

>2. Browse to line 281 (line 309 in 2.5), and change the value of 'slasharguments' from 1 to 0

>3. Use Control-F5 to fully refresh the setup page. 	


连接mysql
```sql
	mysql -u<name> -p<password>
	show databases;
	use moodle;
```
我们可以用sql语句修改下
```sql
	UPDATE mdl_config set value='0' where name='slasharguments';
```
大功告成  

