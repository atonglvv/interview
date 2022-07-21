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