---
title: ChatGPT不仅能写代码还能改bug，这届AI全能
date: 2023-03-03 17:56:48
permalink: /pages/3fe7f6/
sidebar: auto
categories:
  - 随笔
tags:
  - 
author: 
  name: hexin
  link: https://github.com/ahxinin
---
工欲善其事必先利其器，我们先来看一下什么是ChatGPT。小试牛刀，让ChatGPT自己来回答一下：
![图片](/img_convert/8cf87c96e5f7c5f25176240196c762b4.png)

从介绍中可以看出来ChatGPT很擅长处理自然语言，那我们来看看AI处理编程语言的效果如何呢？  


第一个挑战，让ChatGPT实现一个分布式锁。在这里我指定了中间件使用Redis、编程语言使用Java。最终的实现代码如下：  

```
import org.springframework.data.redis.core.RedisTemplate;import org.springframework.data.redis.core.script.DefaultRedisScript;import org.springframework.data.redis.core.script.RedisScript;
import java.util.Collections;
public class DistributedLock {
    
    private RedisTemplate<String, String> redisTemplate;
    
    public DistributedLock(RedisTemplate<String, String> redisTemplate) {  
        this.redisTemplate = redisTemplate;    
    }
    
    public boolean acquire(String lockKey, int expireTime) {        
        RedisScript<String> script = new DefaultRedisScript<>(
           "return redis.call('set', KEYS[1], 'locked', 'NX', 'EX', ARGV[1])",
            String.class);
        String result =redisTemplate.execute(script,Collections.singletonList(lockKey),Integer.toString(expireTime));
        return "OK".equals(result);    
    }
    
    public void release(String lockKey) {        
        redisTemplate.delete(lockKey);    
    }
}
```

而更厉害的是，ChatGPT 给出了实现思路和使用注意事项：  


![图片](/img_convert/2ab14b651b614b94fcf968d77b00f3cb.png)

接下来上点难度，第二个挑战是让ChatGPT对接腾讯云接口，在告诉ChatGPT腾讯云发送短信API的文档链接、使用spring注解方式等信息后，马上就有结果了：

![图片](/img_convert/0cf4c80ea455dcbe03cadd8234a4b586.png)

```
<dependencies>    
    <dependency>        
        <groupId>com.tencentcloudapi</groupId>        
        <artifactId>tencentcloud-sdk-java</artifactId>        
        <version>3.1.63</version>    
    </dependency>
</dependencies>
```

然后，在你的 Spring Boot 应用程序中，你需要定义 `SmsClient` 的 Bean，并使用 `@Configuration` 和 `@Bean` 注解将其设置为单例模式。比如，可以在一个名为 `TencentCloudConfig` 的类中添加如下代码：

```
import com.tencentcloudapi.common.Credential;
import com.tencentcloudapi.common.profile.ClientProfile;
import com.tencentcloudapi.common.profile.HttpProfile;
import com.tencentcloudapi.sms.v20190711.SmsClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configurationpublic class TencentCloudConfig {

    @Value("${tencentcloud.secretId}")    
    private String secretId;
    
    @Value("${tencentcloud.secretKey}")    
    private String secretKey;
    
    @Value("${tencentcloud.sms.appid}")    
    private String smsSdkAppid;
    
    @Value("${tencentcloud.sms.sign}")    
    private String sign;
    
    @Bean    
    public SmsClient smsClient() {        
        Credential cred = new Credential(secretId, secretKey);        
        HttpProfile httpProfile = new HttpProfile();     
        httpProfile.setEndpoint("sms.tencentcloudapi.com");
        ClientProfile clientProfile = new ClientProfile();
        clientProfile.setHttpProfile(httpProfile);
        return new SmsClient(cred, "", clientProfile);    
    }
}
```

最后，我们在 `SmsService` 类中注入 `SmsClient` 的 Bean，进行短信发送的操作。比如，可以在 `SmsService` 类中添加如下代码：

```
import com.tencentcloudapi.sms.v20190711.SmsClient;
import com.tencentcloudapi.sms.v20190711.models.SendSmsRequest;
import com.tencentcloudapi.sms.v20190711.models.SendSmsResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Servicepublic 
class SmsService {
    @Autowired    
    private SmsClient smsClient;
    
    public void sendSms(String phoneNumber, String templateId, String[] templateParams) {        
        // 创建 SendSmsRequest 对象        
        SendSmsRequest req = new SendSmsRequest();        
        req.setSmsSdkAppid("你的短信应用 SDK AppID");        
        req.setSign("短信签名内容");        
        req.setTemplateID(templateId);        
        req.setPhoneNumberSet(new String[] {phoneNumber});     
        req.setTemplateParamSet(templateParams);
        // 发送短信        
        try {            
            SendSmsResponse resp = smsClient.SendSms(req);
            System.out.println(resp.toString());        
        } catch (Exception e) {            
            System.out.println(e.toString());        
        }    
    }
}
```

看来写代码难不倒ChatGPT。接下来看看它改bug的能力吧。  


  
发送一段有 bug 的代码给 ChatGPT：

```
class Account {  
    private int balance;  
    
    // 转账  
    void transfer(Account target, int amt){    
        // 锁定转出账户    
        synchronized(this){           
            // 锁定转入账户      
            synchronized(target){         
                if (this.balance > amt) {          
                    this.balance -= amt;          
                    target.balance += amt;        
                }      
            }    
        }  
    } 
}
```

![图片](/img_convert/bbe9109d800c5ac7c2dede657ba5c048.png)  

```
class Account {  
    private int balance;  
    private static final Object lock = new Object();
  
    // 转账  
    void transfer(Account target, int amt){    
        // 获取锁顺序    
        Account first = this;    
        Account second = target;    
        if (this.hashCode() > target.hashCode()) {      
            first = target;      
            second = this;    
        }
    
        // 锁定转出账户    
        synchronized(first){      
            // 锁定转入账户      
            synchronized(second){        
                if (this.balance > amt) {          
                    this.balance -= amt;          
                    target.balance += amt;        
                }      
            }    
        }  
    } 
}
```

在这个修复后的版本中，使用了一个静态的锁对象来避免不必要的竞争。同时，还根据账户的哈希值来决定获取锁的顺序，从而避免了死锁问题。

看到这里，你觉得ChatGPT表现得怎么样呢。


