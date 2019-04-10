# Spring AOP 实战经验

最近把项目里用到 redis 的部分用 aop 进行重构，记录下来以后回顾

## 重构前

先看下重构前的代码

```java
/**
 * 获取栏目下的专辑
 * 
 * @return
 */
@RequestMapping("category_albums.do")
@ResponseBody
public Result listCategoryAlbums(@RequestParam int categoryId, @RequestParam int page) {
    //防止客户端忘记传page的值，默认1
    if (page <= 0) {
        page = Constant.DEFAULT_PAGE;
    }

    //准备数据
    String redisKey = RedisKeys.TV_VIDEO_CATEGORY_ALBUM_DETAIL;
    int expireTime = CommonConstants.API_CACHE_EXPIRE_INTERVAL;
    // 从redis中获得对象实体
    AlbumsVo albumData = redisManager.getDataFromRedis(AlbumsVo.class, redisKey, categoryId,
            page);
    if (null == albumData) {
        albumData = albumService.getCatagoryAlbumsData(categoryId, page,
                CommonConstants.PAGENUM);
        redisManager.setDataToRedisAsync(albumData, expireTime, redisKey, categoryId, page);
    } else {
        albumBlockedCache.clearBlocked(albumData.getAlbumList());
    }

    return new Result(albumData);
}
```

解析

这段代码很简单，向客户端返回某个栏目下的专辑，处理逻辑

* 首先到 redis 里获取结果，如果命中的话，直接返回给客户端
* 如果未命中，则到数据库里获取结果，将数据库里获取到的结果缓存到 redis，并返回给客户端

很多接口都是类似的处理逻辑，再看一个例子

```java
/**
 * 通过热词获取搜索结果
 * 
 * @param hotword
 * @param type
 * @param page
 */
@RequestMapping("hotword_search.do")
@ResponseBody
public Result hotwordSearch(@RequestParam String hotword, @RequestParam int type,
                            @RequestParam int page) {
    //防止客户端忘记传page的值，默认1
    if (page <= 0) {
        page = Constant.DEFAULT_PAGE;
    }

    //准备数据
    String redisKey = RedisKeys.TV_VIDEO_HOTWORD_SREACH;
    int expireTime = CommonConstants.API_CACHE_EXPIRE_INTERVAL;
    HotwordSearchVo searchData = redisManager.getDataFromRedis(HotwordSearchVo.class,
            redisKey, hotword, type, page);
    if (null == searchData) {
        searchData = hotwordService.getKeywordSearchResult(hotword, type, page);
        // 创建一个单个数据存入redis的异步任务
        redisManager.setDataToRedisAsync(searchData, expireTime, redisKey, hotword, type, page);
    }

    return new Result(searchData);
}
```

## 重构思路

通过 spring 的 aop 技术，拦截从数据库获取结果的方法，在从数据库获取结果之前，先从 redis 获取，如果 redis 命中，则直接返回；否则就继续执行从数据库获取的方法，将返回值缓存到 reids 并返回

实际上不限于从数据库获取结果，例如上例的`hotwordSearch()`，实际上并非从数据库获取结果，是调用的 cp 的接口获取的结果

## 重构步骤

### 标记要拦截的方法

#### 自定义注解

显然用注解是个很不错的想法，定义如下的注解

```java
/**
 * <pre>
 *  这个标注用来为redis的<b>通用化</b>存取设定参数
 * </pre>
 * 
 * @author zhangyan
 * @date 2016年3月29日
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Redis {

    /** 非null值 默认过期时间 **/
    int DEFAULT_TTL = 4 * 60 * 60 + 10 * 60;

    /** null值 默认过期时间 **/
    int NULL_TTL    = 5 * 60;


    /** redsi key，见 {@link RedisKeys} **/
    String value();


    /**
     * <pre>
     *  指示方法的哪些参数用来构造key，及其顺序(编号由0开始)
     *  
     *  示例
     *      keyArgs = {1,0,2}，表示用方法的第二，第一，第三个参数，按顺序来构造key
     *  
     *  默认值的意思是方法的前 n 个参数来构造key，n 最大为10
     *  这样如果构造 key 的参数不多于 10 个且顺序也和方法参数一致，则可以用默认值
     * </pre>
     */
    int[] keyArgs() default { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };


    /** 执行何种操作，默认是先访问 redis **/
    RedisAction action() default RedisAction.REDIS_FIRST;


    /** 过期时间，默认250分钟 **/
    int ttl() default DEFAULT_TTL;


    /** 是否以同步的方式操作redis，默认是false **/
    boolean sync() default false;


    /** 是否要缓存 null 值，默认为true **/
    boolean cacheNull() default true;


    /** 如果要缓存null值，过期时间是多少，默认5分钟 **/
    int nullTtl() default NULL_TTL;

}
```

