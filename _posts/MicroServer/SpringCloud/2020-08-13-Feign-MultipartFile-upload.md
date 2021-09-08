---
layout: post
title: Feign多文件上传处理
categories: [Feign]
description: Feign multipart file upload
keywords: Feign
---

# Feign多文件上传解决方案

关于Feign接口单文件上传，比较简单，和普通的 SpringMVC 几乎是一样的，对应好参数和配置信息即可。

多文件上传却是个头疼的问题。

浏览了几篇博客，虽然有代码可以参考，但是并不详细。那么请看下面史上最细致的Feign多文件上传总结（自吹自擂一下），最细致谈不上，可以让你真正解决问题。

如果你遇到了下面的这个错误提示：

```
Caused by: org.apache.commons.fileupload.FileUploadException: the request was rejected because no multipart boundary was found
```
或者
```
The current request is not a multipart request 
```
恭喜你来对地方了！

有人说第二个错误“就是传输的文件数据为空了，比较简单”，真的简单么？当你什么都写了，参数也有文件了，但是调用Feign的时候就是报这个错误，这才是最头疼的问题，明明看着可以，其实就是不行。


## 开始解说

### 环境版本
#### Spring-boot版本

2.2.0.RELEASE
#### SpringCloud版本

Hoxton.M3
#### JDK版本

1.8
### 接口提供方
接口提供方不需要特殊处理，看一下示例代码，按照示例代码写就可以：
```java
   @RequestMapping(value = "/upload/batch" , method = RequestMethod.POST , consumes = MediaType.MULTIPART_FORM_DATA_VALUE )
    public ResponseData upload( @NotNull( message = "文件业务类型不能为空") @RequestParam("type") Integer type ,
                                @NotNull( message = "用户id不能为空") @RequestParam("userId") Long userId ,
                                @RequestParam("images") MultipartFile[] images ) {
        List<UserImageResVO> imageResVos = fileOpService.uploadImages( userId, type, images);
        return ResponseData.success( imageResVos ) ;
    }
```
注意的地方就是 @RequestMapping上 加上：

```
consumes = MediaType.MULTIPART_FORM_DATA_VALUE
```

这里的 文件参数 MultipartFile[] images 注解使用的是：RequestParam
```java
@RequestParam("images") 
```

之所以在这里提一下，因为在接口消费方使用的不是这个@RequestParam）。

### 接口消费方
#### 引入依赖

```xml
    <dependency>
        <groupId>io.github.openfeign.form</groupId>
        <artifactId>feign-form</artifactId>
        <version>3.8.0</version>
    </dependency>
    <dependency>
        <groupId>io.github.openfeign.form</groupId>
        <artifactId>feign-form-spring</artifactId>
        <version>3.8.0</version>
    </dependency>
    <dependency>
        <groupId>commons-fileupload</groupId>
        <artifactId>commons-fileupload</artifactId>
        <version>1.3.3</version>
    </dependency>
```

这是Feign为支持文件上传提供的依赖，你可以在 这里 选择不同的版本。

#### SpringForm编码器
```java

import feign.RequestTemplate;
import feign.codec.EncodeException;
import feign.codec.Encoder;
import feign.form.MultipartFormContentProcessor;
import feign.form.spring.SpringFormEncoder;
import feign.form.spring.SpringManyMultipartFilesWriter;
import feign.form.spring.SpringSingleMultipartFileWriter;
import org.springframework.web.multipart.MultipartFile;

import java.lang.reflect.Type;
import java.util.Collections;
import java.util.Map;

import static feign.form.ContentType.MULTIPART;

/**
 * @description  处理多个文件上传 编码器
 *
 * @author Songxudong
 * @date 2019/11/14 4:25 下午
 */
public class SpringMultipartEncoder extends SpringFormEncoder {
    public SpringMultipartEncoder(Encoder delegate) {
        super(delegate);
        MultipartFormContentProcessor processor = (MultipartFormContentProcessor) getContentProcessor(MULTIPART);
        processor.addWriter(new SpringSingleMultipartFileWriter());
        processor.addWriter(new SpringManyMultipartFilesWriter());
    }

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
        if (bodyType != null && bodyType.equals(MultipartFile[].class)) {
            MultipartFile[] file = (MultipartFile[]) object;
            if(file != null) {
                Map data = Collections.singletonMap(file.length == 0 ? "" : file[0].getName(), object);
                super.encode(data, MAP_STRING_WILDCARD, template);
                return;
            }
        }
        super.encode(object, bodyType, template);
    }
}
```
之所以需要提供这样一个编码器，是因为在程序内部调用Feign接口已经不是表单环境了，需要重新对文件内容进行编码操作。
开篇的第一个错误和这个就有关系，说“boundary（分割线）”找不到，就是因为不是表单提交，表单提交的话，都会存在分割线（对post请求了解的都应该知道）。

#### Feign多文件支持配置

```java
import com.system.consumer.sys.spring.SpringMultipartEncoder;
import feign.codec.Encoder;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @description feign-SpringMVC 多文件上传配置
 *
 * @author Songxudong
 * @date 2019/11/14 1:52 下午
 */
@Configuration
public class FeignMultipartSupportConfig {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Encoder feignEncoder() {
        return new SpringMultipartEncoder(new SpringEncoder(messageConverters));
    }

}
```

这个配置下面对在注入的Feign接口提供中使用。

#### 在消费方对服务接口进行配置

```java
@FeignClient( name = "medical-resource-provider" , configuration = FeignMultipartSupportConfig.class )
public interface IMedicalResourceService {


    /** 文件批量上传 */
    @PostMapping(value = "/file/upload/batch" , consumes = MediaType.MULTIPART_FORM_DATA_VALUE  )
    ResponseData upload( @RequestParam("type") Integer type , @RequestParam("userId") Long userId ,
                         @RequestPart("images") MultipartFile[] images ) ;
}
```
【注意：】
1）@RequestMapping中使用 consumes = MediaType.MULTIPART_FORM_DATA_VALUE
2）文件参数注解使用 @RequestPart，而不是 @RequestParam
3）将 FeignMultipartSupportConfig 配置加上

对于注意 2），如果还是使用 @RequestParam 的话，就会出现开篇的第一个错误；
consumes = MediaType.MULTIPART_FORM_DATA_VALUE 这个设置，如果在调用接口的定义上不使用就会出现开篇的第二个错误，原因还是因为，这里的调用不是表单环境了，需要我们手动告诉接口，我们的请求是Multipart类型的。在SpringMVC接口中，我们不需要配置也可以，因为会自动携带对应的信息。
