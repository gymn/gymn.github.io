# mybatis generator使用总结

MyBatis Generator(MBG)是面向MyBatis的代码生成工具。目前支持ibatis的2.2.0以后的版本和MyBatis的所有版本。使用者通过简单的配置即可基于数据库中已创建的数据表快速生成Pojo以及增删改查操作。

MBG的优势：

* 一键式代码生成，使用方便
* 可快速生成基本的增删改查模板代码
* 支持自定义Pojo名称，自动应用驼峰式命名
* 生成的Example文件可以满足90%的单表改查操作

缺陷：

* 不支持多表join
* 不支持存储过程

不过用户可以自己修改xml文件添加自己想要的多表联结查询和存储过程

## 准备工作

在使用MBG之前首先需要准备一份配置文件，以下是一个可用的示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
  <classPathEntry location="/Program Files/mysql/mysql-connection-jdbc-5.1.49.jar" />

  <context id="mysqlTables" targetRuntime="MyBatis3">
    <jdbcConnection driverClass="com.mysql.jdbc.Driver"
        connectionURL="jdbc:mysql://localhost:3306/TEST"
        userId="root"
        password="123">
    </jdbcConnection>

    <javaTypeResolver >
      <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <!--生成Model类存放位置-->
    <javaModelGenerator targetPackage="test.model" targetProject="src/main/java">
      <property name="enableSubPackages" value="true" />
      <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <!--生成映射文件存放位置-->
    <sqlMapGenerator targetPackage="test.mapper"  targetProject="src/main/resources">
      <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>
    <!--生成Dao类存放位置-->
    <javaClientGenerator type="XMLMAPPER" 
                         targetPackage="test.mapper"  targetProject="src/main/java">
      <property name="enableSubPackages" value="true" />
    </javaClientGenerator>
    
     <!-- 配置数据表，可以有多个 -->
    <table tableName="userAccounts" domainObjectName="UserAccount" >
      <property name="useActualColumnNames" value="false"/>
    </table>

  </context>
</generatorConfiguration>
```

在src/main/resources文件夹下创建名为：generatorConfig.xml的文件，填入上述内容。

其次，需要准备jar包或者配置maven插件，本文使用后者：

在pom.xml中添加：

```xml
  <build>
    <plugins>
      <plugin>
        <!--Mybatis-generator插件,用于自动生成Mapper和POJO-->
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-maven-plugin</artifactId>
        <version>1.3.2</version>
      </plugin>
    </plugins>
  </build
```

## 执行

这里推荐两种启动方式：

1. 在图形界面双击插件

   ![mybatis-generator](/assets/images/java/mybatis-generator.JPG)

2. 命令行启动

   在命令行输入：

   mvn mybatis-generator:generate

二者最终生成的代码效果相同。

在执行完毕后，MBG替我们生成了4个文件，分别为：

Model类：UserAccount

Mapper接口文件：UserAccountMapper.java

XML映射文件：UserAccountMapper.xml

还有一个奇怪的文件：UserAccountExample.java

### FAQ

1. 如果重复执行会发生什么？ 对于java代码：会在已有的文件旁边新生成一个文件名以.1结尾的新文件，对于xml文件，会将生成的内容append到原来的文件中；

2. 可不可以不要奇怪的Example文件？ 当然可以，table标签以如下方式书写即可：

   ```xml
    <table tableName="userAccounts" domainObjectName="UserAccount" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false>
         <property name="useActualColumnNames" value="false"/>
     </table>
   ```

## Example文件的使用

MBG在代码生成过程中除了常规的文件外还在Model文件旁边多出了一个Example文件，这个文件就是动态sql的关键，虽然文件名看起来和动态sql没有神马关系。为了不浪费这个文件，有必要学一下这个文件的正确使用姿势~

* where条件拼接：

  ```java
  UserAccountExample example = new UserAccountExample();
  UserAccountExample.Criteria criteria = example.createCriteria();
  criteria.andIDEqualTo(2).andIsDeletedEqualTo((byte)0);
  List<UserAccount> userAccountList = userAccountDao.selectByExample(example);
  ```

简单的介绍下：首先创建一个Example实例，然后通过这个实例创建criteria，criteria就是查询条件，可以链式的建造，上述的查询条件相当于：`where id=2 and isDeleted=0` ，而且可以发现，除了相等意外，example还提供了大于、小于、IN、between等判断功能。不过，细心的小伙伴发现，这些条件只能用and连接，似乎没有`orIDEqualTo(2)` 这样的方法提供，当然我们可以用where in来表达“或”的逻辑，不过正确的方法应该如下。

* 多个条件用“或”连接：

  ```java
  UserAccountExample example = new UserAccountExample();
  UserAccountExample.Criteria criteria = example.createCriteria();
  criteria.andIDEqualTo(2);
  UserAccountExample.Criteria criteria2 = example.or();
  criteria2.andIDEqualTo(5);
  
  List<UserAccount> userAccountList = userAccountDao.selectByExample(example);
  ```

  上述代码相当于`where id=2 or id=5` 

* 更新操作：

  ```java
  UserAccountExample example = new UserAccountExample();
  UserAccountExample.Criteria criteria = example.createCriteria();
  criteria.andIDEqualTo(2);
  
  UserAccount userAccount = new UserAccount();
  userAccount.setName("newName");
  userAccountDao.updateByExampleSelective(userAccount, example);
  ```

  Mapper接口中的selective方法的意义是：只将Model中的属性不为null的值更新到表中。