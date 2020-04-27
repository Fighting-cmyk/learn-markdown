# 研究生招生数据爬取及查询功能介绍 #
## 一、研究生招生数据的爬取 ##
### （一）爬取的内容 ###
* 研究生招生的**院系**，**专业**的名称
* 各个院系的介绍中的所有类别，包含所有可爬取到的**概况，培养特色，导师团队**的具体内容
* 各个专业**复试内容及形式，专业详情，初试考试科目，招生人数**的具体内容
* 各个专业简介中所能爬取到的所有**概况，主要研究方向，主要招生导师(2020)，专业课程，研究生毕业后主要去向，专业希望招收具有何种专业背景的考生**的具体内容
* 各个专业关于**复试的所有年份的相关信息**
### （二） 代码注解 ###
【PS】定义以类来融合和执行所有操作，所以后面粘贴过来的代码方法块最前面都有4个缩进空格
* 获取所有院系的名称(departments)，编号(numberlist)及其**对应专业**的链接(linkist)来供后面形成字典和继续爬取
```python
    def main_information(self):
        self.departments = self.climb("https://yjszs.ecnu.edu.cn/system/sszszyml_list.asp", text=re.compile(r"\(\d{3}\).*"))
        self.numberlist = []
        self.linklist = []
        for department in self.departments:
            self.numberlist.append(department[1:4])
            s = self.getHTMLText("https://yjszs.ecnu.edu.cn/system/sszszyml_list.asp")
            soup = bs4.BeautifulSoup(s, "lxml")
            links=soup.find("a", {"href": re.compile(r"sszyml_list.asp\?zydm=.{{6}}&zsnd=2020&yxdm={}.*".format(int(department[1:4])))})
            self.linklist.append(links.attrs["href"])
```
* 获取各个院系的概况，培养特色，导师团队的具体内容并形成列表
```python
    def getintroduction(self):
        condition = ["概况", "培养特色", "导师团队"]
        introduction_list = []
        for number in self.numberlist:
            details=[]
            introduction = self.climb("https://yjszs.ecnu.edu.cn/system/yxjj_detail.asp?yxdm=" + number + "&zsnd=2020", attrs={"style": "  font-size:14pt; color:blue;line-height:30px;table-layout:fixed;WORD-BREAK: break-all; WORD-WRAP: break-word ;margin-top:40px;margin-left: 50px;"})
            for section in introduction:
                details.append(section.contents[1].string)
            introduction_dict = dict(zip(condition, details))
            introduction_dict["初试科目具体范围"]= "https://yjszs.ecnu.edu.cn/system/sscsfw_list.asp?zsnd=2020&yxdm="+number
            introduction_list.append(introduction_dict)
        return introduction_list
```
【PS】这里为了简化代码定义了一个专门用于爬取的方法，所有的爬取工作都用它来执行
```python
    def climb(self,url,text=0,attrs=0,name=0,text1=0,text2=0):
        s = self.getHTMLText(url)
        soup = bs4.BeautifulSoup(s, "lxml")
        if text:
            return soup.find_all(text=text)
        elif name and attrs:
            return soup.find_all(attrs=attrs), soup.find_all(name=name)
        elif text1 and text2:
            return soup.find_all(text=text1), soup.find_all(text=text2)
        else:
            return soup.find_all(attrs=attrs)
```
* 通过前文提到的对应专业的linklist来继续爬取各个专业关于**复试内容及形式，专业详情，初试考试科目，招生人数**的具体信息，并将其整合生成列表
```python
    def major(self):
        majorlist = []
        for link_index,link in enumerate(self.linklist):
            major_type = self.climb('https://yjszs.ecnu.edu.cn/system/'+link,text=re.compile(r"\(.{6}\).*"))
            major_information = {}
            major_first_scope,major_admitted_number=self.first_subjects("https://yjszs.ecnu.edu.cn/system/sszyml_list.asp?zydm=050102&zsnd=2020&yxdm="+self.numberlist[link_index])
            for major_index,major in enumerate(major_type):
                major_information[major] = {}
                fomular = major[1:7] + "&zsnd=2020&yxdm=" + self.numberlist[link_index]
                majorlink = "https://yjszs.ecnu.edu.cn/system/ssfsfw_list.asp?zydm=" + fomular
                years,second_scopes = self.climb(majorlink, text1=re.compile(r"^\d{4}$"),text2=re.compile(r"1.*2.*"))
                final_scopes=[]
                for scope in second_scopes:
                    if scope.strip()[0] == "1":
                        final_scopes.append(scope.strip())
                    elif scope.strip()[0] == '"':
                        final_scopes.append(scope.strip().strip('"'))
                second_scope = dict(zip(years, final_scopes))
                major_introduction_link = "https://yjszs.ecnu.edu.cn/system/sszszy_detail.asp?zydm=" + fomular
                major_intorduction_details = []
                major_items=[]
                maj_intro_details,maj_intro_items = self.climb(major_introduction_link, attrs={"colspan": "9"},name="b")
                for item_number in range(1,len(maj_intro_items)):
                    major_items.append(maj_intro_items[item_number].contents[0].strip()[:-1])
                for detail in maj_intro_details:
                    try:
                        detail.contents[1]
                    except IndexError:
                        major_intorduction_details.append(" ")
                    else:
                        if isinstance(detail.contents[1].contents[0], bs4.element.Tag):
                            teachers = []
                            for teacher in range(1, len(detail.contents), 2):
                                teachers.append(detail.contents[teacher].contents[0].contents[0])
                            teachers="、".join(teachers)
                            major_intorduction_details.append(teachers)
                        else:
                            major_intorduction_details.append(detail.contents[1].contents[0])
                major_intro = dict(zip(major_items, major_intorduction_details))
                for i in list(major_intro.keys()):
                    if major_intro[i] == " ":
                        major_intro.pop(i)
                major_information[major]["复试内容及形式"] = second_scope
                major_information[major]["专业详情"] = major_intro
                major_information[major]["初试考试科目"] = major_first_scope[major_index]
                major_information[major]["招生人数"] = major_admitted_number[major_index]
            majorlist.append(major_information)
        return majorlist
```
* 对于**初试考试科目**以及**具体范围**链接的爬取
```python
    def first_subjects(self,url):
        first_list = []
        subjects,admitted_students=self.climb(url, text1=re.compile(".*①.*"),text2=re.compile("全日制：.*"))
        for subject in subjects:
            if "选考" in subject:
                subject = subject[subject.find("方")::]
                subject_list_repeat = subject.split("  ")
                subject_list=list(set(subject_list_repeat))
                first_list.append(subject_list)
            else:
                subject = subject[subject.find("①")::]
                first_list.append([subject])
        first_list.append("更多详细考试范围请见：")
        for student_index,student in enumerate(admitted_students):
            admitted_students[student_index] = student[student.find("全"):student.find("限")+4].replace("\r","")
        return first_list,admitted_students

    def exam_scope(self, number):
        return "https://yjszs.ecnu.edu.cn/system/sscsfw_list.asp?zsnd=2020&yxdm="+number
```
* 最后把所有获取到的信息整合生成一个字典并写入json文件
```python
    def run(self):
        self.main_information()
        major = self.major()
        getintroduction=self.getintroduction()
        for i in range(len(getintroduction)):
            getintroduction[i]["对应专业"] = major[i]
        dictionall = dict(zip(self.departments, getintroduction))
        with open("data.json", "w", encoding="utf-8") as f:
            f.write(json.dumps(dictionall,ensure_ascii=False))
```
## 二、用户查询功能 ##
### （一）主要功能 ###
* 供用户**查询**以上所提到的所有内容
* 构建了一个类似**客服机器人**的体系来引导用户查询
* 根据用户的方向，**猜测**他们可能想要了解的院系
* 对于提取到的所有信息，用户都可以分别查询，**多次查询**，并且对于**所有**用户输入的内容实现**模糊识别**以及**错误处理**，增强输入的**容错性**
* 用户在各个步骤中可以按照提示**随时退出**跳到上一个步骤
### （二）实现方式 ###
【PS】通过定义多个类来**代码复用**，以达到简化代码的目的，最后通过一个类的调用来实现所有的功能
* 模拟客服，构建开始和结束方式
* 在开始时给用户提供了两种选择：
1. 针对还没有专业方向的用户，直接告知他们研究生招生的所有院系，并且让他们从中决定自己想要了解的院系
2. 一种是针对已经有大致专业方向的用户，通过他们的大致方向（比如“化学”，“教育”），找到他们可能想要了解的具体院系
* 以下这个类就是程序的主类了
```python
class Hello_and_bye(common):
    
    def __init__(self):
        print(colored("欢迎来到ECNU研究生招生信息查询页面！客服小姐姐Helen将为您服务！\n", "yellow"))
        print(colored("客官对特定的，具体的专业进行查找时建议您输入完整的编号或者部分编号或者输入完整专业名称哦~\n", "red"))
        self.q = ["客官您好，不知道想要想要了解哪方面的信息呢？", "不知客官已经对ECNU的院系心有所属了呢？", "还是想要先了解一下院系呢？", "如果已经心有所属了就直接告诉Helen想要了解的专业方向吧~"]
        self.a = self.input_delay(self.q)
        
    def _think(self,words):
        self.departments = []
        if "院系" in words or "了解" in words:
            self.choice = True
            self.a1 = self.input_delay(["\n".join(list(alldata.keys())), "那么请问客官现在决定好要了解的院系了吗？"])
            for item in list(alldata.keys()):
                if re.search(r".*{}.*".format(self.a1), item):
                    know_department(item).run()
        elif '0' <= words[1] <= '9':
            know_department(words).run()
        else:
            self.choice = False
            while True:
                for item in list(alldata.keys()):
                    if re.search(r".*{}.*".format(self.a), item):
                        self.departments.append(item)
                if len(self.departments) == 1:
                    self.delay([f"我猜您想了解的是{self.departments[0]}"])
                    know_department(self.departments[0]).run()
                    break
                # 错误处理
                elif len(self.departments) == 0:
                    self.a=self.input_delay(["很抱歉，没有找到您要的专业方向","是不是可以换一种表达方式或者您打错了呢~","劳烦您重新输入一下吧~"])
                    continue
                else:
                    self.a1 = self.input_delay([f"检测到您可能想对这些院系有进一步了解：{'、'.join(self.departments)}", "不知客官想要详细了解哪个呢？"])
                    self.while_loop(self.learn_type, term=bool(self.a1.lower() != "exit"), other="其他院系", lense=len(self.departments), answer=self.a1, name_list=self.departments)
                    break

    def while_think(self, answer):
        know_department(answer).run()
    
    def run(self):
        while self.a != "exit":
            self._think(self.a)
            if self.choice==False:
                self.a = self.input_format("其他专业方向")
            else:
                self.a = self.input_format("其他院系")    
        self.delay(["客官慢走呦，到这里，Helen要跟您说再见了，下次有问题继续找我呦，Helen随时恭候您的驾到！"])
```
* 通过上面的问题，已经获取到用户想要了解的院系了，那么对于院系的了解，根据json文件中各个院系所能了解到的信息类别告知用户（例如：概况，培养特色，导师团队，、对应专业），并让用户从中选择想要了解的内容，之后进行回复
```python
class know_department(common):

    def __init__(self, department):
        self.department = department
        self.items = list(alldata[department].keys())
        self.items.remove("初试科目具体范围")
        if len(self.items) == 1:
            self.delay(["Helen只搜集到了关于" + self.items[0] + "的信息呢"])
            self.a=self.items[0]
        else:
            self.q = ["敢问客官想对哪个方面有所了解呢？", f"您可以从{'、'.join(self.items)}中选择哦~"]
            self.a = self.input_delay(self.q)
    
    def while_think1(self, answer):
        majordetails(answer, self.department).run()
    
    def while_think2(self,answer):
        self.delay([majordetails.format_words]+alldata[self.department][answer].split("\r\n"))
               
    def think(self, answer, name_list, unique_func=0):
        if re.search(r".*{}.*".format(answer), "对应专业"):
            self.a = "对应专业"
            majors = list(alldata[self.department][self.a].keys())
            self.delay(["好的，那我们一起来看看都有哪些专业吧", f"{'、'.join(majors)}"])
            if len(majors) == 1:
                self.a1 = self.input_format("这个专业")
                if self.a1 != "exit":
                    self.major = list(alldata[self.department][self.a].keys())[0]
                    majordetails(self.major, self.department).run()                    
            else:
                self.auto_answer_format("这些专业")
                self.a1 = input("Me:")
                self.while_loop(self.learn_type, answer=self.a1, other="其他专业", lense=len(majors), name_list=majors, unique_func=self.while_think1)      
            return "对应专业"
        else:
            result = self.learn_type(answer, name_list, unique_func=self.while_think2)
            return result if result else ""
                    
    def run(self):
        self.while_loop(self.think, lense=len(self.items), other="这个院系", answer=self.a, name_list=self.items)
```
* 如果在上一个阶段用户选择了“对应专业”，那么通过下面这个类的执行就会告诉用户该院系中所有的专业，之后根据用户对专业的选择，询问用户想要了解信息（比如：复试内容及形式，专业详情，初试考试科目，招生人数，概况，主要研究方向，主要招生导师(2020)，专业课程，研究生毕业后主要去向，专业希望招收具有何种专业背景的考生等），并根据用户的选择回复
```python
class majordetails(know_department):

    format_words = 'OK,那我们一起来看看吧'

    def __init__(self,major,department):
        self.major = major
        self.department = department
        self.format_index = alldata[self.department]["对应专业"][self.major]
    
    def _choice(self, choice, default, unique_func=0):
        if re.search(r".*{}.*".format(choice), "复试内容及形式"):
            year_all = list(self.format_index["复试内容及形式"].keys())
            if len(year_all) == 1:
                self.delay(["Helen只找到了" + year_all[0] + "的信息呢", self.format_index["复试内容及形式"][year_all[0]]])
            else:
                year = self.input_delay(["客官想了解哪一年的信息呢？", "Helen收集了" + "、".join(year_all) + "这些年的信息呢！"])
                self.while_loop(self.second_years, lense=len(year_all), other="其他年份", answer=year, name_list=list(year_all))
            return "复试内容及形式"

        elif re.search(r".*{}.*".format(choice), "专业详情"):
            i_tems=list(self.format_index["专业详情"].keys())
            items = self.input_delay(["客官想了解哪方面的信息呢？", "您可以从" + "、".join(i_tems) + "中选择哦~"])
            self.while_loop(self.subject_details, lense=len(self.format_index["专业详情"].keys()), other="其他信息", answer=items, name_list=i_tems)
            return "专业详情"

        elif re.search(r".*{}.*".format(choice), "初试考试科目"):
            self.delay([majordetails.format_words] + self.format_index["初试考试科目"] + ["如果你想了解更详细的范围就点击下面的链接吧~", alldata[self.department]["初试科目具体范围"]])
            return "初试考试科目"

        elif re.search(r".*{}.*".format(choice), "招生人数"):
            self.delay([majordetails.format_words]+self.format_index["招生人数"].split("\n\n"))
            return "招生人数"
        
    def while_think(self, answer):
        self.delay(self.format_index["专业详情"][answer].split("\r\n"))

    def subject_details(self, answer, name_list, unique_func=0):
        result = self.learn_type(answer, name_list)
        return result if result else ""
                
    def second_years(self, answer, name_list, unique_func=0):
        if answer in name_list:
            self.delay([self.format_words] + self.format_index["复试内容及形式"][answer].split("\r\n"))
            return answer

    def run(self):
        types=list(self.format_index.keys())
        for type_ in types:
            if not self.format_index[type_]:
                types.remove(type_)
        choice = self.input_delay(["不知客官接下来想了解以下哪个方面呢？", "客官可以从" + "、".join(types) + "中选择哦~"])
        self.while_loop(self._choice, lense=len(self.format_index.keys()), other="其他方面", answer=choice, name_list=types)
```
* common类的作用：
1. 由于上述代码具有重复性，为了简化，设置了一个类供其他类复用，也就是作为其他类的**父类**
2. 为了模拟人聊天时的状态，让客服的每次输出与用户输入间隔一秒，并且把客服的聊天字体颜色设置为**蓝色**以便区分
* 其他介绍在下面程序的注释中都有写到
```python
class common:

    wait = 1

    def color_format(self, text):
        '''用于设置用户和客服的字体颜色来进行区分'''
        return colored("Helen:" + text, "blue")

    def delay(self, text):
        '''用于模拟真实聊天情景中的输入输出时间间隔'''        
        for sentence in text:
            sleep(common.wait)
            print(self.color_format(sentence))

    def auto_answer_format(self, other, othertype=''):
        '''用于模式化回答方式'''
        if othertype:
            self.delay(["接下来关于" + other + "客官还想继续了解吗？", "您还可以选择" + othertype + "哦~", "如果想继续了解的话就直接输入就OK啦~", "如果不想的话就输入exit好啦~"])
        else:
            self.delay(["接下来关于" + other + "客官还想了解吗？", "如果想继续了解的话就直接输入就OK啦~", "如果不想的话就输入exit好啦~"])

    def input_delay(self, text):
        '''获取非模式化文本后的用户输入信息'''
        self.delay(text)
        return input("Me:")

    def input_format(self, other, othertype=''):
        '''获取模式化文本后的用户输入信息'''
        self.auto_answer_format(other, othertype=othertype)
        return input("Me:")
    
    def while_think(self, answer):
        '''用于后面类中的调用和填充'''
        pass
    
    def while_loop(self, func, term=True, count=0, lense=0, other="", answer='', name_list=[],unique_func=0):
        '''实现用户多次查询的功能以及用户输入的错误处理'''
        while count < lense and term:
            if count >= 1:
                answer = self.input_format(other, othertype="、".join(name_list))
            if answer.lower() != "exit":
                while True:
                    answer = func(answer, name_list,unique_func=unique_func)
                    if not answer:
                        answer = self.input_delay(["貌似没有找到客官输入的内容唉~", "劳烦客官检查一下是不是重复输入了，或者输错了呢，重新输入一下看看吧~"])
                        continue
                    else:
                        count += 1
                        if name_list:
                            name_list.remove(answer)
                        break
            else:
                break

    def learn_type(self, answer, format_type,unique_func=0):
        '''实现模糊识别，实现用户在查找过一个方面后不需要向上翻聊天记录就可以直接'''
        for type_ in format_type:
            if re.search(r".*{}.*".format(answer), type_):
                if not unique_func:
                    self.while_think(type_)
                else:
                    unique_func(type_)
                return type_
```
* 用户重复查询的说明：通过每个类中相应的while函数来实现，对于每个范围中的内容，用户都可以分别查询，比如：用户查询完对应专业，可以继续查看概况等，如果退出对于该院系的了解，可以继续了解其他相关院系。（具体可以通过代码运行尝试）
* 当用户将所有可查询类别的信息都查询完以后系统会自动退出到上一个步骤，比如：当用户已经了解过所有对应专业的信息时，不会再询问是否还想了解其他专业，而是询问其他例如概况等信息是否需要了解。
## 三、小结 ##
* 代码很多地方都需要更进一步优化
1. 代码还可以更加简洁
2. 对数据给用户的呈现形式还可以更美观
3. 对于备注信息没有爬取

