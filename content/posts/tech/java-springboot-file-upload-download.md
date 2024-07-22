---
title: "SpringBoot 关于文件的上传与下载"
date: 2024-04-06T12:28:30+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "java"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文讨论了在 Spring Boot 中进行跨服务文件传输的问题，同时介绍了在 Spring Boot 和 OpenFeign 中实现文件上传的方法和问题分析，包括普通文件上传和上传时可能遇到的问题及解决方案等。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---
## 碎碎念

在 Springboot 中，跨服务的文件传输需要考虑性能、一致性、安全性等问题。

对于 Excel 文件的上传下载，是否在对外服务层进行解析和封装会更好呢？

对于其他类型的文件，可以考虑独立出一个文件服务，以实现服务解耦和更好的水平扩展。

## 01 SpringBoot 图片上传

### 普通文件上传

普通的文件上传，直接使用 `MultipartFile` 类型声明参数即可：

```JAVA
@RestController
public class FileUploadController {

    @PostMapping("/upload")
    public ResponseEntity<String> handleFileUpload(
            @RequestParam("file") MultipartFile file,
            @RequestParam("param1") String param1,
            @RequestParam("param2") String param2) {

        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("请选择文件上传");
        }

        // 这里可以添加文件上传的业务逻辑，比如保存文件到服务器

        return ResponseEntity.ok("文件上传成功");
    }
}
```

如果是多文件上传，则使用 `MultipartFile[]` 数组。

如果参数过多，可以使用 `@SpringQueryMap` 把参数放在类里面，这是 OpenFeign 提供的类似 `@QueryMap` 功能，支持把 Query 参数封装到对象中：

```JAVA
@RestController
public class FileUploadController {

    @PostMapping("/upload")
    public ResponseEntity<String> handleFileUpload(
            @RequestParam("file") MultipartFile file,
            @Validated @SpringQueryMap QueryDTO req) {
        // do sth.
    }
}
```

## 02 OpenFeign 文件上传

当从对外的 Web 服务把文件传到 OpenFeign 的服务时，MultipartFile 需要使用 `@RequestPart` 而非 `@RequestParam`，请求头 `content-type` 为 `multipart/form-data`，否则会报错：

*org.springframework.web.multipart.MultipartException:Current request is not a multipart request...*

```JAVA
@FeignClient(name = "OK-productor", fallback = RenderApiFallback.class)
public interface RenderApi {

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    ResponseObject upload(@RequestPart("file") MultipartFile file);
}
```

至于对外的 web 服务或者内部 OpenFiegn 调用的服务，使用 `@RequestPart` 或者 `@RequestParam`，只要前端支就都能正常调用。

### 问题分析

```mermaid
graph LR;
    client --> |http| Web -->|feign| A;
```

#### Q.1 Feign 仅用 `@RequestParam` 注解 MultipartFile

```JAVA
// Web
@PostMapping("/upload")
public R upload(@RequestParam("file") MultipartFile file) {
   return aApi.upload(file);
}

// Feign interface
@PostMapping("/server-a/upload")
R upload(@RequestParam("file") MultipartFile file);

// A
@PostMapping("/upload")
R upload(@RequestParam("file") MultipartFile file) {}
// or
R upload(@RequestPart("file") MultipartFile file) {}
```

A 服务报错：Required request part 'file' is not present。

#### Q.2 Feign 仅用 `@RequestPart` 注解 MultipartFile

针对 Q.1 的问题，将 RequestParam 改为 RequestPart：

```JAVA
// Web
@PostMapping("/upload")
public R upload(@RequestParam("file") MultipartFile file) {
   return aApi.upload(file);
}

// Feign interface
// RequestParam 改为 RequestPart
@PostMapping("/server-a/upload")
R upload(@RequestPart("file") MultipartFile file);

// A
@PostMapping("/upload")
R upload(@RequestParam("file") MultipartFile file) {}
// or
R upload(@RequestPart("file") MultipartFile file) {}
```

Feign 调用直接熔断，无法连接 A 服务。

#### Q.3 Feign 使用 `@RequestPart` 注解 MultipartFile，并且添加 consume 类型

仅用 `@RequestPart` 注解 MultipartFile 会导致熔断，应该是接口对不上，而通过 postman 上传的请求头是 form-data 类型的，所以改为一致再试试：

```JAVA
// Web
@PostMapping("/upload")
public R upload(@RequestParam("file") MultipartFile file) {
   return aApi.upload(file);
}

// Feign interface
// RequestParam 改为 RequestPart
// 添加 form-data 类型
@PostMapping(value = "/server-a/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
R upload(@RequestPart("file") MultipartFile file);

// A
@PostMapping("/upload")
R upload(@RequestParam("file") MultipartFile file) {}
// or
R upload(@RequestPart("file") MultipartFile file) {}
```

