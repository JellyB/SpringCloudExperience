####  Feign Client 远程服务调用
服务间调用使用FeignClient调用：
题库服务（tiku）通过FeignClient调用用户服务（user）服务接口:
##### 通过`application.name`发现服务
``` yml
    spring:
      profiles: dev
      application:
      name: ${server_name}
```

#### 服务提供方FeignClient 书写
FeignClient指定调用注册到eureka上的`application.name` id
fallback指定熔断处理类

```java
@FeignClient(value = "ic-user-server-bgd", fallback = UserServiceClientAbstract.class)
public interface UserServiceClient {
    /**
     * 获取用户详情
     * @param ids
     * @return
     */
    @RequestMapping(value= "/cloud/v1/user/commentInfo", method = RequestMethod.GET)
    SuccessResponse<Map<Long, CommentUserInfoResp>> getCommentUserInfoList(@RequestParam("ids") Set<Long> ids);
}
```

#### 服务调用方
如果报错，对异常进行处理

```java
@Service
@Slf4j
public class UserServiceHandler {
    @Resource
    private  UserServiceClient userServiceClient;

    /**
     * 数据解析处理
     * 
     */
    public Map<Long, CommentUserInfoResp> getCommentUserInfoList(Set<Long> ids) throws BizException{
        try{
            SuccessResponse<Map<Long, CommentUserInfoResp>> response = userServiceClient.getCommentUserInfoList(ids);
            return response.getData();
        }catch (FeignException e){
            log.error("user cloud 接口调用异常！", e);
            throw new BizException(TikuSystemStatusEnum.create(TikuSystemStatusEnum.SYSTEM_ERROR.FEIGN_EXECUTE_EXCEPTION, "用户基本评论信息"));
        }
    }

}
```

#### 熔断处理
```java
//扫描为bean
@Component
public class UserServiceClientFallBackImpl extends UserServiceClientAbstract {
    /**
     * 熔断数据
     * 
     */
    @Override
    public SuccessResponse<Map<Long, CommentUserInfoResp>> getCommentUserInfoList(Set<Long> ids) {
        SuccessResponse successResponse = new SuccessResponse();
        Map<Long, CommentUserInfoResp> maps = Maps.newHashMap();
        //造了1000条数据
        for(int i = 0; i <= 1000; i ++){
            maps.put(new Long(i), CommentUserInfoResp.builder().avatar("http://HelloHuaTu"+ i +".jpg").id(1l).nickname("HelloWorld").build());
        }

        successResponse.setData(maps);
        return successResponse;
    }
}
```
#### applicaton.yml 配置文件
##### 开启熔断
```
    feign:
      hystrix:
        enabled: true
```
##### 配置超时时间
```
    ribbon:
      ReadTimeout: 60000
        ConnectTimeout: 60000
```  
#### 场景分析：
#### 正常情况(在关了翻墙vpn情境下进行~~都是泪)
    `IC-TIKU-SERVER	n/a (1)	(1)	UP (1) - localhost:ic-tiku-server:8080`
    `IC-USER-SERVER-BGD	n/a (1)	(1)	UP (1) - localhost:ic-user-server-bgd:11084`

- `如果不配置熔断`即服务提供方或调用方不处理`fallback`
    此时对用户服务接口端点挂起，题库服务在多次请求用户服务`（默认两次）`后，在`eureka`上回发现服务出错：
    返回值：

    ```
        {
            "data": null,
            "message": "服务内部错误",
            "code": 1000102
        }
    ```
    `eureka`控制面板`http://localhost:11003/`会出现如下警告：
    `EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING       EXPIRED JUST TO BE SAFE.`
    提示服务已经开始不健康了，此种场景下造成的服务不健康目前来看需要重启`eureka`，否则服务接口仍然处于不可用状态。

- `配置熔断没有设置超时时间`，或超时时间太短，会直接走熔断数据;
    对user服务断点住，如设置超时时间为`60ms`
- `正常开启服务熔断`，并且设置服务熔断时间合理比如上述的`60s`，并且设置`fallback`实现;
    此时如果user服务正常，则正常通过`feign`调用用户数据；
    否则会直接走熔断`fallback`返回默认添加的1000条数据内容，而且用户服务只会挂起`2-3`个线程请求
    

