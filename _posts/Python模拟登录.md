---
title: Python模拟登录
date: 2016-05-06 16:22:16
categories: Python
tags: Python
---

Python模拟登录，刚开始使用urllib和cookiejar模块。后来还是发现requests好用。
<!---more-->
#### 以登录DVWA为例：
登录过程如下：  
首先get请求一次login.php  
网页返回cookie和token  
输入用户名和密码，post将用户名和密码还有cookie值与token值一起发送  
服务器接受数据，进行判断，成功跳转到index.php否则跳转到login.php


#### 使用Python实现如下：

首先定义header和loginurl

1、创建请求会话  
`session=requests.session()
`  
2、解析网页得到token
```
def get_token():
    index_page=session.get(loginurl,headers=get_headers())
    html=index_page.text
    ma=re.compile("['']\\w{32}[']")
    pa=ma.search(html)
    token=pa.group()
    token=token[1:-1]
    return token
```
3、要发送的数据
```
def get_values():
    values={
    'username':'admin',
    'password':'password',
    'Login':'Login',
    'user_token':get_token(),
    }
    return values

```

4、登录
```
def login():
    login_page=session.post(loginurl,headers=get_headers(),data=get_values())
    html_respon=login_page.text
    ma=re.compile("You have logged in as \'\w*\'")
    pa=ma.search(html_respon)
    try:
        msg=pa.group()
        print(msg)
    except Exception as e:
        print('登录失败')

```


#### 验证码部分
如果有验证码，可以使用pytesseract模块进行识别  
安装pytesseract需要pillow和tesseract-ocr  
在ubuntu下安装:  
```
sudo apt-get install python3-pil tesseract-ocr    
sudo pip3 install pytesseract
```

识别代码如下
```
captcha_url='http://jwzx.cqupt.edu.cn/createValidationCode.php'
  r=session.get(loginurl)
  r=session.get(captcha_url,headers=get_headers())
  with open('captcha.png','wb') as f:
      f.write(r.content)
      f.close()
  image = Image.open('captcha.png')
  vcode = pytesseract.image_to_string(image)
  print(vcode)
```

试了一下学校的教务在线，识别率一般。
