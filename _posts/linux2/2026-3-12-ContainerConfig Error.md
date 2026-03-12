RROR: for wordpress2 'ContainerConfig' Traceback (most recent call last):
 

 原因 ：**容器/镜像的旧缓存状态坏了**
## 解决办法
      ###  停掉并删除旧容器
		      sudo docker-compose down
	  ###  删除残留容器
	          sudo docker rm -f wordpress wordpress2 mysql cadvisor 2>/dev/null
	  ###  清理旧网络
	          sudo docker network prune -f
	  ###  重新启动 
			  sudo docker-compose up -d
# 如果还报 `ContainerConfig`

说明 **旧镜像 metadata 坏了**，再执行：

      docker-compose pull
      docker-compose up -d
      或者
      docker-compose up -d --force-recreate
      
## 为什么会出现这个问题

     **你之前升级/降级 compose**  
     **容器没删干净**  
     **镜像 metadata 不兼容**


