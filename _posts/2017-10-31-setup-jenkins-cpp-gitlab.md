---
layout: post
title: Cài đặt jenkins cho kết hợp với gitlab cho dự án c++, sử dụng docker    
image: 
categories: [c++, tool]
tags: [c++, gitlab, jenkins, ci, docker]
---

## Mục đích  
Cài đặt jenkins cho kết hợp với gitlab cho dự án c++, sử dụng docker.  

## Cài đặt jenkins lên docker container  
```bash
# Trên máy host 
#
docker pull ubuntu:16.04
docker run -d --name jenkins_build -p8080:8080 --restart=always -it ubuntu:16.04
docker attach jenkins_build

# Trên container jenkins_build
#
# Cài đặt jenkins 
# export http_proxy=http://192.168.1.123:1611
# export https_proxy=http://192.168.1.123:1611
apt update; apt upgrade -y
apt install nano sudo wget git subversion
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | apt-key add -
sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
apt-get update
apt-get install jenkins -y
# Cài đặt jenkins
/etc/init.d/jenkins start

# Trên host 
# Vào browser, truy cập địa chỉ http://<container_ip>:8080
# Đợi tới khi jenkins sẵn sàng 

# Trên container
# Copy mật khẩu trong 'initialAdminPassword' vào 'Administrator password' trên browser của host 
cat /var/lib/jenkins/secrets/initialAdminPassword

# Trên host
# Cài đặt các plugin cần thiết 
#
#

# Trên container
mkdir -p /var/lib/jenkins
chmod -R 777 /var/lib/jenkins

cat << EOF >> /bootstrap.sh
#!/bin/sh
/etc/init.d/jenkins start
/etc/init.d/ssh start
while true
do
  sleep 1s
done
EOF
chmod +x /bootstrap.sh
# Cài đặt các gói phục vụ build dự án
# ...
#

# Trên host
docker commit jenkins_build jenkins
```

Tạo container từ jenkins images 
```bash
docker run -d --name jenkins -p8080:8080 -p2201:22 -v/home/shared:/home/shared --restart=always -it jenkins /bootstrap.sh
```

## Kết nối gitlab và jenkins  
Trên jenkins:  
Manage Jenkins > Configure System > Gitlab > GitLab connections  
- Connection name : local-gitlab
- Gitlab host URL : http://192.168.1.2
- Credentials: Add > Jenkins 
  * Domain : Global credentials
  * Kind: Gitlab API token
  * Token lấy từ 'Access Tokens' của gitlab
- Chú ý: bỏ thiết lập proxy trong Manage Jenkins > Manage Plugins > Advanced

## Tạo project  
Trên jenkins:
New item > Freestyle project 
- Source Code Management > Git : điền đường dẫn tới dự án, sử dụng username và passwd của jenkins user trong gitlab
- Build Triggers:
  * Chọn các event phù hợp 
  * Secret token : Generate 

Trên gitlab:  
- Tạo người sử dụng 'jenkins' với quyền: 
- Truy cập project > Settings > Integrations
  * URL : http://192.168.1.2:8080/project/<project-name> # xem trong 'Build Triggers' của jenkins
  * Secret Token: sử dụng Secret token trong 'Build Triggers' của jenkins 
  * Bỏ chọn 'Enable SSL verification'

## Tham khảo  
- https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Ubuntu
- https://wiki.jenkins.io/display/JENKINS/JenkinsBehindProxy
- https://docs.bitnami.com/bch/how-to/create-ci-pipeline/
- https://docs.gitlab.com/ee/integration/jenkins.html


