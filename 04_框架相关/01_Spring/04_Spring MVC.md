# Spring MVC

### 一. Spring MVC框架核心？

1. 用户访问接口时，请求发送至tomcat后，tomcat会将请求转交给Spring MVC框架处理
2. DispatcherServlet接收到请求后，先去查找控制层，即使用了@Controller注解的类。
3. 再通过@RequestMapping注解，依据请求中uri，查找到类中对应处理该请求的方法
4. 方法执行完成后，能够返回值，可以为对应页面模板名字，返回显示对应页面；前后端分离后，更多是返回JSON串，前端进行逻辑处理后渲染至页面中。

