
### 服务发现

访问Eureka服务的地址： http://registry.ex.test/ ,并查看相关微服务是否已经注册成功。

> 提示：请将服务地址替换为您当前的服务地址

![image](https://user-images.githubusercontent.com/4653664/175772378-6629d7e4-ced8-4abd-90f6-c706a6dc6675.png)

### 访问应用

登录Piggymetrics应用地址： http://gateway.ex.test

> 提示：请将服务地址替换为您当前的服务地址

![image](https://user-images.githubusercontent.com/4653664/175774202-cf16bd99-9b48-4049-a7fc-171d724cdc59.png)

注册账户，如demotest/123456，跳过通知订阅。

![image](https://user-images.githubusercontent.com/4653664/175774274-112bf056-43ff-484d-b090-c1aa8d880be1.png)

点击"Get IN",然后依次添加`INCOME`\`EXPENSE`\`SAVINGS`信息

![image](https://user-images.githubusercontent.com/4653664/175774388-5a3f9618-89e4-4de1-89b9-fc272beefcc9.png)

然后点击“SAVECHANGES”

![image](https://user-images.githubusercontent.com/4653664/175774408-be065ffe-51d9-447e-bdcd-f0cf86627d0e.png)


### 断路器

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







