# Bad Java

## 概述

这里列举了一些可以称之为愚蠢/低效的 java 代码，绝非无中生有，全部是从项目里找到的。我称之为 bad java

在这里示众的目的，是希望每一位读者能吸取教训，在自己的编程实践中避免写出 bad java

## 莫名其妙

这是一些莫名其妙的代码

### if 判断

谁能告诉我这里面的 if 代码块有神马用?

```java
private byte[] getDataFromRedisInBytes(String redisKey, Object... keyArgs) {
    byte[] resultDataInBytes = redisClient.get(String.format(redisKey, keyArgs).getBytes());
    if (resultDataInBytes != null) {
        return resultDataInBytes;
    } else {
        return null;
    }
}
```

## list

### 修改list的元素

18 行是在干嘛？

```java
// 替换被本用户修改的信息
int count = 0;
for (OperateCategoryVO temp : operateCategoryVOList) {

    // 若编辑位是当前用户id，获得当前用户的这条记录的草稿箱信息
    if (temp.getEditor() == userId) {
        CommonDraft<OperateCategoryUpdateBO> draft = draftManager.getDraftForCurrentUser(
                DraftTableEnum.OPERATE_CATEGORY, userId, temp.getId(),
                OperateCategoryUpdateBO.class);
        if (draft != null) {
            OperateCategoryUpdateBO updateBO = draft.getUpdateBO();
            // 使用updateBO的属性覆盖旧属性
            BeanUtils.copyProperties(updateBO, temp);
            // 设置更新时间
            temp.setUpdateTime(draft.getUpdateTime());

            // 覆盖原有vo
            operateCategoryVOList.set(count, temp);
        }
    }
    count++;
}
```

思考一下，下面代码会打印出什么来

```java
List<Album> list = new ArrayList<>();
Album album = new Album();
album.setName("功夫熊猫");
list.add(album);
Album test = list.get(0);
test.setName("大圣归来");
test = new Album();
test.setName("蓝精灵");
System.out.println(list.get(0).getName());
```

### 创建 list

在第 9 行，通过创建一个匿名内部类的方式创建了一个 list，但这样会导致 ide 报警，所以在方法前使用了压制报警的注解 `@SuppressWarnings("serial")`

而且，代码看起来也很臃肿

```java
@SuppressWarnings("serial")
@Override
public void deleteHotword(final int id) {
    long currentUserId = BrowserSession.checkLogin().getUid();
    HotwordEntity hotwordEntity = hotwordMapper.getHotwordDetailById(id);
    if (0 == hotwordEntity.getEditor() || hotwordEntity.getEditor() == currentUserId) {
        hotwordMapper.deleteHotword(id);
        draftManager.deleteDraft(DraftTableEnum.SEARCH_HOTWORD, currentUserId,
                new ArrayList<Integer>() {
                    {
                        add(id);
                    }
                });
    } else {
        throw new BizException(VideoErrorCode.DELETE_HOTWORD_ERROR,
                "该条记录被用户：" + hotwordEntity.getEditor() + "锁定");
    }
}
```

试试下面这种写法

```java
@Override
public void deleteHotword(final int id) {
    long currentUserId = BrowserSession.checkLogin().getUid();
    HotwordEntity hotwordEntity = hotwordMapper.getHotwordDetailById(id);
    if (0 == hotwordEntity.getEditor() || hotwordEntity.getEditor() == currentUserId) {
        hotwordMapper.deleteHotword(id);
        draftManager.deleteDraft(DraftTableEnum.SEARCH_HOTWORD, currentUserId,Arrays.asList(id));
    } else {
        throw new BizException(VideoErrorCode.DELETE_HOTWORD_ERROR,
                "该条记录被用户：" + hotwordEntity.getEditor() + "锁定");
    }
}
```

## log

第 7 行，打印异常堆栈，第 8 行，将异常向外抛出......那么第 7 行的必要性就可以商榷了，会导致什么后果？