这个注解用在需要拦截的方法上，还附带了一些元信息

#### 在目标方法上使用注解

```java
@Override
@Redis(value = RedisKeys.TV_VIDEO_FILTER_RESULT, keyArgs = { 0, 5, 4, 1 })
public AlbumsVo filterChannelAlbums(int channelId, int page, int pageNum,
                                    List<Integer> optionList, int orderBy, String optionStr) {
    return _filterChannelAlbums(channelId, page, pageNum, optionList, orderBy, optionStr);
}
```

### 编写拦截器

```java
/**
 * <pre>
 *  拦截db操作，并到redis里进行同样的操作
 * </pre>
 * 
 * @author zhangyan
 * @date 2016年4月13日
 */
@Aspect
@Component
public class RedisInterceptor {

    private static final Logger LOG = Logger.getLogger(RedisInterceptor.class);

    @Resource
    private RedisHelper         redisHelper;


    @Around("@annotation(redis)")
    public Object doAround(ProceedingJoinPoint pjp, Redis redis) throws Throwable {

        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();

        // 是否穿透 redis
        boolean stab = redis.action() == RedisAction.STAB_REDIS;

        Object[] keyArgs = getKeyArgs(method, pjp.getArgs(), redis.keyArgs());
        String key = keyArgs == null ? redis.value() : String.format(redis.value(), keyArgs);

        if (keyArgs == null || keyArgs.length == 0 || keyArgs[0] == null) {
            LOG.warn("key args is empty,key=" + key);
        }

        Class<?> returnType = method.getReturnType();
        Object result = stab ? null : get(key, returnType);
        if (result == null) {
            result = pjp.proceed();
            if (result != null) {
                setex(key, redis.ttl(), result, redis.sync());
            } else if (redis.cacheNull()) {
                setex(key, redis.nullTtl(), result, redis.sync());
            }
        }
        return result;
    }


    /**
     * 获取构造redis的key的参数数组
     * 
     * @param method
     * @param args
     * @param keyArgs
     * @return
     */
    private Object[] getKeyArgs(Method method, Object[] args, int[] keyArgs) {

        Object[] redisKeyArgs;
        int len = keyArgs.length;
        if (len == 0) {
            return null;
        } else {
            len = min(len, args.length);
            redisKeyArgs = new Object[len];
            int i = 0;
            for (int n : keyArgs) {
                redisKeyArgs[i++] = args[n];
                if (i >= len) {
                    break;
                }
            }
            return redisKeyArgs;
        }
    }


    private int min(int i, int j) {
        return i > j ? j : i;
    }


    private <T> void setex(final String key, final int ttl, final T data, boolean sync) {
        try {
            redisHelper.setex(key, ttl, data, sync);
        } catch (Exception e) {
            LOG.error("redis set error:" + e.getMessage(), e);
        }
    }


    private <T> T get(String key, Class<T> clazz) {
        try {
            return redisHelper.get(key, clazz);
        } catch (Exception e) {
            LOG.error("redis get error:" + e.getMessage(), e);
            return null;
        }
    }
}
```

### 配置 spring 使拦截生效

```markup
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:task="http://www.springframework.org/schema/task"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
     http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
     http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-4.0.xsd">

    <context:component-scan base-package="com.meizu.flymetv.video" />

    <!-- 开启注解式 AOP -->
    <aop:aspectj-autoproxy proxy-target-class="false" />

    <import resource="classpath*:config/video-redis.xml" />
    <import resource="classpath*:config/video-metaq.xml" />

    <task:executor id="executor" pool-size="5-50"
        queue-capacity="6000" />
</beans>
```

### 重构原代码

现在，拦截器承包了对 redis 的操作......，业务代码里就不需要那些相关的代码了

```java
/**
 * 获取栏目下的专辑
 * 
 * @return
 */
@RequestMapping("category_albums.do")
@ResponseBody
public Result listCategoryAlbums(@RequestParam int categoryId, @RequestParam int page) {
    //防止客户端忘记传page的值，默认1
    if (page <= 0) {
        page = Constant.DEFAULT_PAGE;
    }

    AlbumsVo albumData = albumService.getCatagoryAlbumsData(categoryId, page,
            CommonConstants.PAGE_SIZE);
    albumBlockedCache.clearBlocked(albumData.getAlbumList());
    return new Result(albumData);
}
```

......

