---
title: "nmap扫描探测数据库学习"
date: "2024-03-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

`sudo nmap -sV -T4 -A -n $1 -p $2`


![img_6.png](/pic/img_6.png)
nmap6.4版本扫不出service 7.92版本可以

```js
base_path=$(cd `dirname $0`; pwd)
sudo nmap -sV -T4 -A -n --script $base_path/a.nse $1 -p $2 | grep "depth" | awk '{ print $1 " " $2 " " $3 " " $4 }' > $base_path/$3
```
```shell
description = [[Database depth scanner. Support: oracle/sqlserver/mysql/db2/postgresql/sybase]]
license = "none"
categories = {"default"}

startwith = function(service, pre)
     return string.sub(service, 1, string.len(pre)) == pre
end

portrule = function(host, port)
    return port.protocol == "tcp"
           and (startwith(port.service, "oracle") or startwith(port.service, "ms-sql")
           or startwith(port.service, "mysql") or startwith(port.service, "postgresql")
		   or startwith(port.service, "ibm-db2") or startwith(port.service, "sybase-adaptive"))
           and port.state == "open"
end

action = function(host, port)
    if (startwith(port.service, "oracle")) then
		return "oracle " .. host.ip .. " " .. port.number
    end
	if (startwith(port.service, "ms-sql")) then
		return "Mssql " .. host.ip .. " " .. port.number
    end
	if (startwith(port.service, "mysql")) then
		return "mysql " .. host.ip .. " " .. port.number
    end
	if (startwith(port.service, "ibm-db2")) then
    	return "db2 ".. host.ip .. " " .. port.number
    end
	if (startwith(port.service, "postgresql")) then
    	return "Pgsql ".. host.ip .. " " .. port.number
    end
	if (startwith(port.service, "sybase-adaptive")) then
    	return "sybase ".. host.ip .. " " .. port.number
    end
end

```