* 异常日志会更臃肿
* 如果根据日志文件进行异常数量统计，那么统计出来的数据不准确，虚高

如果想要在这里打印一些调试信息，不应该使用 error 级别，更好的做法是将想要打印的调试信息作为 BizException 的 message 属性

```java
@RequestMapping("/add")
@ResponseBody
public ResultModel addHotword(@Valid @RequestBody HotwordBO hotword) {
    try {
        hotwordService.addHotword(hotword);
    } catch (Exception e) {
        logger.error("addHotword error :", e);
        throw new BizException(1001, e.toString());
    }
    return new ResultModel(true);
}
```

## transaction

数据库事务需要对表/记录加锁，其他需要访问被加锁数据的事务就要等待锁的释放。可见，提高性能的关键是以最快的速度完成事务，要达成这个目的，我们应该在事务之前完成所有前置操作，以及在事务之后完成所有后续操作，而不应将非数据库操作放在事务里。

### 上传文件

注意第 29 行，在事务里上传文件到小文件系统，很明显应该先上传文件获得文件 url，再开始事务

```java
@Override
@Transactional
public void addHomepageRecommend(RecommendBO recommend) throws IOException {
    //判断专题/专辑是否存在
    if (recommend.getEntityType() == EntityTypeEnum.ALBUM.getType()) {
        if (albumMapper.getAlbumDetailById(recommend.getEntityId()) == null) {
            throw new BizException(VideoErrorCode.ALBUM_NOT_EXIST, "专辑不存在！");
        }
    } else if (recommend.getEntityType() == EntityTypeEnum.SUBJECT.getType()) {
        if (subjectMapper.querySubject(recommend.getEntityId()) == null) {
            throw new BizException(VideoErrorCode.SUBJECT_NOT_EXIST, "专题不存在！");
        }
    }

    //如果是专辑，判断该专辑是否已经关联首页推荐
    if (recommend.getEntityType() == EntityTypeEnum.ALBUM.getType()) {
        if (albumMapper.getHomepageAlbumDetail(recommend.getEntityId()) != null) {
            throw new BizException(VideoErrorCode.HOMEPAGE_RECOMMEND_EXIST,
                    "该专辑已经被首页推荐关联，请更换专辑");
        }
    }
    HomepageRecommendEntity entity = new HomepageRecommendEntity();
    AlbumEntity albumEntity = new AlbumEntity();
    BeanUtils.copyProperties(recommend, entity);
    Date date = new Date();
    entity.setCreateTime(date);
    entity.setUpdateTime(date);
    if (null != recommend.getImage()) {
        String dfsFilePath = dfsUtil.store(recommend.getImage());
        entity.setImage(dfsFilePath);
    }
    LOGGER.info("addHomepageRecommend entity : " + JSON.toJSONString(entity));
    if (recommend.getEntityType() == EntityTypeEnum.ALBUM.getType()) {
        AlbumDetail detail = albumMapper.getAlbumDetailByAlbumId(recommend.getEntityId());
        BeanUtils.copyProperties(recommend, albumEntity);
        BeanUtils.copyProperties(detail, albumEntity);
        albumEntity.setOperateCreateTime(date);
        albumEntity.setOperateUpdateTime(date);
        albumEntity.setScore((float) detail.getScore());
        if (detail.getActor().length() >= 64) {
            albumEntity.setActor(detail.getActor().substring(0, 64));
        }
        if (detail.getDescription().length() >= 128) {
            albumEntity.setDescription(detail.getDescription().substring(0, 128));
        }
        String dfsFilePath2 = dfsUtil.store(recommend.getImageFHD());
        albumEntity.setImageFHD(dfsFilePath2);
        albumMapper.addHomepageAlbum(albumEntity);
        entity.setScore(albumEntity.getScore());
    } else {
        entity.setScore((float) 0.0);
    }
    homepageMapper.addHomepageRecommend(entity);
}
```

### 发送 mq 消息