```java
/**
 * 通过热词获取搜索结果
 * 
 * @param hotword
 * @param type
 * @param page
 */
@RequestMapping("hotword_search.do")
@ResponseBody
public Result hotwordSearch(@RequestParam String hotword, @RequestParam int type,
                            @RequestParam int page) {
    //防止客户端忘记传page的值，默认1
    if (page <= 0) {
        page = Constant.DEFAULT_PAGE;
    }

    return new Result(hotwordService.getKeywordSearchResult(hotword, type, page));
}
```

和重构前的代码相比，简洁多了，整个世界都干净了......

### redis 穿透

通常情况下，都是优先从 redis 里查询结果。但也有时候需要穿透 redis，到 db 里去获取结果的。

例如，在运营后台修改了栏目的配置条件，这样栏目下的内容就会发生变化，此时数据库里的数据和缓存就不同步了，这时的逻辑应该是从数据库查询数据，并刷新缓存里的数据

`@Redis`注解也支持这种操作，只需要设置 action 属性为 `STAB_REDIS` 即可

但是，同一个方法，action 要么是 `STAB_REDIS`，要么是 `REDIS_FIRST`，拦截器只能实现其中一种操作，如何才能让拦截器拦截同一个方法时，实现不同的 redis 操作呢？

有以下几个办法

1. 方法的参数里添加一个专门的变量，用来告诉拦截器做何种操作
2. 复制该方法为另一个方法，2 个方法作用一样，注解也一样，区别是注解的 action 属性不同
3. 同上，但是通过方法名来区别

经过考虑，最终采用了第 2 种办法，如下

```java
public interface AlbumService {

    AlbumsVo getCatagoryAlbumsData(int categoryId, int page, int pageNum);


    /** 穿透到 db 获取栏目下专辑 **/
    AlbumsVo getCatagoryAlbumsDataDirect(int categoryId, int page, int pageNum);


    AlbumDetailsData getAlbumDetailsData(int albumId);


    /** 穿透到 db 获取专辑详情 **/
    AlbumDetailsData getAlbumDetailsDataDirect(int albumId);

}
```

该接口的实现类如下

```java
@Service("albumService")
public class AlbumServiceImpl implements AlbumService {

    @Resource
    private AlbumMapper        albumMapper;

    @Resource
    private MetaqMessageSender metaqMessageSender;


    private AlbumsVo _getCatagoryAlbumsData(int categoryId, int page, int pageNum) {
        int count = albumMapper.getCategoryAlbumsCount(categoryId);
        int startIndex = (page - 1) * pageNum;
        int offset = pageNum;

        List<AlbumListVo> albums = albumMapper.listCategoryAlbums(categoryId, startIndex, offset);
        AlbumsVo data = new AlbumsVo();
        data.setAlbumList(albums);
        data.setPage(page);
        if (page * pageNum >= count) {
            data.setLast(true);
        }
        return data;
    }


    @Override
    @Redis(RedisKeys.TV_VIDEO_ALBUM_DETAIL)
    public AlbumDetailsData getAlbumDetailsData(int albumId) {
        return _getAlbumDetailsData(albumId);
    }


    @Override
    @Redis(action = RedisAction.STAB_REDIS, sync = true, cacheNull = false, value = RedisKeys.TV_VIDEO_ALBUM_DETAIL)
    public AlbumDetailsData getAlbumDetailsDataDirect(int albumId) {
        return _getAlbumDetailsData(albumId);
    }


    @Override
    @Redis(RedisKeys.TV_VIDEO_CATEGORY_ALBUM_DETAIL)
    public AlbumsVo getCatagoryAlbumsData(int categoryId, int page, int pageNum) {
        return _getCatagoryAlbumsData(categoryId, page, pageNum);
    }


    @Override
    @Redis(action = RedisAction.STAB_REDIS, sync = true, cacheNull = false, value = RedisKeys.TV_VIDEO_CATEGORY_ALBUM_DETAIL)
    public AlbumsVo getCatagoryAlbumsDataDirect(int categoryId, int page, int pageNum) {
        return _getCatagoryAlbumsData(categoryId, page, pageNum);
    }

}
```

2 个方法，都是从数据库里查询栏目下的专辑，但其注解的 action 属性不同，就导致一个用于从缓存查询数据，加快访问速度；另一个则适合于用 db 里的数据刷新 redis 的场合

## 总结

在 spring 配置文件里开启注解式 aop，使用如下配置

```markup
<aop:aspectj-autoproxy proxy-target-class="false" />
```

自定义注解，如下2个元注解必须

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
```

针对自定义的注解编写拦截器代码并配置拦截器，注意如下几个注解的使用

```java
@Aspect
@Component
@Around("@annotation(******)")
```



