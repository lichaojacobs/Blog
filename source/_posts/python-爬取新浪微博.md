---
title: python 爬取新浪微博
date: 2016-12-01 01:23:15
tags:
	- Python
	- Crawler
---


最近因为课设的要求，开始了对新浪微博数据的爬取研究，看了不少博客文章，也试了不少方法，原理无非就是模拟登录，但是感觉目前可用的方法太过分散，而且自从微博改版之后，很多以前适用的方法都基本没有用处了。这里总结一下几种可用的方法以及自己研究之后稳定可用的方法(所有的方法都是基于python2.7)：


----------


###1、绕过.com域名

如果没有爬取主站的刚需，只是对微博相关的数据感兴趣，可以尝试爬取微博cn域名下的内容(即http://weibo.cn)，亲测可用...最简单的办法就是先预先登录一下然后获取返回的cookie，贴入代码中作为请求的headers即可。

```
_header={
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.86 Safari/537.36",
        "Cookie":"_T_WM=03e77f532a8c1a437da863b36a62207d; SUB=_2A256KfecDeRxGeVP61MX9yzKyT-IHXVZ1ZnUrDV6PUNbvtANLRTVkW1LHesQJOUc8nbbLnoALvjmulMBSwDnAw..; SUBP=0033WrSXqPxfM725Ws9jqgMF55529P9D9WhPUuFTXg4zll8rx_8Ap-XA5JpX5KMhUgL.Foepeh2cS0zceoet; SUHB=0cSXC9tcKk2RM7; SSOLoginState=1462601676; gsid_CTandWM=4uTtCpOz5hhWcws1tVSIdd0SYa3"    }
 request = urllib2.Request(url=url, headers=self._header)
 response = urllib2.urlopen(request)
 html = response.read()
```

 
接下来对爬取下来的html就可以通过xpath,或者bs来完成数据提取了。


----------


###2、使用urllib模拟登录微博.com主站


这个过程比较麻烦，前人有了很多铺垫做相应的改动直接拿来用就好啦，以下代码亲测可用：

```
-*- coding: utf-8 -*-import urllib2
import re
import rsa
import cookielib  #从前的cookielibimport base64
import json
import urllib
import binascii
from lxml import etree
import json
 用于模拟登陆新浪微博class launcher():
 
    cookieContainer=None    _headers={
            "User-Agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36"        }
    def __init__(self,username, password):
        self.password = password
        self.username = username


    def get_prelogin_args(self):
        json_pattern = re.compile('\((.*)\)')
        url = 'http://login.sina.com.cn/sso/prelogin.php?entry=weibo&callback=sinaSSOController.preloginCallBack&su=&' + self.get_encrypted_name() + '&rsakt=mod&checkpin=1&client=ssologin.js(v1.4.18)'        try:
            request = urllib2.Request(url)
            response = urllib2.urlopen(request)
            raw_data = response.read().decode('utf-8')
            print "get_prelogin_args"+raw_data;
            json_data = json_pattern.search(raw_data).group(1)
            data = json.loads(json_data)
            return data
        except urllib2.HTTPError as e:
            print("%d"%e.code)
            return None    def get_encrypted_pw(self,data):
        rsa_e = 65537 #0x10001        pw_string = str(data['servertime']) + '\t' + str(data['nonce']) + '\n' + str(self.password)
        key = rsa.PublicKey(int(data['pubkey'],16),rsa_e)
        pw_encypted = rsa.encrypt(pw_string.encode('utf-8'), key)
        self.password = ''   #清空password        passwd = binascii.b2a_hex(pw_encypted)
        print(passwd)
        return passwd


    def get_encrypted_name(self):
        username_urllike   = urllib.quote(self.username)
        byteStr=bytes(username_urllike)
        byteStrEncod=byteStr.encode(encoding="utf-8")
        username_encrypted = base64.b64encode(byteStrEncod)
        return username_encrypted.decode('utf-8')


    def enableCookies(self):
            #建立一个cookies 容器            self.cookieContainer = cookielib.MozillaCookieJar("/Users/lichao/desktop/weibo/cookie/cookie.txt");
            # ckjar=cookielib.MozillaCookieJar("/Users/Apple/Desktop/cookie.txt")            #将一个cookies容器和一个HTTP的cookie的处理器绑定            cookie_support = urllib2.HTTPCookieProcessor(self.cookieContainer)
            #创建一个opener,设置一个handler用于处理http的url打开            opener = urllib2.build_opener(cookie_support, urllib2.HTTPHandler)
            #安装opener，此后调用urlopen()时会使用安装过的opener对象            # proxy_handler = urllib2.ProxyHandler({"http": 'http://localhost:5000'})            # opener=urllib2.build_opener(proxy_handler)            urllib2.install_opener(opener)


    def build_post_data(self,raw):
        post_data = {
            "entry":"weibo",
            "gateway":"1",
            "from":"",
            "savestate":"7",
            "useticket":"1",
            "pagerefer":"Sina Visitor System",
            "vsnf":"1",
            "su":self.get_encrypted_name(),
            "service":"miniblog",
            "servertime":raw['servertime'],
            "nonce":raw['nonce'],
            "pwencode":"rsa2",
            "rsakv":raw['rsakv'],
            "sp":self.get_encrypted_pw(raw),
            "sr":"1280*800",
            "encoding":"UTF-8",
            "prelt":"77",
            "url":"http://weibo.com/ajaxlogin.php?framelogin=1&callback=parent.sinaSSOController.feedBackUrlCallBack",
            "returntype":"META"        }
        data = urllib.urlencode(post_data).encode('utf-8')
        return data


    def login(self):
        url = '新浪通行证'        self.enableCookies()
        data = self.get_prelogin_args()
        post_data = self.build_post_data(data)
        headers = {
            "User-Agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36"        }
        try:
            request = urllib2.Request(url=url,data=post_data,headers=headers)
            response = urllib2.urlopen(request)
            html = response.read().decode('GBK')
            #print(html)        except urllib2.HTTPError as e:
            print(e.code)


        p = re.compile('location\.replace\(\'(.*?)\'\)')
        p2 = re.compile(r'"userdomain":"(.*?)"')


        try:
            login_url = p.search(html).group(1)
            print(login_url)
            request = urllib2.Request(login_url)
            response = urllib2.urlopen(request)
            page = response.read().decode('utf-8')
            print(page)
            login_url = 'http://weibo.com/' + p2.search(page).group(1)
            request = urllib2.Request(login_url)
            response = urllib2.urlopen(request)
            final = response.read().decode('utf-8')


            print("Login success!")
            self.cookieContainer.save(ignore_discard=True, ignore_expires=True)
        except Exception, e:
            print('Login error!')
            print e
            return 0
```

###3、使用selenium实现模拟登录
- selenium +phantomjs

第二种方法有一个问题，因为目前新版的微博页面的渲染方式采用的是分片渲染的，这就导致我们通过第二种静态方式爬取到的页面并不是最终的页面，而是内容嵌在 js里的中间页面，这肯定不是我们想看到的结果。于是，考虑模拟浏览器渲染页面的方式获取到最终的呈现页面。selenium这个工具正好完美的解决了我们的问题，它可以模拟浏览器的行为，并且我们拿到的source可以向jquery操作dom对象那样查找定位元素，非常方便，实现的核心代码如下：

```
import time
from selenium import webdriver
import urllib2
import selenium.webdriver.support.ui as ui
import sys
reload(sys)
sys.setdefaultencoding( "utf-8" )
from selenium.webdriver.common.keys import Keys
Chrome PhantomJS#driver = webdriver.PhantomJS("/Users/test/documents/phantomjs/bin/phantomjs")
driver.get('http://weibo.com/')


try:
    print "登录开始"
    username = driver.find_element_by_xpath('//input[@name="username"]')
    password = driver.find_element_by_xpath('//input[@name="password"]')
    sbtn = driver.find_element_by_xpath('//a[@action-type="btn_submit"]')
    username.send_keys('') #send username       
    password.send_keys('') #send password    sbtn.click()  
    # 提交表单    
    time.sleep(3)  # 等待页面加载   
    # get the session cookie    
    cookie = {item["name"] + ":" + item["value"] for item in driver.get_cookies()}    cookie=driver.get_cookies()
    for item in driver.get_cookies():    cookieItem={"name":item["name"],"value":item["value"],"domain":item["domain"],"httponly":item["httponly"],"path":item["path"],"secure":item["secure"]}    cookie.append(cookieItem)    cookie_file= open("/Users/test/desktop/weibo/cookie/cookie.txt",'w')    cookie_file.write(str(cookie))    print str(str(cookie))
except urllib2.HTTPError as e:
    print e
    print "登录失败"print "开始爬取谣言大厅"driver.get("http://service.account.weibo.com/show?rid=K1CaN7gJl8q8f")
page = driver.page_source
print page
driver.quit()
```

我们将登录之后获取的cookie以键值对的形式存入文本文件中，方便下次直接load而不需要重复登录:

```
def loadCookie(self):
self._driver.get("http://www.sina.com.cn")
    cookie_file=open("/Users/test/desktop/weibo/cookie/cookie.txt",'r')
    cookieStr=cookie_file.read();
print "cookie is: "+cookieStr
    cookieList=list(eval(cookieStr))
for item in cookieList:
cookieDic= type(eval(item))
        self._driver.add_cookie(item)
```

- selenium +chromedirver

使用phantomjs存在一个问题，登录过程老是失败，因为验证码无法识别获取导致登录经常失败，这里我们使用chromedirver这工具结合selenium实现开挂级别的python数据爬取，模拟登录万无一失，核心代码如下：

```
print "登录开始"
username = driver.find_element_by_xpath('//input[@name="username"]')
password = driver.find_element_by_xpath('//input[@name="password"]')
sbtn = driver.find_element_by_xpath('//a[@action-type="btn_submit"]')
veryfiCode=driver.find_element_by_xpath('//input[@name="verifycode"]')
```

 程序启动时会自动开启一个chrome窗口，只不过这个浏览器的行为我们可以通过程序控制，这样是不是方便多了！我们在username这一行打一个断点，然后程序执行到这一步，在浏览器中输入相应的用户名，密码，验证码，然后在pycharm中点击继续，登录成功！真实浏览器结合程序，真是开挂级别的爬取微博啊...