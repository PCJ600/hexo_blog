---
layout: next
title: Service Gateway 项目设计
date: 2025-02-28 20:19:17
categories: Project
tags: Project
---


# Virtual Appliance Design
![](va_design.png)



# heartbeart
![](va_auto_upgrade.png)

<!-- more -->

# Overview
```
frontend ------->  backend ---------> MQ ----------->  VA(On-premise)
                       /|\                             |  
                        |------------------------------|
						
peter.backend.com:443  backend
peter.channel.com:5672 MQ
peter.frontend.com     frontend
```

# Test API

## API for frontend
```
curl -H "x-customer-id: 77a1c9561c8d47fb8036c970a4f2ee73" "https://peter.backend.com/ui/appliances/?start=0&applianceId=974bf535-7930-474e-8da6-780cafff284d"
curl -H "x-customer-id: 77a1c9561c8d47fb8036c970a4f2ee73" "https://peter.backend.com/ui/appliances/?start=1736680946"
curl https://peter.backend.com/ui/appliances/image/
curl -X DELETE https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/
curl -X POST https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/upgrade/
curl https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/settings/
curl -H "x-customer-id: 77a1c9561c8d47fb8036c970a4f2ee73" -X POST https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/settings/
curl https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/storage/
curl -X POST https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/storage/
curl -X POST https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/log/
curl https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/services/

curl -X POST https://peter.backend.com/ui/services/squid/install/
curl -X POST https://peter.backend.com/ui/services/squid/974bf535-7930-474e-8da6-780cafff284d/toggle/
curl https://peter.backend.com/ui/services/squid/974bf535-7930-474e-8da6-780cafff284d/settings/
curl -X POST https://peter.backend.com/ui/services/squid/974bf535-7930-474e-8da6-780cafff284d/settings/

curl https://peter.backend.com/ui/tasks/12345/
```

## API for VA
```
curl -X POST -H "Authorization: Bearer $token" -i https://peter.backend.com/va/register/
curl -X POST https://peter.backend.com/va/register/
curl -X POST https://peter.backend.com/va/974bf535-7930-474e-8da6-780cafff284d/tasks/
curl https://peter.backend.com/va/974bf535-7930-474e-8da6-780cafff284d/healthz/
curl -X POST https://peter.backend.com/va/974bf535-7930-474e-8da6-780cafff284d/log/
curl -X POST https://peter.backend.com/va/974bf535-7930-474e-8da6-780cafff284d/squid/metrics/
```

# backend Design

## Database Design

### Customer 
Customer表
```
desc app_customer;
+----------------+-------------+------+-----+---------+-------+
| Field          | Type        | Null | Key | Default | Extra |
+----------------+-------------+------+-----+---------+-------+
| id             | char(32)    | NO   | PRI | NULL    |       |
| register_token | longtext    | NO   |     | NULL    |       |
| created_at     | datetime(6) | NO   |     | NULL    |       |
| updated_at     | datetime(6) | NO   |     | NULL    |       |
+----------------+-------------+------+-----+---------+-------+
```

### Appliance
ApplianceInfo表
```
mysql> desc app_applianceinfo;
+--------------------+-------------+------+-----+----------------+-------------------+
| Field              | Type        | Null | Key | Default        | Extra             |
+--------------------+-------------+------+-----+----------------+-------------------+
| id                 | char(32)    | NO   | PRI | NULL           |                   |
| customer_id        | char(32)    | NO   | MUL | NULL           |                   |
| created_at         | datetime(6) | NO   |     | NULL           |                   |
| updated_at         | datetime(6) | NO   |     | NULL           |                   |
| version            | varchar(16) | NO   |     | NULL           |                   |
| status             | varchar(16) | NO   |     | NULL           |                   |
| appliance_settings | json        | NO   |     | NULL           |                   |
| capacity           | json        | NO   |     | _utf8mb3\'{}\' | DEFAULT_GENERATED |
| connect_time       | bigint      | NO   | MUL | NULL           |                   |
| expected_status    | varchar(16) | NO   |     | NULL           |                   |
+--------------------+-------------+------+-----+----------------+-------------------+
```

