

### 断路器实验

###### 操作一、创建用户

```
curl --location --request POST --X POST 'http://gateway.ex.test/accounts/' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Content-Type: application/json' \
--data '{"username": "nginx1234","password": "123456"}'
```
> 提示：请将服务地址替换为您当前的服务地址
###### 操作二、获取token

```
curl --location --request POST --X POST 'http://gateway.ex.test/uaa/oauth/token' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Authorization: Basic YnJvd3Nlcjo=' \
--form 'scope=ui' \
--form 'username=nginx123' \
--form 'password=123456' \
--form 'grant_type=password'
```
> 提示：请将服务地址替换为您当前的服务地址

###### 操作三、保存账户数据

```
curl --location --request PUT --X PUT 'http://gateway.ex.test/accounts/current' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Authorization: Bearer  980e1ce3-b0d9-441c-ab50-fcd186966b58' \
--header 'Content-Type: application/json' \
--data '{
	"incomes": null,
	"expenses": null,
	"saving": {
		"amount": 2,
		"currency": "USD",
		"interest": 0,
		"deposit": false,
		"capitalization": false
	},
	"note": null
}'
```
> 提示：请将服务地址替换为您当前的服务地址

登录您的监控服务：http://monitoring.ex.test/hystrix

> 提示：请将服务地址替换为您当前的服务地址

并按照图示输入以下信息 `http://turbine-stream-service:8080/turbine/turbine.stream`，并点击"Monitor Stream"

![image](https://user-images.githubusercontent.com/4653664/175772271-77cdd263-bf1c-4892-885b-f93d8dcbe118.png)

依次执行操作1-2，然后重复执行操作3，登录断路器监控界面，查看断路器的状态是否已经`Open`。

![image](https://user-images.githubusercontent.com/4653664/175772202-c717450a-c47f-4575-97c9-19599cc69e2d.png)







