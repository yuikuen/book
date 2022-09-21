# GitLab Install(Docker)

## 下载镜像

> 根据实际需求下载，可到 [Hub.Docker](https://hub.docker.com/r/gitlab/gitlab-ce/tags) 下载

```bash
$ docker search gitlab/gitlab
NAME                             DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
gitlab/gitlab-ce                 GitLab Community Edition docker image based …   3697                 [OK]
gitlab/gitlab-runner             GitLab CI Multi Runner used to fetch and run…   808                  [OK]
gitlab/gitlab-ee                 GitLab Enterprise Edition docker image based…   352                  
gitlab/gitlab-runner-helper                                                      39                   
gitlab/gitlab-ce-qa              GitLab QA has a test suite that allows end-t…   8                    
gitlab/gitlab-build-images       Please go to: https://gitlab.com/gitlab-org/…   3                    
gitlab/gitlab-dns                Managing GitLab's External DNS                  2                    
gitlab/gitlab-ee-qa              GitLab QA has a test suite that allows end-t…   2                    
gitlab/gitlab-performance-tool   GitLab QA performance test tool - https://gi…   1                    
gitlab/gitlab-qa                 GitLab QA has a test suite that allows end-t…   0                    
gitlab/gitlab-agent-ci-image     CI image for verifying and testing the gitla…   0  

$ docker pull gitlab/gitlab-ce:15.3.3-ce.0
```

## 数据持久化

容器的数据默认是不能持久化保存的，所以需要用 `docker volume` 的方式将存储的数据映射到操作系统的目录中来

```bash
$ docker run -detach \  
  --publish 443:443 --publish 80:80 --publisth 2222:22 \ 
  --name gitlab \
  --restart always \
  --volume /home/gitlab/config:/etc/gitlab \
  --volume /home/gitlab/logs:/var/log/gitlab \
  --volume /home/gitlab/data:/var/opt/gitlab \
  --volume /etc/localtime:/etc/localtime \
  --privileged=true gitlab/gitlab-ce:15.3.3-ce.0 
```

