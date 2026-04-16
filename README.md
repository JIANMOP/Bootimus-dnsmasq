# Bootimus-dnsmasq

```bash
git clone https://github.com/JIANMOP/Bootimus-dnsmasq.git
cd Bootimus-dnsmasq/
mkdir data/
# 修改 docker-compose.yml 和 dnsmasq.conf 里的 ip 为你的 ip
docker compose up -d
# 浏览器访问 http://{你的ip}:8081 	bootimus 控制台 
docker logs bootimus | grep "Password"	# 初始账户 admin 密码
# 浏览器访问 http://{你的ip}:8080	dnsmasq 控制台
```