ApplianceVersion表
```
mysql> desc app_applianceversion;
+------------------+-------------+------+-----+----------------+-------------------+
| Field            | Type        | Null | Key | Default        | Extra             |
+------------------+-------------+------+-----+----------------+-------------------+
| version          | varchar(32) | NO   | PRI | NULL           |                   |
| created_at       | datetime(6) | NO   |     | NULL           |                   |
| updated_at       | datetime(6) | NO   |     | NULL           |                   |
| version_major    | int         | NO   | MUL | NULL           |                   |
| version_minor    | int         | NO   | MUL | NULL           |                   |
| version_revision | int         | NO   | MUL | NULL           |                   |
| version_build    | int         | NO   | MUL | NULL           |                   |
| display_version  | varchar(32) | NO   |     | NULL           |                   |
| firmware_info    | json        | NO   |     | NULL           |                   |
| enable           | tinyint(1)  | NO   |     | NULL           |                   |
| ova_info         | json        | NO   |     | _utf8mb3\'{}\' | DEFAULT_GENERATED |
| published        | tinyint(1)  | NO   |     | NULL           |                   |
+------------------+-------------+------+-----+----------------+-------------------+
```

```
mysql> desc app_appliancemetrics;
+-------------------------+----------+------+-----+---------+----------------+
| Field                   | Type     | Null | Key | Default | Extra          |
+-------------------------+----------+------+-----+---------+----------------+
| id                      | bigint   | NO   | PRI | NULL    | auto_increment |
| appliance_id            | char(32) | NO   | MUL | NULL    |                |
| record_time             | bigint   | NO   |     | NULL    |                |
| cpu_used                | double   | NO   |     | NULL    |                |
| memory_total            | bigint   | NO   |     | NULL    |                |
| memory_used             | bigint   | NO   |     | NULL    |                |
| storage_total           | bigint   | NO   |     | NULL    |                |
| storage_used            | bigint   | NO   |     | NULL    |                |
| storage_unallocated     | bigint   | NO   |     | NULL    |                |
| storage_capacity_detail | json     | NO   |     | NULL    |                |
| storage_usage_detail    | json     | NO   |     | NULL    |                |
+-------------------------+----------+------+-----+---------+----------------+
```

### IOT Task
IotTask表
```
mysql> desc app_iottask;
+---------------+-------------+------+-----+---------+-------+
| Field         | Type        | Null | Key | Default | Extra |
+---------------+-------------+------+-----+---------+-------+
| id            | char(32)    | NO   | PRI | NULL    |       |
| created_at    | datetime(6) | NO   | MUL | NULL    |       |
| updated_at    | datetime(6) | NO   |     | NULL    |       |
| task_type     | varchar(50) | NO   | MUL | NULL    |       |
| appliance_id  | char(32)    | NO   | MUL | NULL    |       |
| customer_id   | char(32)    | NO   |     | NULL    |       |
| message       | json        | NO   |     | NULL    |       |
| status        | varchar(16) | NO   |     | NULL    |       |
| retry_count   | int         | NO   |     | NULL    |       |
| result        | json        | NO   |     | NULL    |       |
| error_message | longtext    | NO   |     | NULL    |       |
+---------------+-------------+------+-----+---------+-------+
```

HeartbeatIotTask表
```
desc app_heartbeatiottask;
+---------------+-------------+------+-----+---------+----------------+
| Field         | Type        | Null | Key | Default | Extra          |
+---------------+-------------+------+-----+---------+----------------+
| id            | bigint      | NO   | PRI | NULL    | auto_increment |
| created_at    | datetime(6) | NO   | MUL | NULL    |                |
| updated_at    | datetime(6) | NO   |     | NULL    |                |
| appliance_id  | char(32)    | NO   | MUL | NULL    |                |
| customer_id   | char(32)    | NO   |     | NULL    |                |
| message       | json        | NO   |     | NULL    |                |
| status        | varchar(16) | NO   | MUL | NULL    |                |
| result        | json        | NO   |     | NULL    |                |
| error_message | longtext    | NO   |     | NULL    |                |
+---------------+-------------+------+-----+---------+----------------+
```

### Service


## IotTask Design
* install/uninstall/enable/disable/configure services
* upgrade firmware/services
* collect metrics about va and services
	* va's cpu,memory, storage, network usage
	* maintain service status(running, disabled, disconnected)	
* heartbeart
* collect debug logs
* collect metrics
* extend storage
* change api key and server ssl certificates
* remote shell

### HeartBeat
use applianceId as taskId to avoid database overwhelm
**Heartbeat Body** 
```
{
	"taskId": applianceId,
	"taskType": 
	"serverTime": 1736588457
}
```
**Heartbeat Result**
```

```

