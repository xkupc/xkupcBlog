---
title: mybatis里的jpa的方式实现
date: 2017-12-26 16:01:00
tags: [java,spring-boot]
categories: [mybatis,jpa,orm]
---
## 前言
最近看了一些文章讨论JPA和Mybatis有什么区别，哪个更灵活，各自有什么优劣。大佬们说了很多很多的不一样，也对相关业务下的技术选型给出了一些建议。小编因为对jpa和mybatis也就业务层面上的使用而已，没深入研究过，所以也说不上他们到底有啥区别。只是觉得jpa在多表联查的动态sql的实现上实在是太麻烦，当然现在jpa已经支持视图了，对于多表的联合查询我们可以考虑使用建立视图的方式处理，其次就是jpa更新数据使用的save方法总是先查询出记录与入参进行对比，对不一致的值进行更新，在数据量大的情况下，性能应该会比mybatis低一点。jpa对于单表基本增删改查要比mybatis简单很多，jap框架已经封装了很多的操作方法，而mybatis需要我们手写sql，同时使用mybatis的话，一旦数据库表结构发生变更，那数据层的修改将会很麻烦，即便是有像mybatis generator这样的自动生成代码的工具，修改方便的前提是你没有自定的sql语句，但那总是不太可能的。小编习惯用的mybatis，最近使用maven插件里的mybatis generator生成代码的时候，发现了它为了我们提供了一种新的类似于jap的sql实现方式。
## mybatis generator
mybatis generator是通过数据库表结构自动生成mybatis相关的映射文件以及数据实体，maven已经为我们提供了这样的一个插件：
```js
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
             <version>1.3.2</version>
                <configuration>
                    <verbose>true</verbose>
                    <overwrite>true</overwrite>
                </configuration>
        </plugin>
```
<!--more-->
同时我们将generatorConfig.xml放入resource目录下,generatorConfig.xml配置如下：
```js
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <!--数据库驱动-->
    <classPathEntry    location="E:\maven\mavenrepos\mysql\mysql-connector-java\5.1.39\mysql-connector-java-5.1.39.jar"/>
    <context id="DB2Tables"    targetRuntime="MyBatis3">
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <!--数据库链接地址账号密码-->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver" connectionURL="jdbc:mysql://xxxx:3306/xxx" userId="root" password="123456">
        </jdbcConnection>
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>
        <!--生成Model类存放位置-->
        <javaModelGenerator targetPackage="com.upc.customer.model" targetProject="src">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>
        <!--生成映射文件存放位置-->
        <sqlMapGenerator targetPackage="com.upc.customer.mapper" targetProject="src">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
        <!--生成Dao类存放位置-->
        <javaClientGenerator type="XMLMAPPER" targetPackage="com.upc.customer.dao" targetProject="src">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <!--生成对应表及类名-->
        <table tableName="user_account_view" domainObjectName="UserAccountView" enableCountByExample="false" enableUpdateByExample="true"
               enableDeleteByExample="false" enableSelectByExample="true"
               selectByExampleQueryId="true"></table>

    </context>
</generatorConfiguration>
```
执行mvn命令：mvn mybatis-generator:generate,即可生成表结构对应对象实体，mapper接口和xml映射文件。
## mybatis的jpa方式实现
在jpa里对于单表的查询涉及动态sql的，我们一般会使用Predicate完成动态sql的拼装。大家可能很熟悉这样的代码：
```java
private Predicate getInputCondition(UserInfoDTO userInfo) {
		List<BooleanExpression> predicates = new ArrayList<>();
		predicates.add(QUserInfo.userInfo.enableFlag.eq(EnableFlag.Y));
    	if(null != userInfo && userInfo.getUserTypeId() != null && !userInfo.getUserTypeId().equals("")){
			BooleanExpression findByUserType = QUserInfo.userInfo.userTypeId.eq(userInfo.getUserTypeId());
			predicates.add(findByUserType);
		}
		if(null != userInfo && userInfo.getMarketerId() != null && !userInfo.getMarketerId().equals("")){
			BooleanExpression findByUserType = QUserInfo.userInfo.marketerId.eq(userInfo.getMarketerId());
			predicates.add(findByUserType);
		}
		return ExpressionUtils.allOf(predicates.toArray(new BooleanExpression[predicates.size()]));
	}
QUserInfo为jap编译是自动通过数据实体类生成。
```
而在mybatis里我们则可能在mybatis的表映射xml文件里,我们通过条件判断的方式，将动态参数加入到sql里。我们可能写这样的代码：
```js
<select id="findUserInfoById" resultMap="BaseResultMap" parameterType="com.upc.customer.dto.UserInfoDTO">
        SELECT
        <include refid="Base_Column_List"/>
        FROM user_info
        WHERE 1=1
        <if test="status !=null">
            and STATUS =#{status,jdbcType=CHAR}
        </if>
        <if test="enableFlag !=null">
            and ENABLE_FLAG = #{enableFlag,jdbcType=CHAR}
        </if>
        <if test="userPhone != null">
            and USER_PHONE = #{userPhone,jdbcType=VARCHAR}
        </if>
        <if test="plantformId != null">
            and PLANTFORM_ID = #{plantformId,jdbcType=VARCHAR}
        </if>
        <if test="openId != null">
            and open_id = #{openId,jdbcType=VARCHAR}
        </if>
        <if test="customerId != null">
            and CUSTOMER_ID = #{customerId,jdbcType=VARCHAR}
        </if>
        <if test="erpNo != null">
            and ERP_NO = #{erpNo,jdbcType=VARCHAR}
        </if>
        <if test="userId != null">
            and id = #{userId,jdbcType=VARCHAR}
        </if>
        <if test="userRank !=null">
            and USER_RANK = #{userRank,jdbcType=VARCHAR}
        </if>
        limit 1
    </select>
```
现在mybatis-generator为我们生成了类似jpa里QUserInfo这样的实体类，那就是后缀为Example实体类，如上面的mybatis-generator配置，我们生成了UserAccountViewExample的实体类。
这个类有三个属性
```java
    //排序字段
    protected String orderByClause;
    //是否去重
    protected boolean distinct;
    //查询条件，Criteria为UserAccountViewExample的内部类
    protected List<Criteria> oredCriteria;
```
我们来看看Criteria是怎么将属性值动态拼接成sql的。Criteria实现抽象类GeneratedCriteria，我们看到抽象类里每一个属性都对应着这么多个方法：
```java
  public Criteria andIdIsNull() {
            addCriterion("ID is null");
            return (Criteria) this;
        }

        public Criteria andIdIsNotNull() {
            addCriterion("ID is not null");
            return (Criteria) this;
        }

        public Criteria andIdEqualTo(String value) {
            addCriterion("ID =", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdNotEqualTo(String value) {
            addCriterion("ID <>", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdGreaterThan(String value) {
            addCriterion("ID >", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdGreaterThanOrEqualTo(String value) {
            addCriterion("ID >=", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdLessThan(String value) {
            addCriterion("ID <", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdLessThanOrEqualTo(String value) {
            addCriterion("ID <=", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdLike(String value) {
            addCriterion("ID like", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdNotLike(String value) {
            addCriterion("ID not like", value, "id");
            return (Criteria) this;
        }

        public Criteria andIdIn(List<String> values) {
            addCriterion("ID in", values, "id");
            return (Criteria) this;
        }

        public Criteria andIdNotIn(List<String> values) {
            addCriterion("ID not in", values, "id");
            return (Criteria) this;
        }

        public Criteria andIdBetween(String value1, String value2) {
            addCriterion("ID between", value1, value2, "id");
            return (Criteria) this;
        }

        public Criteria andIdNotBetween(String value1, String value2) {
            addCriterion("ID not between", value1, value2, "id");
            return (Criteria) this;
        }

         protected void addCriterion(String condition, Object value1, Object value2, String property) {
            if (value1 == null || value2 == null) {
                throw new RuntimeException("Between values for " + property + " cannot be null");
            }
            criteria.add(new Criterion(condition, value1, value2));
        }

```
每一个条件都是一个Criterion，多个Criterion通过and合成一个Criteria，若条件中有or则可以封装多个Criteria，通过Criteria.or()方法连接。我们来看看多个Criteria是如何在xml映射成sql的
CriterionUserAccountViewExample的内部类，其构造方法：
```java
public static class Criterion {
        //条件
        private String condition;
        //参数值
        private Object value;
        //between涉及多个参数值
        private Object secondValue;
        //判断条件是否有传参值 如is not null
        private boolean noValue;
        //判断条件是否有一个传参值 如=,like
        private boolean singleValue;
        //判断条件是否有是有两个传参值 如between
        private boolean betweenValue;
        //判断条件是是否有多个传参值 如In，not In
        private boolean listValue;

        private String typeHandler;

        public String getCondition() {
            return condition;
        }

        public Object getValue() {
            return value;
        }

        public Object getSecondValue() {
            return secondValue;
        }

        public boolean isNoValue() {
            return noValue;
        }

        public boolean isSingleValue() {
            return singleValue;
        }

        public boolean isBetweenValue() {
            return betweenValue;
        }

        public boolean isListValue() {
            return listValue;
        }

        public String getTypeHandler() {
            return typeHandler;
        }

        protected Criterion(String condition) {
            super();
            this.condition = condition;
            this.typeHandler = null;
            this.noValue = true;
        }

        protected Criterion(String condition, Object value, String typeHandler) {
            super();
            this.condition = condition;
            this.value = value;
            this.typeHandler = typeHandler;
            if (value instanceof List<?>) {
                this.listValue = true;
            } else {
                this.singleValue = true;
            }
        }

        protected Criterion(String condition, Object value) {
            this(condition, value, null);
        }

        protected Criterion(String condition, Object value, Object secondValue, String typeHandler) {
            super();
            this.condition = condition;
            this.value = value;
            this.secondValue = secondValue;
            this.typeHandler = typeHandler;
            this.betweenValue = true;
        }

        protected Criterion(String condition, Object value, Object secondValue) {
            this(condition, value, secondValue, null);
        }
    }
```
xml：
```js
 <sql id="Example_Where_Clause" >
    <where >
      <foreach collection="oredCriteria" item="criteria" separator="or" >
        <if test="criteria.valid" >
          <trim prefix="(" suffix=")" prefixOverrides="and" >
            <foreach collection="criteria.criteria" item="criterion" >
              <choose >
                <when test="criterion.noValue" >
                  and ${criterion.condition}
                </when>
                <when test="criterion.singleValue" >
                  and ${criterion.condition} #{criterion.value}
                </when>
                <when test="criterion.betweenValue" >
                  and ${criterion.condition} #{criterion.value} and #{criterion.secondValue}
                </when>
                <when test="criterion.listValue" >
                  and ${criterion.condition}
                  <foreach collection="criterion.value" item="listItem" open="(" close=")" separator="," >
                    #{listItem}
                  </foreach>
                </when>
              </choose>
            </foreach>
          </trim>
        </if>
      </foreach>
    </where>
  </sql>
```
jpa大抵也是这个套路吧，可能未来的趋势就是数据层的东西越来越抽象，抽象到对于开发者来说是透明的，开发者根本不用关心sql要怎么写，给出参数，有框架自动生成动态sql吧。
