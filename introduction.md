# 研究生招生数据爬取及查询功能介绍 #
## 一、研究生招生数据的爬取 ##
### （一）爬取的内容 ###
* 研究生招生的院系，专业的名称
* 各个院系的介绍中的所有类别，例如：概况，培养特色，导师团队的具体内容，初试范围链接
* 各个专业复试范围以及专业简介的链接
### （二） 代码注解 ###
【PS】定义以类来融合和执行所有操作，所以后面粘贴过来的代码方法块最前面都有4个缩进空格
* 获取所有院系的名称(departments)，编号(numberlist)及其**对应专业**的链接(linkist)来供后面形成字典和继续爬取
```python
    def __init__(self):
        self.departments = self.climb("https://yjszs.ecnu.edu.cn/system/sszszyml_list.asp", text=re.compile(r"\(\d{3}\).*"))
        self.numberlist = []
        self.linklist = []
        for department in self.departments:
            self.numberlist.append(department[1:4])
            s = self.getHTMLText("https://yjszs.ecnu.edu.cn/system/sszszyml_list.asp")
            soup = BeautifulSoup(s, "lxml")
            links=soup.find("a", {"href": re.compile(r"sszyml_list.asp\?zydm=.{{6}}&zsnd=2020&yxdm={}.*".format(int(department[1:4])))})
            self.linklist.append(links.attrs["href"])
```
* 获取各个院系的概况，培养特色，导师团队的具体内容并形成列表
```python
    def getintroduction(self):
        condition = ["概况", "培养特色", "导师团队"]
        introduction_list = []
        for number in range(len(self.numberlist)):
            details=[]
            introduction = self.climb("https://yjszs.ecnu.edu.cn/system/yxjj_detail.asp?yxdm=" + self.numberlist[number] + "&zsnd=2020", attrs={"style": "  font-size:14pt; color:blue;line-height:30px;table-layout:fixed;WORD-BREAK: break-all; WORD-WRAP: break-word ;margin-top:40px;margin-left: 50px;"})
            for section in introduction:
                details.append(section.contents[1].string)
            introduction_list.append(dict(zip(condition, details)))
        return introduction_list
```
【PS】这里为了简化代码定义了一个专门用于爬取的方法，所有的爬取工作都用它来执行
```python
   def climb(self, url,text=0,attrs=0):
        s = self.getHTMLText(url)
        soup = BeautifulSoup(s, "lxml")
        if text:
            return soup.find_all(text=text)
        else:
            return soup.find_all(attrs=attrs)
```
* 通过前文提到的对应专业的linklist来继续爬取各个专业的**复试范围**和**专业详情**介绍的链接
```python
    def major(self):
        majorlist = []
        for link in range(len(self.linklist)):
            major_type = self.climb('https://yjszs.ecnu.edu.cn/system/'+self.linklist[link],text=re.compile(r"\(.{6}\).*"))
            major_information = {}
            for major in major_type:
                major_information[major] = {}
                fomular = major[1:7] + "&zsnd=2020&yxdm=" + self.numberlist[link]
                majorlink = "https://yjszs.ecnu.edu.cn/system/ssfsfw_list.asp?zydm=" + fomular
                major_information[major]["复试范围"] = majorlink
                major_introduction_link = "https://yjszs.ecnu.edu.cn/system/sszszy_detail.asp?zydm=" + fomular
                major_information[major]["详情"] = major_introduction_link
            majorlist.append(major_information)
        return majorlist
```
* 由于发现每个院系的各个专业**初试范围**相同，所以爬虫的时候就把每个院系对应了同一个初试范围
```python
    def firstscope(self):
        first_list = []
        for number in self.numberlist:
            firstscope = "https://yjszs.ecnu.edu.cn/system/sscsfw_list.asp?zsnd=2020&yxdm=" + number
            first_list.append(firstscope)
        return first_list
```
* 最后把所有获取到的信息整合生成一个字典并写入json文件
```python
    def run(self):
        firstscope = self.firstscope()
        major = self.major()
        getintroduction=self.getintroduction()
        for i in range(len(getintroduction)):
            getintroduction[i]["初试范围"] = firstscope[i]
            getintroduction[i]["对应专业"] = major[i]
        dictionall = dict(zip(self.departments, getintroduction))
        with open("data.json", "w", encoding="utf-8") as f:
            f.write(json.dumps(dictionall,ensure_ascii=False))
```
## 二、用户查询功能 ##
### （一）主要功能 ###
* 供用户查询以上所提到的所有内容
* 构建了一个类似客服机器人的体系来引导用户查询
* 对于一些用户没有按照规定格式查询的内容可以
* 对于提取到的所有信息，用户都可以分别查询，多次查询
* 用户在各个步骤中可以随时退出跳到上一个步骤
### （二）实现方式 ###
【PS】通过定义多个类来代码复用，以达到简化代码的目的，最后通过一个类的调用来实现所有的功能
* 模拟客服，构建开始和结束方式
* 在开始时给用户提供了两种选择：
1. 针对还没有专业方向的用户，直接告知他们研究生招生的所有院系，并且让他们从中决定自己想要了解的院系
2. 一种是针对已经有大致专业方向的用户，通过他们的大致方向（比如“化学”，“教育”），找到他们可能想要了解的具体院系
* 以下这个类就是程序的主类了
```python
class Hello_and_bye(common):
    
    def __init__(self):
        print(colored("欢迎来到ECNU研究生招生信息查询页面！客服小姐姐Helen将为您服务！\n", "yellow"))
        print(colored("客官一定要注意当输入规定的名称时一定要准确！并且带有前面的编号，可以选择直接复制粘贴哦~\n", "red"))
        self.q = ["客官您好，不知道想要想要了解哪方面的信息呢？", "不知客官已经对ECNU的院系心有所属了呢？", "还是想要先了解一下院系呢？", "如果已经心有所属了就直接告诉Helen想要了解的专业方向吧~"]
        self.a = self.input_delay(self.q)
        
    def _think(self,words):
        departments = []
        if "院系" in words or "了解" in words:
            self.choice=True
            self.a1=self.input_delay(["\n".join(list(alldata.keys())),"那么请问客官现在决定好要了解的院系了吗？"])
            know_department(self.a1).think()
        elif '0' <= words[1] <= '9':
            know_department(words).think()
        else:
            self.choice=False
            for item in list(alldata.keys()):
                if re.search(r".*{}".format(self.a), item):
                    departments.append(item)
            if len(departments) == 1:
                self.delay([f"我猜您想了解的是{departments[0]}"])
                know_department(departments[0]).think()
            else:
                self.a1 = self.input_delay([f"检测到您可能想对这些院系有进一步了解：{'、'.join(departments)}", "不知客官想要详细了解哪个呢？", "选择时需要准确输入哦~"])
                count=0
                while self.a1.lower()!="exit" and count<len(departments):
                    if count >= 1:
                        self.a1 = self.input_format("其他院系")
                    if self.a1!="exit":
                        know_department(self.a1).think()
                        count = 1
                    else:
                        break
    
    def run(self):
        while self.a != "exit":
            self._think(self.a)
            if self.choice==False:
                self.a = self.input_delay(["客官是否还想了解其他专业方向呢？", "如果想要继续了解就直接输入想要了解的专业方向吧~", "如果不想了解，就输入exit退出吧~"])
            else:
                self.a = self.input_delay(["客官是否还想了解其他院系呢？", "如果想要继续了解就直接输入想要了解的院系吧~", "如果不想了解，就输入exit退出吧~"])
        self.delay(["客官慢走呦，到这里，Helen要跟您说再见了，下次有问题继续找我呦，Helen随时恭候您的驾到！"])
```
* 通过上面的问题，已经获取到用户想要了解的院系了，那么对于院系的了解，根据json文件中各个院系所能了解到的信息类别告知用户（例如：概况，培养特色，导师团队，初试范围，对应专业），并让用户从中选择想要了解的内容，之后进行回复
```python
class know_department(common):

    def __init__(self, department):
        self.department = department
        self.items = list(alldata[department].keys())
        self.q = ["敢问客官想对哪个方面有所了解呢？", f"您可以从{'、'.join(self.items)}中选择哦~"]
        self.a = self.input_delay(self.q)

    def think(self):
        count1=0
        while count1 < len(self.items):
            if count1 >= 1:
                self.a = self.input_format("这个院系")
            if self.a.lower()!="exit":
                if self.a=="初试范围":
                    self.clicklink([ f"{alldata[self.department][self.a]}"])
                elif self.a == "对应专业":
                    majors=list(alldata[self.department][self.a].keys())
                    self.delay(["好的，那我们一起来看看都有哪些专业吧", f"{'、'.join(majors)}"])
                    if len(majors) == 1:
                        self.a1 = self.input_format("这个专业")
                        if self.a1!="exit":
                            majordetails().choice(self.a,self.department,list(alldata[self.department][self.a].keys())[0])                        
                    else:
                        self.format("这些专业")
                        self.a1 = input("Me:")
                        count=0
                        while count < len(majors):
                            if count >= 1:
                                self.a1 = self.input_format("其他专业")
                            if self.a1 !="exit":
                                majordetails().choice(self.a,self.department,self.a1)
                                count += 1
                            else:
                                break                  
                else:   
                    self.delay(['OK,那我们一起来看看吧'] + alldata[self.department][self.a].split("\r\n"))
                count1 += 1
            else:
                break
```
* 如果在上一个阶段用户选择了“对应专业”，那么通过下面这个类的执行就会告诉用户该院系中所有的专业，之后根据用户对专业的选择，询问用户想要了解复试范围还是专业详情，并根据用户的选择回复
```python
class majordetails(common):
    
    def choice(self, a, department, major):
        self.a = a
        choice = self.input_delay(["不知大侠接下来想了解复试范围还是专业详情呢？"])
        x = ["专业详情", "复试范围"]
        if "详情" in choice:
            choice=choice[2::]
            x.pop(0)
        else:
            x.pop(1)
            x[0]="详情"
        self.clicklink([f"{alldata[department][self.a][major][choice]}", f"接下来不知道客官还想不想了解一下{x[0]}呢？", "如果不想了解的话就输入exit退出吧~"])
        self.a1 = input("Me:")
        if self.a1 != "exit":
            self.clicklink([f"{alldata[department][self.a][major][x[0]]}"])
```
* common类的作用：
1. 由于上述代码具有重复性，为了简化，设置了一个类供其他类复用，也就是作为其他类的父类
2. 为了模拟人聊天时的状态，让客服的每次输出与用户输入间隔一秒，并且把客服的聊天字体颜色设置为蓝色以便区分
```python
class common:

    wait = 1

    def _format(self, text):
        return colored("Helen:" + text, "blue")

    def delay(self,text):
        for word in text:
            sleep(common.wait)
            print(self._format(word))

    def format(self,text):
        self.delay(["接下来关于"+text+"客官还想继续了解的吗？","如果想继续了解的话就直接输入就OK啦~","如果不想的话就输入exit好啦~"])

    def clicklink(self,link):
        self.delay(["快来点击下面的链接了解详细内容吧~"] + link)

    def input_delay(self, text):
        self.delay(text)
        return input("Me:")

    def input_format(self, text):
        self.format(text)
        return input("Me:")
```
* 用户重复查询的实现：通过每个类中相应的while函数来实现，对于每个范围中的内容，用户都可以分别查询，比如：用户查询完对应专业，可以继续查看概况等，如果退出对于该院系的了解，可以继续了解其他相关院系。（具体可以通过代码运行尝试）
* 当用户将所有可查询类别的信息都查询完以后系统会自动退出到上一个步骤，比如：当用户已经了解过所有对应专业的信息时，不会再询问是否还想了解其他专业，而是询问其他例如概况等信息是否需要了解。