### Upgrade Appliance
```
{
	"taskId": "uuid",
	"taskType": "upgradeAppliance",
	"targetVersion": "1.0.0.100",
	"firmwarePath": "https://XXX/",
	"firmwareSha256": "XXX"
}
```

### Install Services
```
{
	"taskId": "uuid",
	"taskType": "installService",
	"serviceCode": "va-squid",
	"targetVersion": "1.0.0.10000",
	"branch": "main",
	"imagePath": "download_url",
	"imageSha256": "sha256sum"
}
```

### Uninstall Services
```
{
	"taskId": "uuid",
	"taskType": "installService",
	"serviceCode": "va-squid"
}
```

### Collect log
```
{
	"start": "2021-01-01",
	"end": "2021-02-02",
	"uploadPath": ""
}
```

### Collect Appliance Metrics
```
{

}
```

### Unregister VA
```
{
	"taskId": "uuid",
	"taskType": "unregister"
}
```

### Update Appliance Settings
```
{
	"taskId": "uuid",
	"taskType": "updateApplianceConfig",
	"body": {
		"settings1": "value1",
		"settingsn": "valuen"
	}
}
```

### Extend Appliance Storage
```
{
	"storage": {
		"extend": [
			{
				"targetVolume": "data | image",
				"size": 1024
			}
		]
	}
}
```


## backend API design

### API for VA

#### VA register
POST /va/register
* VA发起注册请求, Header中携带JWT
* 校验请求头的JWT
	* 解析JWT中的customer_id, 查数据库是否存在该customer
	* 检查JWT是否过期，如果过期就为客户重新生成一个新的JWT，继续校验
* 从请求体中读取VA信息，将VA信息写入数据库
* 响应VA，返回(iotHost, iotCert, applianceId)给VA。

**Response Body**
```
{
	"applianceId": "uuid",
	"applianceToken": "",
	"iotHost": "peter.channel.com"
	"iotPort": 5672
	"message": "error message..." 
}
```

#### Update IOT Task Results
POST /va/{appliance_id}/tasks

**Request Body**
```
{
	"taskId": "uuid",
	"taskStatus": "success | failed | ignored",
	"errorMessage": "",
	"taskResult": {
		"field1": ...
	}
}
```

#### IOT Health Check
GET /va/{appliance_id}/healthz

