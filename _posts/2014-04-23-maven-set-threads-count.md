---
layout: post
title: Maven set dynamic thread count for TestNG 
---
Вот и первая статейка =)
Кароче такое дело, все ж автоматизаторы хотят ШОБ их тесты проганялись быстро и конечно же паралельно, мы ж крутые нах.
Хотя может и в сраку паралельность, лишний геморой себе и окружающим. То там отпадет, то там конфилит, поди пойми баг то, или конфликт, или Луна не в водолее блэт!
Кароче дело такое, но если решились то есть разные способы это сделать. Я покажу как это делал и делаю я.
Вы можете сказать: "Фууу Maven выбрла как сборщик, олд фаг и не хипстерок".
Скажу в свое оправдание что [Gradle](https://gradle.org/) что [Maven](https://maven.apache.org/) мне вапще пох, просто Maven знаю немного лучше и больше с ним работал.
Что их них лучше решать вам, в интернетах куча статей. Ну а про то что [TestNg](http://testng.org/doc/documentation-main.html) может паралельно запускать тест думаю вы знаете(если нет, то скорее всего вы днарь).
Так вот, в TestNG есть такая штука как [testng-xml](http://testng.org/doc/documentation-main.html#testng-xml) файл, в нем можна формировать тест сьюты и делать прочее мракоесие,
матчасть погуглите в интернетах.

Вот так примерно выхлядит testng.xml для паралельного запуска тестовых методов:
```xml
<suite name="Demo suite" verbose="1" parallel="methods" thread-count="1">

    <test name="Demo Navigation tests tests" >
        <classes>
            <class name="org.seleniumhq.NavigationTests"/>
        </classes>
    </test>
</suite>
```
Нас интересуют вот эти два **parallel** и **thread-count**:
- **parallel** - If specified, sets the default mechanism used to determine how to use parallel threads when running tests. If not set, default mechanism is not to use parallel threads at all. This can be overridden in the suite definition.
- **thread-count** -This sets the default maximum number of threads to use for running tests in parallel. It will only take effect if the parallel mode has been selected (for example, with the -parallel option). This can be overridden in the suite definition.
 
Жесткий копипаст с оф докиментаци, но малоли може комуто впадлу гуглить, я же добрый человек.
 
Итак шо мы имеем, мы можем сказать как паралелить и во сколько потоков. Все агонь, я крутой давайте больше денег блэт!
Вроде работает, но мы конечно же люди умные, занем что там в тестНГ не так все хорошо, но в общем гавно и палки работает более менее как надо.
Понаписывали тестов, сделали джобов на СиАй все работает, но вдруг мы замечаем, что на каких-то наших виртуалках 5 браузеров это архидахуа, а на киких - то и в 10 потоков можна заранить.
Шо ж делать, давайте параметризировать testng.xml.
 
Идем гуглить...
Вводим заветную строку в гугл "testng dynamic parameters testng.xml"
Но это я бы так ввел, може я слишком тупой и нормальный человек как то по другому это сделает, но я сделал бы так)
 
Кароче находим первую статью на любимом [СтакОверфлоу](https://stackoverflow.com/) и можем чутка наложить в штаны.
Видим какуюто садомию с созданием кастомной xml из кода, из КОДА, from the code блэт! Как по мне там карочи ацкий ад, я бы так не делал. Не ну бывают конечно ситуации
когда это надо, но цэ точно ни ана. Тогда вспоминаем про мавЭн и включаем разум. Включаем еще и логику и думаем. у нас мавЭн проект и его стуркута [ТАКАЯ](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)
Итого вспомнили структуру и видим там такую штуку **src/test/filters -> Test resource filter files**. Хоп хэй лалалэй, Вот жеш оно!
Идем логически дальше, [Filtering](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html) и видим первую заветную строчку!
**Variables can be included in your resources. These variables, denoted by the ${...} delimiters, can come from the system properties, your project properties, from your filter resources and from the command line.**
Ае ае бежим в припрыжку менять наш код!
 
Сначала меняем [pom.xml](https://maven.apache.org/pom.html#What_is_the_POM):
  ```xml
 <testResources>
     <testResource>
         <directory>src/test/resources/</directory>
         <filtering>true</filtering>
             <targetPath>${project.build.testOutputDirectory}/</targetPath>
     </testResource>
  </testResources> 
  ```
  
**!!!ВАРНИНГ!!!** Не забываем что мы пишем тесты! По-сему указываем testResource и <targetPath>${project.build.testOutputDirectory}/</targetPath>.
Это так к слову, вдруг вы все в папке main делает, так шо не раковать солдат!
 
Дальше прокачиваем на testng.xml:
 ```xml
 <suite name="Demo suite" verbose="1" parallel="methods" thread-count="${thread.count}">
 
     <test name="Demo Navigation tests tests" >
         <classes>
             <class name="org.seleniumhq.NavigationTests"/>
         </classes>
     </test>
 </suite>
 ```
 
Вроде все скопмпились, хотя че ему че уме не компилиться, ничеж не меянли почти, дурак или шо.
Запускаем тесты попивая кофейок аки король вселенной mvn clean compile test **-Dthread.count=3** и тут на нахуй!!
 
 **[ERROR] There was an error in the forked process**
 
 **[ERROR] java.lang.NumberFormatException: For input string: "${thread.count}"**
 
 **[ERROR] org.apache.maven.surefire.booter.SurefireBooterForkException: There was an error in the forked process**

 **[ERROR] java.lang.NumberFormatException: For input string: "${thread.count}"**

ВТФ! Все же по инструкци! Начинаем чутка печалиться и думать шо не так...

Ага, вдруг мавен думает что такой проперти нет. Так возьмем и добавим!
 ```xml
<properties>
     <thread.count>1</thread.count>
 </properties>
   ```
   
Запускаем еще раз... Пик пик пик! Хрен угадал, вонючий **java.lang.NumberFormatException: For input string: "${thread.count}"**
Блиииииинннннн, а счатье было так близко!
Но ты же боец, ты не воючий слюнтяй! Собираешь волю в кулак в включаешь оставшуюся часть мозга, которую еще не занял юуту или другие соц сети.
Если оно не подменило нужный нам ресур, наверное дело в какомто - пути. Начинаем опят гуглить.
Смотрим в pom и и видим что в фильтре мы указали такой вот путь:
   
   ```xml
   <targetPath>${project.build.testOutputDirectory}/</targetPath> 
   ```
а testng.xml у нас берется по привычке вот отсюдава:
   ```xml
   <suiteXmlFile>src/test/resources/testng.xml</suiteXmlFile>
   ```
   
Ну че, бывает, кто не затупливал в жизни то. Быстро меняем наши путь в [maven-surefire-plugin](http://maven.apache.org/surefire/maven-surefire-plugin/):
 ```xml
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-surefire-plugin</artifactId>
      <version>2.20</version>
        <configuration>
            <suiteXmlFiles>
                  <suiteXmlFile>${project.build.testOutputDirectory}/testng.xml</suiteXmlFile>
            </suiteXmlFiles>
        </configuration>
  </plugin>пше 
   ```
   
АЛИЛУЯ возьме свет!!!!! Все завелось, все работает как мы хотим! Ты счастлив, колеги тоже, может быть еще кто-то.
Зада выполнена, папиисят и расходимся.


В Allure смотрим что все отработало!

![_config.yml]({{ site.baseurl }}/images/allure-threads.png)

Код на гитхабе [ЗДЭС](https://github.com/apanashchenko/maven-thread-count)
Заюзал для примера [Selenide](http://selenide.org/) сильно нравицааааа и [Allure](http://allure.qatools.ru/) тож самое!  

   
**З.Ы.** Вот такая штука, просто и легко! надеюсь кому-то это помогло сокраить пару часов жизни на инфестигейшен!
         Хэв э найс дэй!
   
   
 
 
 
 
 
 
  
 
 
 
 

