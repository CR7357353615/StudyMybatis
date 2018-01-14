# Mybatis动态复杂sql
---
## 1.selectKey标签
###### 插入后返回主键
```java
public class User {  
    private int userId;  
    private String userName;  
    private String password;  
    private String comment;  
    //setter and getter  
}  
```
```xml
<insert id="insertAndGetId" useGeneratedKeys="true" keyProperty="userId" parameterType="com.chenzhou.mybatis.User">  
    insert into user(userName,password,comment)  
    values(#{userName},#{password},#{comment})  
</insert>  
```

属性 |	描述 |取值
---- | ----- | ----
useGeneratedKeys|设置是否使用JDBC的getGenereatedKeys方法获取主键并赋值到keyProperty设置的领域模型属性中。MySQL和SQLServer执行auto-generated key field，因此当数据库设置好自增长主键后，可通过JDBC的getGeneratedKeys方法获取。|false（默认），true
keyProperty |	selectKey | 对应java对象中的属性名	 
resultType |	生成结果类型，MyBatis 允许使用基本的数据类型，包括String、int类型。|	 
order	| 1：BEFORE，会先选择主键，然后设置keyProperty，再执行insert语句；|BEFORE
--|2：AFTER，就先运行insert语句再运行selectKey 语句。	|AFTER
statementType|	MyBatis 支持STATEMENT，PREPARED和CALLABLE的语句形式， 对应Statement，PreparedStatement 和CallableStatement响应|STATEMENT
--|--|PREPARED
--|--|CALLABLE
## 2.if标签
### 不使用< where>标签
##### 如果使用以下这种形式，如果name不为空，则会拼接成一个 where AND name = '张三'这种。这样的语法是错误的，所以需要在前面加一个1=1.
```xml
<select>
select id,name
from user
where 1 = 1
<if test="name != null and name != ' ' ">  
    AND name = #{name, jdbcType=VARCHAR}  
</if>  
</select>
```
### 使用< where>标签
##### 如果使用了where标签，则会自动将第一个AND，OR等去掉，不会有语法错误。
```xml
<select>
select id,name
from user
<where>
<if test="name != null and name != ' ' ">  
    AND name = #{name, jdbcType=VARCHAR}  
</if>  
</select>
```
## 3.if+set标签
##### 在< update>语句中，使用< if>可以将null值排除，使用< set>可以自动拼接set语句，并且剔除掉末尾不相关的逗号。
```xml
<update id="updateStudent_if_set" parameterType="liming.student.manager.data.model.StudentEntity">  
    UPDATE STUDENT_TBL  
    <set>  
        <if test="studentName != null and studentName != '' ">  
            STUDENT_TBL.STUDENT_NAME = #{studentName},  
        </if>  
        <if test="studentSex != null and studentSex != '' ">  
            STUDENT_TBL.STUDENT_SEX = #{studentSex},  
        </if>  
        <if test="studentBirthday != null ">  
            STUDENT_TBL.STUDENT_BIRTHDAY = #{studentBirthday},  
        </if>  
        <if test="studentPhoto != null ">  
            STUDENT_TBL.STUDENT_PHOTO = #{studentPhoto, javaType=byte[], jdbcType=BLOB, typeHandler=org.apache.ibatis.type.BlobTypeHandler},  
        </if>  
        <if test="classId != '' ">  
            STUDENT_TBL.CLASS_ID = #{classId}  
        </if>  
        <if test="placeId != '' ">  
            STUDENT_TBL.PLACE_ID = #{placeId}  
        </if>  
    </set>  
    WHERE STUDENT_TBL.STUDENT_ID = #{studentId};      
</update>  
```
## 4.trim标签
属性|作用
--|--
prefix|在trim标签内sql语句加上前缀
suffix|在trim标签内sql语句加上后缀
prefixOverrides|指定去除多余的前缀内容
suffixOverrides|指定去除多余的后缀内容
```xml
<select>
select id,name
from user
where 1 = 1
<if test="name != null and name != ' ' ">  
    AND name = #{name, jdbcType=VARCHAR}  
</if>  
</select>
```
```xml
<update id="updateStudent_if_set" parameterType="liming.student.manager.data.model.StudentEntity">  
    UPDATE STUDENT_TBL  
    <set>  
        <if test="studentName != null and studentName != '' ">  
            STUDENT_TBL.STUDENT_NAME = #{studentName},  
        </if>  
        <if test="studentSex != null and studentSex != '' ">  
            STUDENT_TBL.STUDENT_SEX = #{studentSex},  
        </if>  
        <if test="studentBirthday != null ">  
            STUDENT_TBL.STUDENT_BIRTHDAY = #{studentBirthday},  
        </if>  
        <if test="studentPhoto != null ">  
            STUDENT_TBL.STUDENT_PHOTO = #{studentPhoto, javaType=byte[], jdbcType=BLOB, typeHandler=org.apache.ibatis.type.BlobTypeHandler},  
        </if>  
        <if test="classId != '' ">  
            STUDENT_TBL.CLASS_ID = #{classId}  
        </if>  
        <if test="placeId != '' ">  
            STUDENT_TBL.PLACE_ID = #{placeId}  
        </if>  
    </set>  
    WHERE STUDENT_TBL.STUDENT_ID = #{studentId};      
</update>  
```
以上代码可以改写为：
```xml
<select>
select id,name
from user
<trim prefix="WHERE" prefixOverrides="AND|OR">
    <if test="name != null and name != ' ' ">  
        AND name = #{name, jdbcType=VARCHAR}  
    </if>  
<trim>
</select>
```
```xml
<update id="updateStudent_if_set" parameterType="liming.student.manager.data.model.StudentEntity">  
    UPDATE STUDENT_TBL  
    <trim prefix="SET" suffixOverrides=",">  
        <if test="studentName != null and studentName != '' ">  
            STUDENT_TBL.STUDENT_NAME = #{studentName},  
        </if>  
        <if test="studentSex != null and studentSex != '' ">  
            STUDENT_TBL.STUDENT_SEX = #{studentSex},  
        </if>  
        <if test="studentBirthday != null ">  
            STUDENT_TBL.STUDENT_BIRTHDAY = #{studentBirthday},  
        </if>  
        <if test="studentPhoto != null ">  
            STUDENT_TBL.STUDENT_PHOTO = #{studentPhoto, javaType=byte[], jdbcType=BLOB, typeHandler=org.apache.ibatis.type.BlobTypeHandler},  
        </if>  
        <if test="classId != '' ">  
            STUDENT_TBL.CLASS_ID = #{classId}  
        </if>  
        <if test="placeId != '' ">  
            STUDENT_TBL.PLACE_ID = #{placeId}  
        </if>  
    </trim>  
    WHERE STUDENT_TBL.STUDENT_ID = #{studentId};      
</update>  
```
## 5.choose(when,otherwise)
##### when和otherwise就像java中的 if if else
```xml
<select id="getStudentList_choose" resultMap="resultMap_studentEntity" parameterType="liming.student.manager.data.model.StudentEntity">  
    SELECT ST.STUDENT_ID,  
           ST.STUDENT_NAME,  
           ST.STUDENT_SEX,  
           ST.STUDENT_BIRTHDAY,  
           ST.STUDENT_PHOTO,  
           ST.CLASS_ID,  
           ST.PLACE_ID  
      FROM STUDENT_TBL ST   
    <where>  
        <choose>  
            <when test="studentName !=null ">  
                ST.STUDENT_NAME LIKE CONCAT(CONCAT('%', #{studentName, jdbcType=VARCHAR}),'%')  
            </when >  
            <when test="studentSex != null and studentSex != '' ">  
                AND ST.STUDENT_SEX = #{studentSex, jdbcType=INTEGER}  
            </when >  
            <when test="studentBirthday != null ">  
                AND ST.STUDENT_BIRTHDAY = #{studentBirthday, jdbcType=DATE}  
            </when >  
            <when test="classId != null and classId!= '' ">  
                AND ST.CLASS_ID = #{classId, jdbcType=VARCHAR}  
            </when >  
            <when test="classEntity != null and classEntity.classId !=null and classEntity.classId !=' ' ">  
                AND ST.CLASS_ID = #{classEntity.classId, jdbcType=VARCHAR}  
            </when >  
            <when test="placeId != null and placeId != '' ">  
                AND ST.PLACE_ID = #{placeId, jdbcType=VARCHAR}  
            </when >  
            <when test="placeEntity != null and placeEntity.placeId != null and placeEntity.placeId != '' ">  
                AND ST.PLACE_ID = #{placeEntity.placeId, jdbcType=VARCHAR}  
            </when >  
            <when test="studentId != null and studentId != '' ">  
                AND ST.STUDENT_ID = #{studentId, jdbcType=VARCHAR}  
            </when >  
            <otherwise>  
                1=1
            </otherwise>  
        </choose>  
    </where>    
</select>
```
## 6.foreach
##### foreach主要用在构建in条件中，他可以在SQL语句中进行迭代一个集合。
属性|意义
--|--
collection|传入的集合
item|表示集合中每一个元素进行迭代时的别名
index|表示迭代的位置
open|语句以什么开始
separator|迭代之间以什么符号作为分隔符
close|表示以什么结束
##### collection属性有三种情况
##### 1.传入一个但参数，且参数类型是List，collection的值为list
##### 2.传入的是单参数并且类型是数组，collection的值为array
##### 3.传入的参数是多个的时候，需要封装成一个map

### 1.单参数List类型
```xml
<select id="dynamicForeachTest" parameterType="java.util.List" resultType="Blog">
    select * from t_blog where id in
    <foreach collection="list" index="index" item="item" open="(" separator="," close=")">
            #{item}       
    </foreach>
</select>
```
### 2.单参数array数组的类型
```xml
<select id="dynamicForeach2Test" parameterType="java.util.ArrayList" resultType="Blog">
     select * from t_blog where id in
     <foreach collection="array" index="index" item="item" open="(" separator="," close=")">
          #{item}
     </foreach>
</select>
```
### 3.自己把参数封装成Map的类型
```xml
<select id="dynamicForeach3Test" parameterType="java.util.HashMap" resultType="Blog">
     select * from t_blog where title like "%"#{title}"%" and id in
     <foreach collection="ids" index="index" item="item" open="(" separator="," close=")">
           #{item}
     </foreach>
</select>
```
