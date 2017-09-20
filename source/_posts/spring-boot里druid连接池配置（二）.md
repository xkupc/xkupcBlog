---
title: spring-boot里druid连接池配置（二）
date: 2017-09-19 17:33:17
tags: [druid,spring-boot]
categories: java中间件
---
### 简介
上篇我们简单介绍了druid的简单配置，利用druid实现了数据库的监控和配置密码的加密，今天我们从源码的角度分析druid的监控实现。
### 数据库监控配置
在上篇的代码里我们通过springboot自带的Servlet注册Bean进行Servlet注册了StatViewServlet，通过自带的Filter注册bean注册了WebStatFilter，今天我们俩介绍一下这个两个bean的作用。
## StatViewServlet
看看它的配置，我们就知道，它是一个实现druid监控视图的servlet,通过这个servlet我们配置了监控视图web页的登录账号和密码，通过设置黑白名单ip配置web页的访问权限,接下来我们来看看这个配置在哪生效的。
跟踪StatViewServlet的父类ResourceServlet:
```
 private void initAuthEnv() {
        String paramUserName = getInitParameter(PARAM_NAME_USERNAME);
        if (!StringUtils.isEmpty(paramUserName)) {
            this.username = paramUserName;
        }

        String paramPassword = getInitParameter(PARAM_NAME_PASSWORD);
        if (!StringUtils.isEmpty(paramPassword)) {
            this.password = paramPassword;
        }

        String paramRemoteAddressHeader = getInitParameter(PARAM_REMOTE_ADDR);
        if (!StringUtils.isEmpty(paramRemoteAddressHeader)) {
            this.remoteAddressHeader = paramRemoteAddressHeader;
        }
        ....

```
<!--more-->
可以看到通过initAuthEnv完成我们配置参数的初始化工作。接着我们来看看配置怎么生效的，既然是servlet，那必然会有service方法。很快我们在ResourceServlet里找到了service方法，我们来看看druid的登陆：
```
 if ("/submitLogin".equals(path)) {
            String usernameParam = request.getParameter(PARAM_NAME_USERNAME);
            String passwordParam = request.getParameter(PARAM_NAME_PASSWORD);
            if (username.equals(usernameParam) && password.equals(passwordParam)) {
                request.getSession().setAttribute(SESSION_USER_KEY, username);
                response.getWriter().print("success");
            } else {
                response.getWriter().print("error");
            }
            return;
        }
```
这里druid简单的实现了一个登陆操作，将用户名作为一个session存储。当然作为一个内部使用的web页，我们不该有太多的苛求。接下来我们看看这个service还做了什么，我们看到service承担了整个监控视图的所有请求的处理和分发，对于数据库监控数据的请求，比如sql监控数据请求http://localhost:31009/druid/sql.json?orderBy=SQL&orderType=desc&page=1&perPageCount=1000000& 我们看druid是如何处理的呢，我们注意到这行代码：
```
 if (path.contains(".json")) {
            String fullUrl = path;
            if (request.getQueryString() != null && request.getQueryString().length() > 0) {
                fullUrl += "?" + request.getQueryString();
            }
            response.getWriter().print(process(fullUrl));
            return;
        }
```
当所有包含.json请求的都会通过process方法来处理，小编突然觉得druid对url的处理实在是好简单，突然明白为嘛它作为一个数据库连接池，还要搞一个EncodingConvertFilter的过滤器了，大公司套路深，我要回农村。。。。
直接看process方法，我们在StatViewServlet找到了process方法，这个厉害了，druid还可以通过配置jmx连接地址，来监控远程druid服务。也就说我们可以将实现druid连接池的应用和监控视图的web应用分离，毕竟让一个内管后台直接访问生产机器还是有风险的。
继续跟踪我们发现所有的请求都是通过DruidStatService.service(url)进行处理的.我们来看看druid是否处理我们上面那个请求的：
```
  if (url.startsWith("/sql.json")) {
            return returnJSONResult(RESULT_CODE_SUCCESS, getSqlStatDataList(parameters));
        }
```
跟踪getSqlStatDataList方法：
```
 private List<Map<String, Object>> getSqlStatDataList(Map<String, String> parameters) {
        Integer dataSourceId = null;

        String dataSourceIdParam = parameters.get("dataSourceId");
        if (dataSourceIdParam != null && dataSourceIdParam.length() > 0) {
            dataSourceId = Integer.parseInt(dataSourceIdParam);
        }

        List<Map<String, Object>> array = statManagerFacade.getSqlStatDataList(dataSourceId);
        List<Map<String, Object>> sortedArray = comparatorOrderBy(array, parameters);
        return sortedArray;
    }
```
这里有一个dataSourceId请求里是没有的，因为我们使用的是单数据源。
我们看到拿到请求结果后，druid对结果进行了排序和分页，我们学习下是如何排序和分页的。这里给我们一点在内存分页的启示了吧，不过话说页面把记录总数写死的做法这样真的好么，还好有重置选项
```
        // 排序方法
        if (orderBy.trim().length() != 0) {
            Collections.sort(array, new MapComparator<String, Object>(orderBy, ORDER_TYPE_DESC.equals(orderType)));
        }

        // 分页方法
        int fromIndex = (page - 1) * perPageCount;
        int toIndex = page * perPageCount;
        if (toIndex > array.size()) {
            toIndex = array.size();
        }

        return array.subList(fromIndex, toIndex);
```
注意到这里使用了门面模式，通过DruidStatManagerFacade去操作druid的核心类DruidDataSource。
接下来我们来看看druid是如何记录sql执行以及如何存储这些记录的。看一段代码：
```
public List<Map<String, Object>> getSqlStatDataList(Integer dataSourceId) {
        //获取数据源实例
        Set<Object> dataSources = getDruidDataSourceInstances();
        //若数据源id为空，则获取所有数据源数据
        if (dataSourceId == null) {
            List<Map<String, Object>> sqlList = new ArrayList<Map<String, Object>>();

            for (Object datasource : dataSources) {
                sqlList.addAll(getSqlStatDataList(datasource));
            }

            return sqlList;
        }

        for (Object datasource : dataSources) {
            if (dataSourceId != null && dataSourceId.intValue() != System.identityHashCode(datasource)) {
                continue;
            }

            return getSqlStatDataList(datasource);
        }
        // 小编觉得此处的return好像是多余的，但是增加了程序的健壮性
        return new ArrayList<Map<String, Object>>();
    }
```
看到这里我们一定很好奇druid是如何维护这个dataSource的呢？
接下来会比较枯燥的代码，列位看官可以以个舒服的姿势看下去，最好是能有杯茶，放点枸杞是极好的。
我们直接进入DruidDataSource，嗯，这是一个很大很大的类，果然国内人写代码就是跟国外的不一样。
继续我们上面的跟踪。DruidDataSource里维护一个JdbcDataSourceStat的对象dataSourceStat来记录jdbc数据源相关信息。然后在JdbcDataSourceStat里通过LinkedHashMap<String, JdbcSqlStat>来记录sql的执行记录。
```
//初始化大小为16，负载因子为0.75，以插入顺序为排列顺序
sqlStatMap = new LinkedHashMap<String, JdbcSqlStat>(16, 0.75f, false) 
```
```
public DruidDataSource(boolean fairLock){
        super(fairLock);
        //根据系统配置初始化监控内存
        configFromPropety(System.getProperties());
    }
```
现在我们知道了所有的监控记录都是放在内存里的，接下来我们来看看druid是如何监控到sql执行并放到内存里的。
## WebStatFilter
可能大家已经猜到了，既然数据库的状态的变化是由外部请求引起的，那么监听记录的保存肯定是对请求的追踪，接下来我们就看看WebStatFilter是如何跟踪请求执行的。
直接看WebStatFilter的doFilter的方法，druid对请求以及请求的session,webApp的状态，请求url进行了记录，我们没看到sql的相关实体类，进一步看WebRequestStat里有相关jdbc属性记录：
![webRequestStat](/images/druid/druid_webRequestStat.png)
既然有这些属性必然会在执行sql的时候进行相关属性的变更，请求的执行必然会涉及连接池去连接，那么会不会在连接的通过切面之类的完成呢，考虑druid要做兼容，最有可能通过过滤器来做，有了这个思路我们在源码目录里找到了过滤器的目录，看到StatFilter，看类名我们就知道了就是对请求各种监控的更新的过滤器。
## 结语
好了，对于druid的监听就介绍到这里了，里面还有很多细节，在这里就不介绍了，因为。。。。我也没完全懂！！！！！


