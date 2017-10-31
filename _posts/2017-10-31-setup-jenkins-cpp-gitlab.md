---
layout: post
title: Cài đặt jenkins cho kết hợp với gitlab cho dự án c++    
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
docker run -d --name jenkins_build -p8080:8080 -p2201:22 -v/home/shared:/home/shared -v/home/jenkins:/var/lib/jenkins --restart=always -it ubuntu:16.04
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
# Thêm vào đầu file /etc/init.d/jenkins
# LANG="en_US.UTF-8"
/etc/init.d/jenkins start

# Trên host 
# Vào browser, truy cập địa chỉ http://<container_ip>:8080
# Đợi tới khi jenkins sẵn sàng 

# Trên container
# Copy mật khẩu trong 'initialAdminPassword' vào 'Administrator password' trên browser của host 
cat /var/lib/jenkins/secrets/initialAdminPassword

# Trên trang cấu hình jenkins 
# Cài đặt các plugin cần thiết từ internet (recommented: select all) 
# Có thể cần cấu hình proxy nếu không vào được internet 
#

# Trên container
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
docker run -d --name jenkins -p8080:8080 -p2201:22 -v/home/shared:/home/shared -v/home/jenkins:/var/lib/jenkins --restart=always -it jenkins /bootstrap.sh
```

## Cài đặt gitlab
- Tạo 1 tài khoản jenkins với các cấu hình:
  * Projects limit : 0
  * Access level : Regular
  * External : true
- Thêm tài khoản jenkins vào project cần kết nối 
- Tạo Access Tokens của tài khoản jenkins 
  * Đăng nhập vào gitlab với tài khoản jenkins
  * Vào Settings > Access Tokens > Create personal access token [select all scopes]

## Kết nối gitlab và jenkins  
Trên jenkins:  
Bỏ thiết lập proxy trong Manage Jenkins > Manage Plugins > Advanced
Manage Jenkins > Configure System > Gitlab > GitLab connections  
- Connection name : local-gitlab
- Gitlab host URL : http://<gitlab_ip:port>
- Credentials: Add > Jenkins 
  * Domain : Global credentials
  * Kind: Gitlab API token
  * Token lấy từ 'Access Tokens' của gitlab
- Test Connection : Success là thành công  
- Save 

## Tạo project  
Trên jenkins:
New item > Freestyle project 
- Source Code Management > Git : điền đường dẫn tới dự án, sử dụng username và passwd của jenkins user trong gitlab
  * Additional Behavious:
    * Advanced sub-modules behavious
      * Recursively update submodules : checked
- Build Triggers:
  * Build when a change is pushed to GitLab
    * Chọn các event phù hợp 
    * Advanced > Secret token : Generate : copy token vừa tạo được 
    * Copy đường dẫn sau dòng GitLab CI Service URL
- Build
  * Add build step > Execute shell
    * Command : nhập lệnh bash shell sử dụng để build và chạy unittest 
- Post-build Actions : Thao tác sau khi build và chạy unittest xong 
  * Scan for compiler warnings : Ghi lại cảnh báo khi biên dịch 
  * Publish HTML reports : Ghi lại thư mục report, ví dụ:
    * HTML directory to archive: test/report/coverage
    * Index page[s]: index.html
    * Index page title[s] (Optional): coverage
    * Report title: coverage
- Save
    
Trên gitlab:  
- Tạo người sử dụng 'jenkins' với quyền: 
- Truy cập project > Settings > Integrations
  * URL : đường dẫn 'GitLab CI Service URL' copy từ jenkins
  * Secret Token: 'Secret token' copy từ jenkins
  * Bỏ chọn 'Enable SSL verification'
  * Add webhook 
  * Test > Hook executed successfully: HTTP 200 là thành công 

## Tham khảo  
- https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Ubuntu
- https://wiki.jenkins.io/display/JENKINS/JenkinsBehindProxy
- https://docs.bitnami.com/bch/how-to/create-ci-pipeline/
- https://docs.gitlab.com/ee/integration/jenkins.html


