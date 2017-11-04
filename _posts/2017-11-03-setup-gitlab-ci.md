---
layout: post
title: Cài đặt gitlab-ci trên docker 
image: 
categories: [c++, tool]
tags: [c++, gitlab, ci, docker]
---

Gitlab tích hợp sẵn công cụ CI là gitlab-ci. Mô hình hoạt động như sau:  
Mỗi khi có event(push, merge...) gitlab sẽ trigger gitlab-runner. 
gitlab và gitlab-runner chạy trên 2 container khác nhau. Gitlab-runner sẽ pull một image 
được cấu hình từ local docker hoặc từ internet về và thực hiện các câu lệnh được khai báo trong 
`gitlab-ci.yml` sau đó trả về kết quả trong Pipelines của gitlab.

## Cài đặt gitlab-ci lên docker container
```source-shell
# Trên máy host 
#
docker pull gitlab/gitlab-runner
# Map /var/run/docker.sock để gitlab-runner có thể pull docker images từ docker của host 
docker run -d --name gitlab-runner --restart always \
  -v /home/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
  
docker exec -it gitlab-runner-test gitlab-runner register
# Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
#   Nhập địa chỉ của gitlab 
#   http://192.168.1.2
# Please enter the gitlab-ci token for this runner
#   Nhập token lấy từ Settings > Pipelines > Specific Runners trên gitlab
# Please enter the gitlab-ci description for this runner:
#   Nhập tên của runner sẽ được hiển thị trong gitlab 
#   gitlab-ci
# Please enter the executor:...
#   docker
# Please enter the default Docker image (e.g. ruby:2.1):
#   Nhập tên của docker images sẽ được runner pull về, ta sẽ tự tạo image này trong localhost của docker 
#   gitlab-ci-test:latest
docker exec -it gitlab-runner-test gitlab-runner start
docker exec -it gitlab-runner-test gitlab-runner status

# Sửa tập tin /home/gitlab-runner/config/config.toml
#   Cho phép runner pull docker images từ localhost
# nano /home/gitlab-runner/config/config.toml
#   [runners.docker]
#      pull_policy = "if-not-present"
```

## Tạo docker images sử dụng cho runner  
```bash
docker run -d --name gitlab-ci -it ubuntu:16.04
docker attach gitlab-ci

# on gitlab-ci container
export http_proxy=http://192.168.1.123:1611
export https_proxy=http://192.168.1.123:1611
apt update; apt upgrade -y
apt install -y wget gcc g++ make cmake nano
# Cài đặt các gói cần thiết để thực hiện test, build, deploy...
# Ctrl+p, Ctrl + q to deattach from container

# on host
# Commit gitlab-ci thành images ở localhost của docker 
docker commit gitlab-ci gitlab-ci
```

## Cấu hình gitlab
Vào project, Setting > Pipelines
- Shared Runners
  * Disable shared Runners

Cấu hình external ip cho gitlab hoặc có thể sử dụng tùy chọn `--hostname 192.168.1.2` 
khi tạo container
```bash
docker exec -it gitlab vi /etc/gitlab/gitlab.rb
#   external_url 'http://192.168.1.2'
docker restart gitlab
```

## Tích hợp hiển thị coverage trong gitlab
- Sử dụng `gcovr -r .` để hiển thị tỉ lệ kiểm thử ra stdout
- Trong gitlab: Setting > Pipelines > Test coverage parsing : ^TOTAL.*\s+(\d+\%)$

## Một số ưu nhược điểm của gitlab-ci khi so sánh với jenkins  
- Tích hợp sâu vào gitlab, hiển thị rõ ràng các bước hoạt động, đẹp hơn jenkins
- Tích hợp hiển thị phần trăm coverage :)
- Không publish được các trang HTML: coverage, cppcheck, pvs check...

Tập tin `gitlab-ci.yml` ví dụ  
```yml
before_script:
  - gcc --version
  - g++ --version
  - mkdir -p report
  
stages:
  - test
  - deploy
  
test:
  stage: test
  script:
    - g++ -fprofile-arcs -ftest-coverage main.cpp -o test
    - ./test
    - gcovr -r .
  artifacts:
    paths:
      - report/
pages:
  stage: deploy
  dependencies:
    - test
  script:
    - mv report/ public/
  artifacts:
    paths:
      - public
```

## Tham khảo  
- https://docs.gitlab.com/runner/install/linux-repository.html
- https://docs.gitlab.com/runner/register/index.html


