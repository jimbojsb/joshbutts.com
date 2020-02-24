+++
date = 2020-02-24T00:00:00Z
slug = "tableplus-shortcuts"
tags = ["docker", "bash"]
title = "A quick Bash function to Open TablePlus"
draft = true
+++

```bash
function tp {
	PORT=$(docker-compose port mysql 3306 | cut -d: -f2)
	NAME=$(basename $(pwd))
	DATABASE=$(cat docker-compose.yml | grep MYSQL_DATABASE | cut -d: -f2 | xargs)
	open "mysql://root:root@127.0.0.1:$PORT/$DATABASE?Enviroment=local&Name=$NAME&statusColor=DAEBC2"
}
```