# NySpiderNotes
##Python网络爬虫教程
###1.抓取&解析
#####1.1 获取页面
######html页面
使用requests库,详细用法参考文档<a href="http://cn.python-requests.org/zh_CN/latest/">requests的文档</a>
这里以IT桔子为例，获取投资页面

```python
import requests

headers = {
    'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:39.0) Gecko/20100101 Firefox/39.0',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.5',
    'Accept-Encoding': 'gzip, deflate',
    'Connection': 'keep-alive'}

html=requests.get('https://www.itjuzi.com/investevents',headers=headers).text
```
html就是获取到的网页源码

######Ajax的处理
对于Ajax页面，从网页的url加载网页的源代码之后，会在浏览器里执行JavaScript程序。这些程序会加载更多的内容，“填充”到网页里。这就是为什么如果你直接去爬网页本身的url，你会找不到页面的实际内容。
处理这种情况，可以通过抓包找到Ajax请求的url,直接获取这个url里面的内容即可。一般Ajax加载的数据都是json数据，可以通过python的json模块转成python的数据结构来解析。
#####1.2 解析
解析html页面可以用 bs4 lxml 或者正则表达式，就我个人而言，用bs4用得比较多。
详细的用法可以查看文档
<a href="http://beautifulsoup.readthedocs.org/zh_CN/latest/">BeautifulSoup的文档</a>
```python
from bs4 import BeautifulSoup

def parser(html):
		result=[]
        soup=BeautifulSoup(html,'html.parser')
		table=soup.find_all('ul',attrs={'class':'list-main-eventset'})[1].find_all('li')
        for li in table:
            item={}
            i=li.find_all('i')
            item['date']=i[0].get_text()
            item['url']=i[1].find('a').get('href')
            spans=i[2].find_all('span')
            item['name']=spans[0].get_text()
            item['industry']=spans[1].get_text()
            item['local']=spans[2].get_text()
            item['round']=i[3].get_text()
            item['capital']=i[4].get_text()
            companys=i[5].find_all('a')
            lists=[]
            if(companys==[]):
                lists.append(i[5].get_text())
            else:
                for a in companys:
                    lists.append(a.get_text())
            item['Investmenters']=lists
            result.append(item)
		return result
```
解析结果：
![image](http://7xsjsi.com2.z0.glb.clouddn.com/itjuzi.png)

###2.登录&验证码识别
#####2.1登录
######1.表单登录
```python
import requests

data={
	'user':'xxxx',
	'passwd':'******'
}
session=requests.session()
session.post(url,data=data)
```
上面是最基本的表单登录，实际中，先在浏览器中手动登录网站，用抓包工具截获表单post的数据，再模拟浏览器登录即可

######2.cookie登录
使用cookie登录比较简单，首先需要手动登陆网站一次，获取服务器返回的cookie，这里就带有了用户的登陆信息，再将cookie填入程序接就可以实现登录了。使用cookie登录有时效性问题，但是可以避免做验证码识别。
```python
import requests

cookies=''#填入获取到的cookies
session=resuests.session()
session.get(url,cookies=cookies)#这样,session就是登录状态的了

```

#####2.2 验证码识别
爬取网页数据的过程中，经常会遇到需要输入验证码情况，比如登录验证码，爬取过多数据的时候需要验证码，接下来接讲一讲怎么做验证码识别
######1.获取验证码
以[华中科技大学选课网](http://wsxk.hust.edu.cn/)为例，验证码是简单的数字验证码
![image](http://wsxk.hust.edu.cn/randomImage.action)
通过抓包分析，验证码是从http://wsxk.hust.edu.cn/randomImage.action 获取的
编写爬虫获取一定数量的验证码
```python
def imagesget():
    os.mkdir('images')
    count=0
    while True:
        img=requests.get('http://wsxk.hust.edu.cn/randomImage.action').content
        with open('images/%s.jpeg'%count,'wb') as imgfile:
            imgfile.write(img)
        count+=1
        if(count==100):
            break
```

######2.处理验证码(二值化，去噪点)
验证码数字都是深色的，因此可以将深色全部转化为黑色，浅色转化为白色，这样我们就可以得到一张没有噪点的黑白图片
```python
def convert_image(imageName):
    image=Image.open(imageName)
    image=image.convert('L')
    image2=Image.new('L',image.size,255)
    for x in range(image.size[0]):
        for y in range(image.size[1]):
            pix=image.getpixel((x,y))
            if pix<120:
                image2.putpixel((x,y),0)
    return image2
```
转化后的验证码如下
![image](http://7xsjsi.com2.z0.glb.clouddn.com/image.jpeg)
######2.切割图片
验证码中各个字符间没有粘连，因此可以直接从空白处切割

```python
def cut_image(image):
    inletter=False
    foundletter=False
    letters=[]
    start=0
    end=0
    for x in range(image.size[0]):
        for y in range(image.size[1]):
            pix=image.getpixel((x,y))
            if(pix==0):
                inletter=True
        if foundletter==False and inletter ==True:
            foundletter=True
            start=x
        if foundletter==True and inletter==False:
            end=x
            letters.append((start,end))
            foundletter=False
        inletter=False
    images=[]
    for letter in letters:
        img=image.crop((letter[0],0,letter[1],image.size[1]))
        images.append(img)
    return images
```

######3.AI 与向量空间图像识别

在这里我们使用向量空间搜索引擎来做字符识别，它具有很多优点：

    不需要大量的训练迭代
    不会训练过度
    你可以随时加入／移除错误的数据查看效果
    很容易理解和编写成代码
    提供分级结果，你可以查看最接近的多个匹配
    对于无法识别的东西只要加入到搜索引擎中，马上就能识别了。

当然它也有缺点，例如分类的速度比神经网络慢很多，它不能找到自己的方法解决问题等等。

关于向量空间搜索引擎的原理可以参考这篇文章：http://la2600.org/talks/files/20040102/Vector_Space_Search_Engine_Theory.pdf

向量空间搜索引擎名字听上去很高大上其实原理很简单。拿文章里的例子来说：

有3篇文档，筛选出其中的单词作为特征，对应单词的数量作为特征的值。取n个单词就生成一个n维空间，而每一篇文档就是在这个空间中的矢量，我们只要计算矢量之间的角度就能得到文章的相似度了。

######4.处理获取到的验证码，得到数据集
将获取到的验证码分割，保存，将分割后的数字图片分类

![image](http://7xsjsi.com2.z0.glb.clouddn.com/2016-04-08.png)

######用 Python 类实现向量空间：
```python
class CaptchaRecognize:
    def __init__(self):
        self.letters=['0','1','2','3','4','5','6','7','8','9']
        self.loadSet()

    def loadSet(self):
        self.imgset=[]
        for letter in self.letters:
            temp=[]
            for img in os.listdir('./icon/%s'%(letter)):
                temp.append(buildvector(Image.open('./icon/%s/%s'%(letter,img))))
            self.imgset.append({letter:temp})

    #计算矢量大小
    def magnitude(self,concordance):
        total = 0
        for word,count in concordance.items():
            total += count ** 2
        return math.sqrt(total)

    #计算矢量之间的 cos 值
    def relation(self,concordance1, concordance2):
        relevance = 0
        topvalue = 0
        for word, count in concordance1.items():
            if word in concordance2:
                topvalue += count * concordance2[word]
        return topvalue / (self.magnitude(concordance1) * self.magnitude(concordance2))
```
######5.识别图片
```python
def recognise(self,image):
        image=convert_image(image)
        image.save('image.jpeg')
        images=cut_image(image)
        vectors=[]
        for img in images:
            vectors.append(buildvector(img))
        result=[]
        for vector in vectors:
            guess=[]
            for image in self.imgset:
                for letter,temp in image.items():
                    relevance=0
                    num=0
                    for img in temp:
                        relevance+=self.relation(vector,img)
                        num+=1
                    relevance=relevance/num
                    guess.append((relevance,letter))
            guess.sort(reverse=True)
            result.append(guess[0])
        return result
```
识别结果：
![image](http://7xsjsi.com2.z0.glb.clouddn.com/2016-04-08re.png)
完整代码可以查看 https://github.com/Nyloner/NyPython/tree/master/CaptchaRecognise



