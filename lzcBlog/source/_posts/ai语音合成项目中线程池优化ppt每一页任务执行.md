---
title: ai语音合成项目中线程池优化ppt每一页任务执行
date: 2025-01-15 21:55:44
tags: 线程池
categories: 业务场景
---

**在最近做的ai语音合成项目中，有一个需求是：用户上传ppt，用ppt的url请求创建ppt讲解合成任务，这里需要关联用户和ppt，还需要下载解析ppt，拿到每一页做处理，并解析出每一页的内容向mq发送消息异步请求python大模型，还需做任务，ppt详情入库操作。**

这里一个ppt会有很多页，每一页都需要做两次数据库操作加mq消息发送，如果页码较多，那响应速度会很慢，所以这里引入一个自定义的单例线程池bean，通过线程池来并发执行每一页的任务，加速任务的执行，并且每页的任务异步执行，不阻塞主线程，且能减少线程开销，控制线程资源。

```java
package com.soundmentor.soundmentorweb.service.impl;

import com.alibaba.fastjson.JSON;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.soundmentor.soundmentorbase.enums.TaskStatusEnum;
import com.soundmentor.soundmentorbase.enums.TaskTypeEnum;
import com.soundmentor.soundmentorbase.utils.PPTXUtil;
import com.soundmentor.soundmentorpojo.DO.TaskDO;
import com.soundmentor.soundmentorpojo.DO.UserPptDetailDO;
import com.soundmentor.soundmentorpojo.DO.UserPptRelDO;
import com.soundmentor.soundmentorpojo.DTO.ppt.PPTPageSummaryTaskDTO;
import com.soundmentor.soundmentorpojo.DTO.task.CreatePPTSummaryTaskParam;
import com.soundmentor.soundmentorweb.mapper.TaskMapper;
import com.soundmentor.soundmentorweb.mapper.UserPptDetailMapper;
import com.soundmentor.soundmentorweb.mapper.UserPptRelMapper;
import com.soundmentor.soundmentorweb.service.PPTService;
import com.soundmentor.soundmentorweb.service.UserInfoApi;
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.xslf.usermodel.XMLSlideShow;
import org.apache.poi.xslf.usermodel.XSLFSlide;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.annotation.Resource;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ThreadPoolExecutor;

@Service
@Slf4j

public class PPTServiceImpl implements PPTService {
    @Resource
    private  UserPptDetailMapper userPptDetailMapper;
    @Resource
    private  UserPptRelMapper userPptRelMapper;
    @Resource
    private  UserInfoApi userInfoApi;
    @Resource(name = "task-thread-pool-executor")
    private  ThreadPoolExecutor threadPoolExecutor;
    @Autowired
    private TaskMapper taskMapper;

    @Override
    public Integer createPPTSummary(CreatePPTSummaryTaskParam param) {
        String pptUrl = param.getPptUrl();
        XMLSlideShow xmlSlideShow = PPTXUtil.loadPPTX(pptUrl);
        List<XSLFSlide> slides = xmlSlideShow.getSlides();
        UserPptRelDO userPptRelDO = userPptRelMapper.selectOne(new LambdaQueryWrapper<UserPptRelDO>()
                .eq(UserPptRelDO::getPptUrl, pptUrl)
                .eq(UserPptRelDO::getUserId, userInfoApi.getUser().getId()));
        if (Objects.isNull(userPptRelDO))
        {
            userPptRelDO = new UserPptRelDO();
            userPptRelDO.setPptUrl(pptUrl);
            userPptRelDO.setUserId(userInfoApi.getUser().getId());
            userPptRelDO.setPageCount(slides.size());
            userPptRelDO.setCreateTime(LocalDateTime.now());
            userPptRelMapper.insert(userPptRelDO);
        }
        for (int i = 0; i < slides.size(); i++) {
            Integer userPptId = userPptRelDO.getId();
            Integer page = i;
            XSLFSlide slide = slides.get(page);
            threadPoolExecutor.execute(()->{
                try {
                    taskExec(userPptId,page,slide);
                } catch (Exception e) {
                    log.error("ppt{}的{}页任务执行失败,请重试",userPptId,page);
                }
            });
        }
        return userPptRelDO.getId();
    }

    /**
     * ppt页生成讲解任务执行
     * @param userPptId ppt标识
     * @param page 页码
     * @param slide ppt页对象
     */
    @Transactional
    public void taskExec(Integer userPptId, Integer page,XSLFSlide slide)
    {
        UserPptDetailDO userPptDetailDO = userPptDetailMapper.selectOne(new LambdaQueryWrapper<UserPptDetailDO>()
                .eq(UserPptDetailDO::getUserPptId, userPptId)
                .eq(UserPptDetailDO::getPptPage, page));
        if(Objects.isNull(userPptDetailDO))
        {
            userPptDetailDO = new UserPptDetailDO();
            userPptDetailDO.setUserPptId(userPptId);
            userPptDetailDO.setPptPage(page);
            userPptDetailDO.setCreateTime(LocalDateTime.now());
            userPptDetailMapper.insert(userPptDetailDO);
        }
        TaskDO taskDO = new TaskDO();
        PPTPageSummaryTaskDTO pptPageSummaryTaskDTO = new PPTPageSummaryTaskDTO();
        pptPageSummaryTaskDTO.setUserPptId(userPptId);
        pptPageSummaryTaskDTO.setPage(page);
        pptPageSummaryTaskDTO.setContent(PPTXUtil.getSlideInfo(slide));
        taskDO.setTaskDetail(JSON.toJSONString(pptPageSummaryTaskDTO));
        taskDO.setType(TaskTypeEnum.PPT_SUMMARY.getCode());
        taskDO.setUpdateTime(LocalDateTime.now());
        taskDO.setCreateTime(LocalDateTime.now());
        taskDO.setStatus(TaskStatusEnum.CREATED.getCode());
        taskMapper.insert(taskDO);
        //todo mq发消息调python
        userPptDetailDO.setLastTaskId(taskDO.getId());
        userPptDetailMapper.updateById(userPptDetailDO);
    }
}
```

在经过测试后，发现用线程池比直接串行提高了50%的响应速度

![](C:\Users\DELL\AppData\Roaming\marktext\images\2025-01-15-22-14-16-5c6007f39b59c4cbfc9145882bbfe6e.png)

![](C:\Users\DELL\AppData\Roaming\marktext\images\2025-01-15-22-14-07-eb8ebdc2c9b56dacd1150e7073ea2e6.png)

> 线程池参数如下：

```java

```
