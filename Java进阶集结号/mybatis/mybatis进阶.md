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

1. mybatis 动态sql中使用<where>标签与直接写where关键字有什么区别？
2. mybatis 动态sql标签中循环标签中有哪些属性，各自的作用。
3. mybatis 和 hibernate 的区别有哪些？
4. RowBounds是一次性查询全部结果吗？为什么？
5. MyBatis 定义的接口，怎么找到实现的？
6. Mybatis的底层实现原理。
7. Mybatis是如何进行分页的？分页插件的原理是什么？

> Mybatis 使用 RowBounds 对象进行分页，也可以直接编写 sql 实现分页，也可以使用Mybatis 的分页插件。
> 分页插件的原理：实现 Mybatis 提供的接口，实现自定义插件，在插件的拦截方法内拦截待执行的 sql，然后重写 sql。
> 举例：select * from student，拦截 sql 后重写为：select t.* from （select * from student）tlimit 0，10
> 

1. Mybatis执行批量插入，能返回数据库主键列表吗？
2. Mybatis都有哪些Executor执行器？它们之间的区别是什么？
3. Mybatis动态sql有什么用？执行原理？有哪些动态sql？
4. mybatis有几种分页方式？
5. MyBatis框架的优点和缺点
6. 使用MyBatis框架，当实体类中的属性名和表中的字段名不一样 ，怎么办 ？
7. 通常一个Xml映射文件，都会写一个Dao接口与之对应，请问，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗？
8. Xml映射文件中，除了常见的select|insert|updae|delete标签之外，还有哪些标签？
9. 简述Mybatis的插件运行原理，以及如何编写一个插件。
10. +