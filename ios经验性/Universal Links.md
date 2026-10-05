# Universal Links

[iOS Universal Links教程 - 简书](https://www.jianshu.com/p/f1a1e1833eec)

 iOS9推出的特性，当用户点击通用链接时，iOS设备可以不通过Safari或网页，直接打开App，比如在备忘录中直接打开App。

通用链接是标准的HTTPS链接，既可以打开App，也可以打开网页（在未安装App的时候)。 

可以使用通用链接，在不同App页面跳转，以及传递参数。 

# 原理：
1. APP侧配置支持applinks的Associated Domains。
2. 在域名的.well-known文件下或者根路径下上传apple-app-site-association文件，里面通过json描述指示了APP能处理的该域名下的路径（可以是一个或多个）
3. APP安装或更新时，系统会从配置的域名下Get获取.well-known/apple-app-site-association文件并解析访问Universal Links的配置。
4. 当从Safari中访问该域名下的指定路径时，iOS系统会查询当前url是否在APP有注册，有的话就可以直接打开对应APP的内容。