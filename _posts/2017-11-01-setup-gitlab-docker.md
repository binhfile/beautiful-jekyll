---
layout: post
title: Cài đặt gitlab trên docker 
image: 
categories: [c++, tool]
tags: [c++, gitlab, ci, docker]
---

## Mục đích  
Cài đặt gitlab trên docker.  

## Cài đặt gitlab lên docker container  
```bash
# Trên máy host 
#
mkdir -p /home/gitlab/config /home/gitlab/logs /home/gitlab/data
docker pull gitlab/gitlab-ce
docker run -d \
 --name gitlab \
 --hostname gitlab.tcdt.vrd.vn \
 -p80:80 \
 -p443:443 \
 -p2200:22 \
 -v/home/shared:/home/shared \
 -v/home/gitlab/config:/etc/gitlab:Z \
 -v/home/gitlab/logs:/var/log/gitlab:Z \
 -v/home/gitlab/data:/var/opt/gitlab:Z \
 --restart=always -it gitlab/gitlab-ce
# Đợi tới khi container khởi động xong 
```

Trên browser của host, truy cập địa chỉ `http://<container_ip>`. 
- Nhập mật khẩu cho tài khoản root
- Đăng nhập với tài khoản root và mật khẩu vừa thiết lập 

## Backup
```bash
# Make sure container is running
docker exec -t gitlab_old gitlab-rake gitlab:backup:create
# gitlab will create backup file in /var/gitlab/data/backups/
# copy it and copy file
# /var/gitlab/config/gitlab-secrets.json
```

## Restore
```bash
# copy backup file to directory
cp 1509555926_2017_11_01_9.3.0_gitlab_backup.tar /home/gitlab/data/backups/
docker exec -t gitlab gitlab-ctl stop unicorn
docker exec -t gitlab gitlab-ctl stop sidekiq
# Verify
docker exec -t gitlab gitlab-ctl status
docker exec -t gitlab gitlab-rake gitlab:backup:restore BACKUP=1509555926_2017_11_01_9.3.0
# copy gitlab-secrets.json
cp gitlab-secrets.json /home/gitlab/config/
docker exec -t gitlab gitlab-ctl restart
docker exec -t gitlab gitlab-rake gitlab:check SANITIZE=true

```

## Tham khảo  
- https://developer.ibm.com/code/2017/07/13/step-step-guide-running-gitlab-ce-docker
- https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md