注意最后 2 行，在事务里发送 mq 消息，很明显应该先提交事务，再发送 mq 消息

```java
@Transactional
@Override
public void batchUpdateRecommends(List<IssueParamBO> issueParamList) {
    int currentPageOnShelfIndex = 0;
    List<Integer> idList = new ArrayList<Integer>();
    for (IssueParamBO param : issueParamList) {
        idList.add(param.getId());
        if (param.getStatus() == TVStatusEnum.YES.getStatus()) {
            currentPageOnShelfIndex++;
        }
    }

    int count = homepageMapper.getHomepageRecommendOtherPageOnShelfCount(idList);
    if (count + currentPageOnShelfIndex < 7) {
        throw new BizException(VideoErrorCode.HOMEPAGE_RECOMMEND_ONSHELF_COUNT_LESS,
                "上架首页推荐总数少于7条");
    }

    long userId = BrowserSession.checkLogin().getUid();
    HomepageRecommendEntity homepageRecommendEntity;
    RecommendUpdateBO updateBO;
    AlbumEntity albumEntity;
    List<Integer> deleteIds = new ArrayList<Integer>();
    for (IssueParamBO param : issueParamList) {
        // 获得编辑位信息
        Long editor = draftManager.getDraftEditor(DraftTableEnum.HOMEPAGE_RECOMMEND,
                param.getId());
        //防止发布被删除的记录报错
        if (editor == null) {
            continue;
        }
        // 只能发布自己编辑过的或者未被编辑的
        if (editor == userId) {
            // 获得草稿
            CommonDraft<RecommendUpdateBO> draft = draftManager.getDraftForCurrentUser(
                    DraftTableEnum.HOMEPAGE_RECOMMEND, userId, param.getId(),
                    RecommendUpdateBO.class);
            // 获取草稿updateBO
            updateBO = draft.getUpdateBO();
            // updateBO转成entity
            homepageRecommendEntity = EntityConvertUtil.boToEntity(updateBO,
                    HomepageRecommendEntity.class);
            albumEntity = EntityConvertUtil.boToEntity(updateBO, AlbumEntity.class);
            // 设置正确的albumId
            albumEntity.setId(homepageRecommendEntity.getEntityId());
            // 编辑位复位
            homepageRecommendEntity.setEditor(0);
            // 设置上架还是下架
            homepageRecommendEntity.setStatus(param.getStatus());
            // 设置顺序号
            homepageRecommendEntity.setOrder(param.getOrder());
            // 这里进行专辑图片优化
            if (homepageRecommendEntity.getEntityType() == EntityTypeEnum.ALBUM.getType()) {
                // 使用运营图片替换专辑图片
                albumEntity.setImage(homepageRecommendEntity.getImage());
                // 更换专辑主色调
                albumEntity.setImageColors(
                        ColorPicker.getColor(homepageRecommendEntity.getImage()));
            }
            // 更新基本信息
            LOGGER.info("homepageMapper.updateRecommend");
            homepageMapper.updateRecommend(homepageRecommendEntity);
            albumMapper.updateHomepageAlbum(albumEntity);
            // 更新上下架信息
            LOGGER.info("homepageMapper.issueRecommend");
            homepageMapper.issueRecommend(homepageRecommendEntity);
            // 把id添加到可删除草稿列表
            deleteIds.add(param.getId());
        } else if (editor == 0) {// 直接点击发布的，没存草稿的,也就是只修改上下架和顺序号的
            homepageRecommendEntity = EntityConvertUtil.boToEntity(param,
                    HomepageRecommendEntity.class);
            homepageMapper.issueRecommend(homepageRecommendEntity);
        } else {
            LOGGER.error("首页推荐:" + param.toString() + "，发布失败，当前记录被运营人员" + editor + "编辑锁定");
        }
    }
    // 清除草稿箱内容
    draftManager.deleteDraft(DraftTableEnum.HOMEPAGE_RECOMMEND, userId, deleteIds);

    // 发送mq消息
    metaqMessageSender.send(new HomepageRecommendReq());
    metaqMessageSender.send(new HomepageAlbumReq());
}
```