#### Upload Service Metrics
POST /va/{appliance_id}/{service_code>/metrics

### Upload Audit Log
POST /var/{appliance_id}/log


### API for frontend

<font color='red'>**Appliance Management**</font>

#### Get Appliance Image Info
GET /ui/appliances/image

**Procedure**
* check customer_id in request header
* query latest published version from DB
* recreate customer token if expired 
* get ova s3 download link valid for 7 days.
* response image info

**Response**

```
{
	"version": "1.0.0",
	"displayVersion": "1.0.0",
	"registrationToken": "",
	"packages": {
		"ovaDownloadLink": "string",
		"ovaName": "",
		"ovaSize": "string",
		"ovaSha256": "string",
	]
}
```

**Test**
add fake DB data
```
insert into app_applianceversion(version, created_at, updated_at, version_major, version_minor, version_revision, version_build, display_version, firmware_info, enable, ova_info, published) values('1.0.0.10000', now(), now(), 1, 0, 0, 10000, '1.0.0', '{}', 1, '{}', 1);
insert into app_applianceversion(version, created_at, updated_at, version_major, version_minor, version_revision, version_build, display_version, firmware_info, enable, ova_info, published) values('2.0.0.10000', now(), now(), 2, 0, 0, 10000, '2.0.0', '{}', 1, '{}', 1);
```
test API
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/image/"
```

#### List Appliance
GET /ui/appliances

**Procedure**
* check customer_id in request header
* parse applianceId, connectStartTime from request params
* Query appliances which expectedStatus are not unregistered from DB, then response

**Request Params**
| param | optional | example | description |
| ----- | -------- | ------- | ----------- |
| start | True | 1700020200 | filter register time of appliances |
| applianceId | True | uuid | |

**Response**
```
{
	"data": [
		{
			"applianceId": uuid,
			"hostname": "localhost",
			"ipv4Address": "1.2.3.4",
			"status": "preparing | running | disconnected | upgrading",
			"upgradeStatus": "downloading | upgrading",		// optional
			"logCollectStatus": "packaging | uploading",
			"version": "2.0.0",
			"installedServiceList": [
				{
					"serviceCode": "squid",
					"version": "1.1.1",
					"healthStatus": "healthy | unhealthy",
					"fullName": "Squid",
					"description": "Forward Proxy",
				},
			],
			"lastConnectedTime": 1700020200,
			"upgradeStartedTime": 1700020200, // if upgradeStartedTime exists, appliance is upgrading
			"totalStorage": 1024576,
			"usedStorage": 10240,
			"totalCpu": 8,
			"totalMemory": 16384
		}
	]
}
```

**Test**
```
# Query customer's appliances
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/"

# Query specified appliance
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/?start=1736680946"
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/?start=1736680946&applianceId=974bf535-7930-474e-8da6-780cafff284d"
```

#### Delete Appliance
DELETE /ui/appliances/<appliance_id>/

**Procedure**
* check customer_id in request header
* query delete appliance from DB, customer can only delete their own appliance 
* delete this appliance  
	* update expectedStatus and status to unregistered
	* send unregister iottask 

**Response**
empty

**Test**
```
curl -X DELETE -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/"
```


#### Upgrade Appliance
POST /ui/appliances/<appliance_id>/upgrade

#### Get Appliance Settings
GET /ui/appliances/<appliance_id>/settings

**Procedure**
* check customer_id in request header
* query appliance settings from DB, such as schedule time, cert info
* resp appliance settings

**Response**
```
{
	"nextUpdateTime": 164800000,
	"scheduleUpdateDay": 0 | 1,
	"scheduleUpdateTime": "23:00",
	"certInfo": {
		"subject": "",
		"issuedBy": "",
		"issueTo": "",
		"validFrom": "",
		"validTo": "",
		"expired": True | False,
		"certContent": ""
	},
	"defaultCertInfo": {
	
	}
}
```
**Test**
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/settings/"
```

#### Modify Appliance Settings
POST /ui/appliances/<appliance_id>/settings

**Procedure**
* check customer_id in request header, customer can only modify their own appliance's settings
* parse settings, update va settings to DB
* notify va, response task_id to   

**Request body**
```
{
	"certContent": "",
	"scheduleUpdateTime": "XX:XX",
	"hostname": ""
}
```
**Response**
```
{
	"taskId": ""
}
```
**Test**
```
curl -X POST -H "Content-Type:application/json" -d  '{"scheduleUpdateTime" : "23:00"}' -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" \
"https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/settings/"
```

#### Get Storage Details
GET /ui/appliances/<appliance_id>/storage

**Procedure**
* check customer_id in request header, customer can only get their own appliance's storage
* query storage details from DB table appliance_metric
* response storage details

**Response**
```
{
	"total": 1000000,
	"unallocated": 1000,
	"used": 1024,
	"usedDetail": {
		"data":	100,
		"image": 200,
		"system": 300,
	},
	"totalDetail": {
		"data": 1024,
		"image": 1024,
		"system": 1024,
	},
	"extendOngoing": true | false	# 上一个extendTask是否在执行中
}
```
**Test**
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" \
"https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/storage/"
```

#### Extend Storage
POST /ui/appliances/<appliance_id>/storage

**Procedure**
* check customer_id in request header, customer can only get their own appliance's storage
* validate request, parse extend storage detail
* if previous task is ongoing, response failure
* notify VA to extend storage
* response 200 OK

**Request**
```
{
	"data": [
		{
			"targetVolume": "data | image",
			"size": 400765
		}
	]
}
```

**Test**
```
curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -H "Content-Type:application/json" -d '{"data": [{"targetVolume": "data", "size": 1024}]}' \
"https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/storage/"
```

#### Get Appliance Metrics
POST /ui/appliances/<appliance_id>/metrics

#### Collect Appliance Log
POST /ui/appliances/<appliance_id>/log

**Procedure**
* parse customer_id in request header, check DB if customer is valid.
* get upload url(aws s3), 支持收某段时间的LOG(7天，2周，1个月)
* notify VA to collect logs
* wait log collected (30 minutes)
* resp download link

**Request**
```
{
	"downloadLink": "url"
}
```

**Test**
```
curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -H "Content-Type:application/json" \
"https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/log/"
```

<font color='red'>**Service Management**</font>

#### List Service
GET /ui/appliances/<appliance_id>/services

**Procedure**
* parse customer_id in request header, check if customer is valid, va belongs to this customer.
* query applianceServiceSettings DB table, find all services of this appliance. 
* response a list of services to frontend.

**Response**
```
{
	"data": [
		{
			"serviceCode": "va-squid",
			"version": "1.0.0.10000",
			"displayVersion": "1.0.0",
			"latestVersion": "A.B.C.DDDDD",		# latest avaliable service version 
			"latestDisplayVersion": "A.B.C",
			"status": "uninstalled | installing | running | disabled",
			"healthStatus": "healthy | unhealthy",
			"requestCPU": 1
			"requestMemory": 1024 
		}
	]
}
```

**Test**
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" \
"https://peter.backend.com/ui/appliances/974bf535-7930-474e-8da6-780cafff284d/services/"
```

#### Install/Uninstall/Upgrade Service
POST /ui/services/<service_code>/install

##### Install Service
**Procedure**
* parse customer_id in request header, check DB if customer is valid.
* find latest service version
* if service already installed, return OK, else: 
	* check if reach resources limit
	* get service package download url
	* create or update related dbentry, change expected status to running	
    * notify VA
* resp 200 OK 


**Install Service Request**
```
{
	"applianceId": "974bf535-7930-474e-8da6-780cafff284d",
	"action": "install | uninstall | upgrade"
}
```

**IOT Command**
```
{
	"serviceCode": "serviceCode",
    "taskType": "installService"
    "targetVersion": latest_version.version,
    "branch": latest_version.branch,
    "imagePath": download_url,
    "imageSha256": latest_version.package_sha256
}
```

**Test**
install service
```
insert into app_serviceinfo(service_code, created_at, updated_at, default_setting) values('va-squid', now(), now(), '{}'); 
insert into app_serviceversion(service_code, version, display_version, package_path, package_sha256, request_cpu, request_memory, disable, created_at, updated_at, branch, appliance_dependency, package_size) values \
('va-squid', '1.0.0.10000', '1.0.0', 'https://va-squid.tar.gz', '125a100301e72f3fe3a59cdab01a563ca239c7ea2728b4f39a44a3b6064d4286', 1, 1024*1024, 0, now(), now(), 'main', '', 0);

curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -d '{"applianceId": "974bf535-7930-474e-8da6-780cafff284d", "action": "install"}' -H "Content-Type:application/json"  \
"https://peter.backend.com/ui/services/va-squid/install/"
```
uninstall service
```
curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -d '{"applianceId": "974bf535-7930-474e-8da6-780cafff284d", "action": "uninstall"}' -H "Content-Type:application/json"  \
"https://peter.backend.com/ui/services/va-squid/install/"
```


#### Enable/Disable Service
POST /ui/services/<service_code>/<appliance_id>/toggle

**Procedure**
* parse customer_id from header, parse appliance_id, service_code from uri
* query service settings from DB
* change service status
* notify to VA

**Request**
```
{
	"action": "enable | disable"
}
```

**Response**
```
{
	"message": "success"
}
```
**Test**
```
curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -d '{"action": "enable"}' -H "Content-Type:application/json"  \
"https://peter.backend.com/ui/services/va-squid/974bf535-7930-474e-8da6-780cafff284d/toggle/"
```


#### Get Service Settings
GET /ui/services/<service_code>/<appliance_id>/settings

**Procedure**
* query service settings from DB based on appliance_id, service_code
* response to frontend

**Response**
```
{
	"settings": {}
}
```
**TEST**
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" "https://peter.backend.com/ui/services/va-squid/974bf535-7930-474e-8da6-780cafff284d/settings/"
```

#### Configure Service Settings
POST /ui/services/<service_code>/<appliance_id>/settings

**Procedure**
* parse settings from req body
* parse appliance_id, service_code from uri
* query current service settings from DB
* notify VA to configure service 

**Test**
```
curl -X POST -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" -H "Content-Type:application/json" -d '{"setting": {"settings101": "value101"}, "setting_version": 101}' \
"https://peter.backend.com/ui/services/va-squid/974bf535-7930-474e-8da6-780cafff284d/settings/"
```

#### Query Task result
GET /ui/tasks/<task_id>

**Procedure**
* parse customer_id from header, parse appliance_id, task_id from uri
* query task result from DB
* resp task result

**Response**
```
{
	"delivered": False, # if received from va, return True.
	"status": task.status,
	"result": task.result,
	"error_message": task.error_message
}
```

**Test**
```
curl -H "x-customer-id: 77a1c956-1c8d-47fb-8036-c970a4f2ee73" \
"https://peter.backend.com/ui/tasks/0c5b000b68ad4d3faedded836e92f24b/"
```