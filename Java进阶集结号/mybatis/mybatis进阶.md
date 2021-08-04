### mybatis
1. mybatis 中 #{}和 ${}的区别是什么？

> 1. #{}是占位符，$是拼接符；
>
> 2. #{}的SQL是预编译的（先将参数用?代替再编译，编译完成后使用PreparedStatement的set方法来赋值参数），${}是先替换参数再编译（处理${}时，就是把${}替换成变量的值再去编译）；
>
> 3. #{}能防止SQL注入，而${}不能防止SQL注入；
>
>    一般能用#{}的就别用${}。MyBatis排序式使用order by动态参数时需要注意，用${}而不是#{}。

2. 什么是SQL注入 ，如何避免。

> SQL注入就是攻击者提交带有恶意的数据与SQL语句进行字符串方式的拼接，使后台的SQL的语义发生变化，最终产生数据泄露的现象。

3. 说一下 mybatis 的一级缓存和二级缓存

> 一级缓存：
>
> 在参数和SQL完全一样的情况下，我们使用同一个SqlSession对象调用一个Mapper方法，往往只执行一次SQL，因为使用SelSession第一次查询后，MyBatis会将其放在缓存中，以后再查询的时候，如果没有声明需要刷新，并且缓存没有超时的情况下，SqlSession都会取出当前缓存的数据，而不会再次发送SQL到数据库。
>  ![一级缓存简单示意图](https://img-blog.csdnimg.cn/20201126194312471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhbnpodWFuaHU=,size_16,color_FFFFFF,t_70#pic_center) 
>
>  一级缓存时执行commit，close，增删改等操作，就会清空当前的一级缓存；当对SqlSession执行更新操作（update、delete、insert）后并执行commit时，不仅清空其自身的一级缓存（执行更新操作的效果），也清空二级缓存（执行commit()的效果）。 
>
> **二级缓存：**
>
> 二级缓存是SqlSessionFactory级别的缓存，同一个SqlSessionFactory产生的SqlSession都共享一个二级缓存，二级缓存中存储的是数据，当命中二级缓存时，通过存储的数据构造对象返回。
>
> 查询数据的时候，查询的流程是二级缓存>一级缓存>数据库。
>
>  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201126194528869.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhbnpodWFuaHU=,size_16,color_FFFFFF,t_70#pic_center) 
>
>  在配置文件中 开启二级缓存的总开关 ： <setting name="cacheEnabled" value="true" /> 
>
>  在mapper映射文件中开启二级缓存 ：
>
>  <cache eviction="FIFO" flushInterval="60000" size="512"  readOnly="true"/> 
>
>  参数名属性eviction收回策略flushInterval刷新间隔size引用数目readOnly只读 
>
>  **关于eviction的各个参数属性:** 
>
> - 参数名属性eviction="LRU"最近最少使用的:移除最长时间不被使用的对象。
>   （默认）
> - eviction="FIFO"先进先出:按对象进入缓存的顺序来移除它们。
> - eviction="SOFT"软引用:移除基于垃圾回收器状态和软引用规则的对象。
> - eviction="WEAK"弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象。
>
>  注意实体类要实现Serializable 

4. mybatis 是否支持延迟加载？延迟加载的原理是什么？

> MyBatis 实现一对一有几种方式?具体怎么操作的？
> 有联合查询和嵌套查询
> 联合查询是几个表联合查询,只查询一次, 通过在resultMap 里面配置 association 节点配置一对一的类就可以完成；
> 嵌套查询是先查一个表，根据这个表里面的结果的 外键 id，去再另外一个表里面查询数据,也是通过 association 配置，但另外一个表的查询通过 select 属性配置。
> <mapper namespace="com.lcb.mapping.userMapper">
> <!--association 一对一关联查询 -->
> <select id="getClass" parameterType="int"
> resultMap="ClassesResultMap">
> select * from class c,teacher t where c.teacher_id=t.t_id and
> c.c_id=#{id}
> </select>
> <resultMap type="com.lcb.user.Classes" id="ClassesResultMap">
> <!-- 实体类的字段名和数据表的字段名映射 -->
> <id property="id" column="c_id"/>
> <result property="name" column="c_name"/>
> <association property="teacher"
> javaType="com.lcb.user.Teacher">
> <id property="id" column="t_id"/>
> <result property="name" column="t_name"/>
> </association>
> </resultMap>MyBatis 实现一对多有几种方式,怎么操作的？
> 有联合查询和嵌套查询。
> 联合查询是几个表联合查询,只查询一次,通过在resultMap 里面的 collection 节点配置一对多的类就可以完成；
> 嵌套查询是先查一个表,根据这个表里面的 结果的外键 id,再去另外一个表里面查询数据,也是通过配置 collection,但另外一个表的查询通过 select 节点配置
> <!--collection 一对多关联查询 -->
> <select id="getClass2" parameterType="int"
> resultMap="ClassesResultMap2">
> select * from class c,teacher t,student s where c.teacher_id=t.t_id
> and c.c_id=s.class_id and c.c_id=#{id}
> </select>
> <resultMap type="com.lcb.user.Classes" id="ClassesResultMap2">
> <id property="id" column="c_id"/>
> <result property="name" column="c_name"/>
> <association property="teacher"
> javaType="com.lcb.user.Teacher">
> <id property="id" column="t_id"/>
> <result property="name" column="t_name"/>
> </association>
> <collection property="student"
> ofType="com.lcb.user.Student">
> <id property="id" column="s_id"/>
> <result property="name" column="s_name"/>
> </collection>
> </resultMap>
> </mapper>
> Mybatis 是否支持延迟加载？如果支持，它的实现原理是什么？
> Mybatis 仅支持 association 关联对象和 collection 关联集合对象的延迟加载，association 指的就是一对一，collection 指的就是一对多查询。在 Mybatis配置文件中，可以配置是否启用延迟加载 lazyLoadingEnabled=true|false。它的原理是，使用 CGLIB 创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用 a.getB().getName()，拦截器 invoke()方法发现 a.getB()是null 值，那么就会单独发送事先保存好的查询关联 B 对象的 sql，把 B 查询上来，然后调用 a.setB(b)，于是 a 的对象 b 属性就有值了，接着完成 a.getB().getName()方法的调用。这就是延迟加载的基本原理。
> 当然了，不光是 Mybatis，几乎所有的包括 Hibernate，支持延迟加载的原理都
> 是一样的。

5. mybatis 动态sql中使用<where>标签与直接写where关键字有什么区别？

> <select id="selectUserByUsernameAndSex" resultType="user" parameterType="com.ys.po.User">
> 	select * from user
> 	<where>
> 		<if test="username != null">
> 			username=#{username}
> 		</if>
> 		<if test="sex != null">
> 			and sex=#{sex}
> 		</if>
> 	</where>
> <select>
>
>  这个where标签会判断它包含的标签中有返回值的话，它就插入一个“where”。此外，如果标签返回的内容是以AND或OR开头的，则它会剔除掉AND或OR。

6. mybatis 动态sql标签中循环标签中有哪些属性，各自的作用。

> foreach标签主要用于构建in条件，他可以在sql中对集合进行迭代。如下：
>
> 　　<delete id="deleteBatch"> 
>
> 　　　　delete from user where id in
>
> 　　　　<foreach collection="array" item="id" index="index" open="(" close=")" separator=",">
>
> 　　　　　　#{id}
>
> 　　　　</foreach>
>
> 　　</delete>
>
> 　　我们假如说参数为----  int[] ids = {1,2,3,4,5}  ----那么打印之后的SQL如下：
>
> 　　delete form user where id in (1,2,3,4,5)
>
> 　　释义：
>
> 　　　　collection ：collection属性的值有三个分别是list、array、map三种，分别对应的参数类型为：List、数组、map集合，我在上面传的参数为数组，所以值为array
>
> 　　　　item ： 表示在迭代过程中每一个元素的别名
>
> 　　　　index ：表示在迭代过程中每次迭代到的位置（下标）
>
> 　　　　open ：前缀
>
> 　　　　close ：后缀
>
> 　　　　separator ：分隔符，表示迭代时每个元素之间以什么分隔
>
> 我们通常可以将之用到批量删除、添加等操作中。

7. mybatis 和 hibernate 的区别有哪些？

> Mybatis和hibernate不同，它不完全是一个ORM框架，因为MyBatis需要程序员自己编写Sql语句。
>
> Mybatis直接编写原生态sql，可以严格控制sql执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。但是灵活的前提是mybatis无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套sql映射文件，工作量大。
>
> Hibernate对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用hibernate开发可以节省很多代码，提高效率。

8. RowBounds是一次性查询全部结果吗？为什么？

>  RowBounds 表面是在“所有”数据中检索数据，其实并非是一次性查询出所有数据，因为 MyBatis 是对 jdbc 的封装，在 jdbc 驱动中有一个 Fetch Size 的配置，它规定了每次最多从数据库查询多少条数据，假如你要查询更多数据，它会在你执行 next()的时候，去查询更多的数据。就好比你去自动取款机取 10000 元，但取款机每次最多能取 2500 元，所以你要取 4 次才能把钱取完。只是对于 jdbc 来说，当你调用 next()的时候会自动帮你完成查询工作。这样做的好处可以有效的防止内存溢出。 

9. MyBatis 定义的接口，怎么找到实现的？

> 

10. Mybatis的底层实现原理。

> 

11. Mybatis是如何进行分页的？分页插件的原理是什么？

> Mybatis 使用 RowBounds 对象进行分页，也可以直接编写 sql 实现分页，也可以使用Mybatis 的分页插件。
> 分页插件的原理：实现 Mybatis 提供的接口，实现自定义插件，在插件的拦截方法内拦截待执行的 sql，然后重写 sql。
> 举例：select * from student，拦截 sql 后重写为：select t.* from （select * from student）tlimit 0，10
> 

12. Mybatis执行批量插入，能返回数据库主键列表吗？

> 

13. Mybatis都有哪些Executor执行器？它们之间的区别是什么？

> Mybatis有三种基本的执行器（Executor）：
>
> - SimpleExecutor：每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象。
> - ReuseExecutor：执行update或select，以sql作为key查找Statement对象，存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map内，供下一次使用。简言之，就是重复使用Statement对象。
> - BatchExecutor：执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理。与JDBC批处理相同
>
> mybatis运行图：
>
>  ![mybatis.jpg](https://www.yukx.com/upload/images/202012/141607954574660022363.jpg) 

14. Mybatis动态sql有什么用？执行原理？有哪些动态sql？

> 

15. mybatis有几种分页方式？

> - 数组分页
>
> ![1628003872104](C:\Users\Administrator.USER-20190223MK\AppData\Roaming\Typora\typora-user-images\1628003872104.png)
>
> - sql分页
>
> ```
> <select id="queryStudentsBySql" parameterType="map" resultMap="studentmapper">
>         select * from student limit #{currIndex} , #{pageSize}
> </select>
> ```
>
> - 拦截器分页
>
> - RowBounds分页
>
> 数据量小时，RowBounds不失为一种好办法。但是数据量大时，实现拦截器就很有必要了。
>
> mybatis接口加入RowBounds参数
>
> ```
> public List<UserBean> queryUsersByPage(String userName, RowBounds rowBounds);
> ```
>
> service
>
> ```
> @Override
>     @Transactional(isolation = Isolation.READ_COMMITTED, propagation = Propagation.SUPPORTS)
>     public List<RoleBean> queryRolesByPage(String roleName, int start, int limit) {
>         return roleDao.queryRolesByPage(roleName, new RowBounds(start, limit));
>     }
> ```

16. MyBatis框架的优点和缺点

> 一、MyBatis框架的优点：
>
> 1. 与JDBC相比，减少了50%以上的代码量。
>
> 2. MyBatis是最简单的持久化框架，小巧并且简单易学。
>
> 3. MyBatis相当灵活，不会对应用程序或者数据库的现有设计强加任何影响，SQL写在XML里，从程序代码中彻底分离，降低耦合度，便于统一管理和优化，并可重用。
>
> 4. 提供XML标签，支持编写动态SQL语句。
>
> 5. 提供映射标签，支持对象与数据库的ORM字段关系映射。
>
> 二、MyBatis框架的缺点：
>
> 1. SQL语句的编写工作量较大，尤其是字段多、关联表多时，更是如此，对开发人员编写SQL语句的功底有一定要求。
>
> 2. SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。
>
>
> 三、MyBatis框架适用场合：
>
> MyBatis专注于SQL本身，是一个足够灵活的DAO层解决方案。
>
> 对性能的要求很高，或者需求变化较多的项目，如互联网项目，MyBatis将是不错的选择。

17. 使用MyBatis框架，当实体类中的属性名和表中的字段名不一样 ，怎么办 ？

> **方法一：写SQL语句时起别名**
>
> ```
> 	<select id="getEmployeeById" resultType="com.atguigu.mybatis.entities.Employee">
> 		select id,first_name firstName,email,salary,dept_id deptID from employees where id = #{id}
> 	</select>
> ```
>
> **方法二：在MyBatis的全局配置文件中开启驼峰命名规则**
>
> ```
> mapUnderscoreToCamelCase：true/false 
> <!--是否启用下划线与驼峰式命名规则的映射（如first_name => firstName）-->
> <configuration>  
>     <settings>  
>         <setting name="mapUnderscoreToCamelCase" value="true" />  
>     </settings>  
> </configuration>
> ```
>
> **方法三：在Mapper映射文件中使用resultMap来自定义映射规则**
>
> ```
> 	<select id="getEmployeeById" resultMap="myMap">
> 		select * from employees where id = #{id}
> 	</select>
> 	
> 	<!-- 自定义高级映射 -->
>     <resultMap type="com.atguigu.mybatis.entities.Employee" id="myMap">
>     	<!-- 映射主键 -->
>     	<id column="id" property="id"/>
>     	<!-- 映射其他列 -->
>     	<result column="last_name" property="lastName"/>
>     	<result column="email" property="email"/>
>     	<result column="salary" property="salary"/>
>     	<result column="dept_id" property="deptId"/>
>     </resultMap>
> ```

18. 通常一个Xml映射文件，都会写一个Dao接口与之对应，请问，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗？

>    Dao接口即Mapper接口。接口的全限名（命名空间）就是映射文件中的namespace的值，用于绑定Dao接口；接口的方法名就是映射文件中Mapper的Statement的id值；接口方法内的参数就是传递给sql的参数。 
>
>    在Mybatis中，每一个 <select>、<insert>、<update>、<delete>标签，都会被解析为一个MapperStatement对象，用于描述一条SQL语句。Mapper接口是没有实现类的，当调用接口方法时，由接口全限名+方法名拼接字符串作为key值，可唯一定位一个MapperStatement。 
>
>    举例来说：cn.mybatis.mappers.StudentDao.findStudentById，可以唯一找到namespace为 com.mybatis.mappers.StudentDao下面 id 为 findStudentById 的 MapperStatement。 
>
> ![1628004197775](C:\Users\Administrator.USER-20190223MK\AppData\Roaming\Typora\typora-user-images\1628004197775.png)
>
>    Mapper接口里的方法，是不能重载的，因为是使用 全限名+方法名 的保存和寻找策略。  Mapper 接口的工作原理  是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理对象proxy，代理对象会拦截接口方法，转而执行MapperStatement所代表的sql，然后将sql执行结果返回。 
>
>    那什么是动态代理呢？动态代理就是在程序运行期间由JVM通过反射等机制动态生成的，所以不会存在代理类的字节码文件，故我们在Mybatis中使用mapper接口的时候没有它的实现类，代理对象和真实对象的关系是由运行时期才决定的。 

19. Xml映射文件中，除了常见的select|insert|updae|delete标签之外，还有哪些标签？

> <where><foeach><if><choose><when><otherwise>等等；

20. 简述Mybatis的插件运行原理，以及如何编写一个插件。

> Mybatis自定义插件针对Mybatis四大对象（Executor、StatementHandler 、ParameterHandler 、ResultSetHandler ）进行拦截，具体拦截方式为：
>
> - Executor：拦截执行器的方法(log记录)
> - StatementHandler ：拦截Sql语法构建的处理
> - ParameterHandler ：拦截参数的处理
> - ResultSetHandler ：拦截结果集的处理
>
> Mybatis自定义插件必须实现Interceptor接口：
>
> ```
> public interface Interceptor {
>     Object intercept(Invocation invocation) throws Throwable;  //拦截器具体处理逻辑方法 
>     Object plugin(Object target);  //根据签名signatureMap生成动态代理对象 
>     void setProperties(Properties properties);  //设置Properties属性
> }
> ```
>
> 自定义插件demo：
>
> ```
> // ExamplePlugin.java
> @Intercepts({@Signature(
>   type= Executor.class,
>   method = "update",
>   args = {MappedStatement.class,Object.class})})
> public class ExamplePlugin implements Interceptor {
>   public Object intercept(Invocation invocation) throws Throwable {
>   Object target = invocation.getTarget(); //被代理对象
>   Method method = invocation.getMethod(); //代理方法
>   Object[] args = invocation.getArgs(); //方法参数
>   // do something ...... 方法拦截前执行代码块
>   Object result = invocation.proceed();
>   // do something .......方法拦截后执行代码块
>   return result;
>   }
>   public Object plugin(Object target) {
>     return Plugin.wrap(target, this);
>   }
>   public void setProperties(Properties properties) {
>   }
> }
> ```
>
> 一个@Intercepts可以配置多个@Signature，@Signature中的参数定义如下：
>
> - type：表示拦截的类，这里是Executor的实现类；
> - method：表示拦截的方法，这里是拦截Executor的update方法；
> - args：表示方法参数。

