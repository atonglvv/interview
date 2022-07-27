# @RequestMapping

## params

@RequestMapping注解的params属性通过请求的请求参数匹配请求映射。

@RequestMapping注解的params属性是一个字符串类型的数组，可以通过四种表达式设置请求参数和请求映射的匹配关系。

- "param”：要求请求映射所匹配的请求必须携带param请求参数，例如username
- "!param”：要求请求映射所匹配的请求必须不能携带param请求参数，例如!username
- "param=value"：要求请求映射所匹配的请求必须携带param请求参数且param=value,例如username=admin
- "param!=value"：要求请求映射所匹配的请求必须携带param请求参数但是param!=value,例如password!=123456

**Demo：**

必须携带username请求参数

```java
		@RequestMapping(value = "/testParamsAndHeaders",
                params = {"username")
      	public  String testParamsAndHeaders(){
        　　	return "success";
      	}
```

## headers

@RequestMapping注解的headers属性通过请求的请求头信息匹配请求映射。

@RequestMapping注解的headers属性是一个字符串类型的数组，可以通过四种表达式设置请求头信息和请求映射的匹配关系。

- "header"：要求请求映射所匹配的请求必须携带header请求头信息

- "!header"：要求请求映射所匹配的请求必须不能携带header请求头信息

- "header=value"：要求请求映射所匹配的请求必须携带header请求头信息且header=value

- "header!=value"：要求请求映射所匹配的请求必须携带header请求头信息且header!=value

若当前请求满足@RequestMapping注解的value和method属性，但是不满足headers属性，此时页面显示404错误，即资源未找到。

## ant

SpringMVC支持ant风格的路径

- ？：表示任意的单个字符

- *：表示任意的0个或多个字符

- **：表示任意的一层或多层目录

注意：在使用 * * 时，只能使用 * * 必须在两个 / 中间



# @RequestHeader 

@RequestHeader 是获取请求头中的数据，通过指定参数 value 的值来获取请求头中指定的参数值。其他参数用法和 @RequestParam 完全一样。

```java
@Controller
public class handleHeader {
 
	@GetMapping("/getHeader")
	public String getRequestHeader(@RequestHeader("User-Agent") String agent) {
		System.out.println(agent);
		return "success";
	}
}
```



# @cookievalue

用法同  @RequestParam @RequestHeader。



# RequestEntity

RequestEntity 类型用于获取整个请求报文，包括请求头、请求体等信息。

```java
@Controller
public class MyController {

    @RequestMapping(value = {"/hello"})
    public String hello(RequestEntity<String> requestEntity){
        HttpHeaders headers = requestEntity.getHeaders();
        String body = requestEntity.getBody();
        URI url = requestEntity.getUrl();
        HttpMethod method = requestEntity.getMethod();
        System.out.println("请求头：" + headers.toString());
        System.out.println("请求体：" + body);
        System.out.println("URL：" + url.toString());
        System.out.println("请求方法：" + method.toString());
        return "hello";
    }
}
```



# ResponseEntity

ResponseEntity 可以作为controller的返回值。

常用于处理文件下载接口。

```java
@RequestMapping("/download")
public ResponseEntity<byte[]> download(@RequestParam String fileName) throws IOException {
    byte[] bytes = xxx;
    return ResponseEntity.ok()
        .headers(headers)
        .body(bytes);
}
```



# 拦截器Interceptor的执行顺序？

