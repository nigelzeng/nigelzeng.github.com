---
layout:     post
title:      nginx的error_page知识
category:   nginx
description: 关于如何配置nginx保持代理服务器的404响应码。
---

好久没写博客，感觉都有些生疏了。这几个月接手了几个新的应用，忙着去了解这些应用的上下游依赖，导致没精力更新博客。
趁着有喘息的机会，更新一篇，博客不能断。

###背景
服务器上的结构：nginx+tomcat，nginx做代理。

某天开发找到我说想要把后端的404 http response code返回到用户浏览器的时候保持404。因为默认的统一配置是把一些4xx、5xx的响应码跳转到指定的错误页面，这样导致客户端浏览器看到的是302跳转。

对于某些需要让浏览器收录的应用来说，如果后端应用服务器返回的404页面被webserver响应成302跳转，那么搜索引擎默认会对302继续进行访问，收录302重定向之后的页面，而我们却想让浏览器直接不收录这个302页面，因为它是不存在的。（有点儿绕，最终目的就是保证后端的404到浏览器端也是404）

###error_page指令知识
error_page指令可以配置在http、server、location、location的if语句中，它的语法很简单：
`error_page code... [=[response]] uri`

[error_page][]的配置方法如下：
     error_page   404          /404.html;
     error_page   502 503 504  /50x.html;
     error_page   403          http://example.com/forbidden.html;
     error_page   404          = @fetch;

*   第一行和第二行把访问静态文件产生的404响应重定向到http://server_name/404.html页面。
*   第三行则直接重定向到指定的http://example.com/forbidden.html（可以是外站）
*   第四行则是指定到一个location，这个location一定要在配置文件中配置。类似于：
     location @fetch {
          proxy_pass http://server;
     }

如果需要把代理服务器的错误响应也重定向到指定的页面，请务必加上：
`proxy_intercept_errors  on;`


###如何配置
这段配置其实很简单，我自己走了很多弯路，没有想到点上：

*   首先确保打开`proxy_intercept_errors  on;`
*   配置error_page：`error_page 404 /error1.html;`
*   配置location：
     location = /error1.html {
          root /usr/local/nginx/htdocs;
     }

这段配置`error_page 404 /error1.html;`指定把tomcat的404响应在nginx内部重写为/error1.html这个uri，然后这个uri会匹配到error1.html这个location，root指令则指定了这个请求会访问本地/usr/local/nginx/htdocs下的对应文件。

以上配置的结果，后端tomcat的404响应经过nginx后，依然是404响应。

这里列一个产生意外结果的配置：
     proxy_intercept_errors  on;
     error_page 404 =200 /error1.html;
     location = /error1.html {
          root /home/admin/cai/htdocs;
     }

这个配置的结果则是把tomcat返回的404，由nginx修改为200状态码，页面内容为error1.html。

**---EOF---**

[error_page]:   http://wiki.nginx.org/HttpCoreModule#error_page "error_page"
