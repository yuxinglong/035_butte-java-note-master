# 一、JPA框架简介

JPA(Java Persistence API)意即Java持久化API，是Sun官方在JDK5.0后提出的Java持久化规范。主要是为了简化持久层开发以及整合ORM技术，结束Hibernate、TopLink、JDO等ORM框架各自为营的局面。JPA是在吸收现有ORM框架的基础上发展而来，易于使用，伸缩性强。

# 二、SpringBoot2整合

## 1、核心依赖

```xml
<!-- JPA框架 -->
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-data-jpa</artifactid>
</dependency>
```

## 2、配置文件

```
spring:
  application:
    name: node09-boot-jpa
  datasource:
    url: jdbc:mysql://localhost:3306/data_jpa?useUnicode=true&amp;characterEncoding=UTF-8&amp;allowMultiQueries=true
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

ddl-auto几种配置说明

- create

每次加载hibernate时都删除上一次的生成的表，然后根据bean类重新来生成新表，容易导致数据丢失，（建议首次创建时使用）。

- create-drop

每次加载hibernate时根据bean类生成表，但是sessionFactory一关闭,表就自动删除。

- update

第一次加载hibernate时根据bean类会自动建立起表的结构，以后加载hibernate时根据bean类自动更新表结构，即使表结构改变了但表中的行仍然存在不会删除以前的行。

- validate

每次加载hibernate时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。

## 3、实体类对象

就是根据这个对象生成的表结构。

```java
@Table(name = "t_user")
@Entity
public class User {
    @Id
    @GeneratedValue
    private Integer id;
    @Column
    private String name;
    @Column
    private Integer age;
    // 省略 GET SET
}
```

## 4、JPA框架的用法

定义对象的操作的接口，继承JpaRepository核心接口。

```java
import com.boot.jpa.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
@Repository
public interface UserRepository extends JpaRepository<user,integer> {

    // 但条件查询
    User findByAge(Integer age);
    // 多条件查询
    User findByNameAndAge(String name, Integer age);
    // 自定义查询
    @Query("from User u where u.name=:name")
    User findSql(@Param("name") String name);
}
```

## 5、封装一个服务层逻辑

```java
import com.boot.jpa.entity.User;
import com.boot.jpa.repository.UserRepository;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
@Service
public class UserService {
    @Resource
    private UserRepository userRepository ;
    // 保存
    public void addUser (User user){
        userRepository.save(user) ;
    }
    // 根据年龄查询
    public User findByAge (Integer age){
        return userRepository.findByAge(age) ;
    }
    // 多条件查询
    public User findByNameAndAge (String name, Integer age){
        return userRepository.findByNameAndAge(name,age) ;
    }
    // 自定义SQL查询
    public User findSql (String name){
        return userRepository.findSql(name) ;
    }
    // 根据ID修改
    public void update (User user){
        userRepository.save(user) ;
    }
    //根据id删除一条数据
    public void deleteStudentById(Integer id){
        userRepository.deleteById(id);
    }
}
```

# 三、测试代码块

```java
import com.boot.jpa.JpaApplication;
import com.boot.jpa.entity.User;
import com.boot.jpa.service.UserService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import javax.annotation.Resource;
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = JpaApplication.class)
public class UserJpaTest {
    @Resource
    private UserService userService ;
    @Test
    public void addUser (){
        User user = new User() ;
        user.setName("知了一笑");
        user.setAge(22);
        userService.addUser(user);
        User user1 = new User() ;
        user1.setName("cicada");
        user1.setAge(23);
        userService.addUser(user1);
    }
    @Test
    public void findByAge (){
        Integer age = 22 ;
        // User{id=3, name='知了一笑', age=22}
        System.out.println(userService.findByAge(age));
    }
    @Test
    public void findByNameAndAge (){
        System.out.println(userService.findByNameAndAge("cicada",23));
    }
    @Test
    public void findSql (){
        // User{id=4, name='cicada', age=23}
        System.out.println(userService.findSql("cicada"));
    }
    @Test
    public void update (){
        User user = new User() ;
        // 如果这个主键不存在，会以主键自增的方式新增入库
        user.setId(3);
        user.setName("哈哈一笑");
        user.setAge(25);
        userService.update(user) ;
    }
    @Test
    public void deleteStudentById (){
        userService.deleteStudentById(5) ;
    }
}
```

**参考源码** ：https://gitee.com/cicadasmile/spring-boot-base/tree/master/node09-boot-jpa