- consumes:  指定处理请求的提交内容类型（Content-Type）

当 Feign 使用 `@RequestPart` 注解 MultipartFile，并且指定 consume 类型为 `multipart/form-data`，**可以正常传输文件！**

#### Q.4 A 服务时好时坏

在 Feign 接口中使用了 `@RequestPart` 注解，配和 `consumes = "multipart/form-data"`，已经可以正常通过 Feign 调用传输文件了。

但是，在测试过程中发现 A 服务时好时坏，报错：`Required request part 'file' is not present`，这是为什么呢？

后来发现，在调用 Feign 接口时，拦截器默认给所有请求添加了 `Content-Type: application/json`，而 Feign 调用时本身就加上了 `content-type: multipart/form-data`：

```JAVA
@Configuration
public class FeginInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        Map<String,String> headers = getHeaders(getHttpServletRequest());
        for(String headerName : headers.keySet()){
            String headerValue;
            if("content-type".equals(headerName)){
                headerValue = "application/json;charset=UTF-8";
                // 修改：跳过 content-type，不再添加该请求头，使用默认头即可
                // continue;
            }else {
                headerValue = getHeaders(getHttpServletRequest()).get(headerName);
            }
            template.header(headerName, headerValue);
        }
    }
}
```

其中，`template.header(headerName, headerValue)` 的操作是追加（append），而不是添加（代码位置 `feign.RequestTemplate#appendHeader`）：

```JAVA
private RequestTemplate appendHeader(String name, Iterable<String> values) {
    if (!values.iterator().hasNext()) {
        this.headers.remove(name);
        return this;
    } else {
        this.headers.compute(name, (headerName, headerTemplate) -> {
            // 这里执行的是 HeaderTemplate.append
            // 原值：Content-Type multipart/form-data; charset=UTF-8; boundary=18e89936153
            // 追加：Content-Type multipart/form-data; charset=UTF-8; boundary=18e89936153, application/json;charset=UTF-8
            return headerTemplate == null ? HeaderTemplate.create(headerName, values) : HeaderTemplate.append(headerTemplate, values);
        });
        return this;
    }
}
```

经过拦截器的追加之后，准备请求 Feign 时，可以看到 `Content-Type` 是两个值，也就是上面的 `multipart/form-data` 和 `application/json;charset=UTF-8`（代码位置 `feign.SynchronousMethodHandler#executeAndDecode`）

- 在调用前，获取到的 RibbonRequest 还是那两个请求头，类型为 `UnmodifiableRandomAccessList`（代码位置 `org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute`）

```JAVA
Object executeAndDecode(RequestTemplate template) throws Throwable {
        Request request = this.targetRequest(template);

        long start = System.nanoTime();

        Response response;
        try {
            response = this.client.execute(request, this.options);
        }
        
        // ...
}
```

从多次观察 `feign.SynchronousMethodHandler#executeAndDecode` 中 client 的请求结果，可以看出，只要 `Content-Type` 这个随机数组的第一个请求头是 `multipart/form-data`，请求就成功。

也就是说，项目的 Feign 自动添加了请求头（只考虑 json 格式），导致其他的请求会带上多个一样的请求头，真正的请求用到的是请求头数组的第一个，而请求头数组是随机的，导致请求时而成功，时而失败。

所以，如果项目中添加了 Feign 的拦截配置，当请求头是 `Content-Type` 是，直接跳过即可。

**为什么要给 Feign 添加 json 类型的 content-type？**

*我猜原本的 Feign 配置是为了给一些没有带 `Content-Type` 请求头或者带其他格式请求头的请求都统一转为 json，但没有考虑到文件传输的问题。原本的文件传输接口，直接在 Web 服务实现的，它是通用上传接口，在 Web 服务实现并没有什么问题，也省去了中间服务调用的损耗。还有一种就是文件传输到 Web，直接解析成参数再调用相应的服务，这也不需要在服务间传递文件。但是，部分特殊的需求，如传输二进制文件、压缩包等，并且需要使用到相应服务的数据库资源等，在相应的服务操作是比较合适的。*

## References

[1] HE-RUNNING. Feign 传输 Multipartfile 文件的正确方式，Current request is not a multipart request 报错解决. https://blog.csdn.net/weishaoqi2/article/details/106479476, 2020-06-01.  
[2] 哈哈哈哈哈基米. 关于 openfeign 调用时 content-type 的问题. https://blog.csdn.net/ssH18868325485/article/details/132339361, 2023-08-17.  