### 无谓的开启事务

看下面代码，启动事务以后，判断传入参数`videoPackList.size()`，如果为 0 就退出事务......为毛不在事务启动前判断？这样很有可能会启动事务然后啥也不干就退出，对数据库来说开一个事务可不是个小负担

```java
/**
 * 将专辑, 视频入库
 */
@Transactional(propagation = Propagation.REQUIRED)
public SaveInfo save(ChannelVideo channelVideo, AlbumPack albumPack,
                     List<VideoPack> videoPackList) {
    if (videoPackList.size() == 0) {
        return new SaveInfo();
    }
    Long collectCount = 0L;
    Integer currentCount = 0;
    Album album = albumPack.getAlbum();

    ......

}
```

## 参数处理

### by value or by ref

> Java manipulates objects 'by reference,' but it passes object references to methods 'by value.'

来自于 [Does Java pass by reference or pass by value?](https://www.javaworld.com/article/2077424/learn-java/learn-java-does-java-pass-by-reference-or-pass-by-value.html)

先看代码片段 1

这段代码，将 albumPack 的 album 成员交给 youkuStoreHelper，调用 handlerAlbum 进行处理，返回一个新的 album 后，将 albumPack 的 album 成员设置为这个返回值

```java
// 专辑类型,处理描述URL,图片取色
YoukuAlbumDto album = youkuStoreHelper.handleAlbum(albumPack.getAlbum());
albumPack.setAlbum(album);
```

再看看代码片段 2

我们看下 handleAlbum 在做什么

```java
@Override
public YoukuAlbumDto handleAlbum(YoukuAlbumDto album) {
    // 类型标签,
    ShowCategorySearchReq req = new ShowCategorySearchReq();
    req.setShowid(album.getCpAlbumId());
    ShowCategoryRsp rsp = youkuApi.categorySearch(req);
    List<String> categorys = rsp.getShow_genre();
    String category = "";
    for (String tmpCategory : categorys) {
        category += tmpCategory + "/";
    }
    album.setCategory(
            category.length() > 0 ? category.substring(0, category.length() - 1) : "");

    YoukuAlbumDto existAlbum = albumService.getAlbumDetailByCpAlbumId(album.getCpAlbumId());
    if (existAlbum == null) {

        // 专辑描述入dfs
        String description = album.getDescription();
        if (!StringUtil.isEmpty(album.getDescription())) {
            String descUrl = fastDfs.store("txt", description.getBytes());
            album.setDescriptionUrl(descUrl);
            album.setDescription(description.length() > 100
                    ? description.substring(0, 100) + "..." : description);
        } else {
            album.setDescriptionUrl("");
            album.setDescription("");
        }
        album.setImageColors(ColorPicker.getColor(album));
    } else if (StringUtil.isEmpty(existAlbum.getImageColors())) {
        album.setImageColors(ColorPicker.getColor(album));
    }
    return album;
}
```

我们发现 handleAlbum 在对传入的 album 进行了一系列操作后，并没有 new 一个 album 返回，而是直接将这个传入的 album 返回了

那么我不禁要问一下，在代码片段 1 里，将返回的 album 重新 set 到 albumPack 的意义在哪里？

本质上，这个问题和前述里修改 list 的元素 是一样的

## 类型转换

### Object 转 String

把一个对象转换成 String，为毛不用`toString()`方法？

```java
Map<String, Object> simpleAppMap = applicationService.getSimpleAppById(appId);
if (simpleAppMap.containsKey("category_2")) {
    int category2Id = Integer.parseInt(simpleAppMap.get("category_2") + "");
......
```

### 原始数据类型转 String

```java
public MultiPage<Map<String, Object>> recommend(long appId, int sortType, int start, int max) {
    ......

        if (simpleAppId.equals(appId + "")) {
            // except self
            continue;
        }
```



