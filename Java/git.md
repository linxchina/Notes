# Jenkins构建脚本

```shell
## 1、将私钥告诉jenkins，gitlab上配公钥。立即构建会自动下载到workspace下。项目配置里，最下面构建里选择执行shell   参数填入 sh -x /script/deploy.sh（注意权限，jenkins是用什么用户运行的）。
## 2、Jenkins 机器执行ssh -copy-id ~/.ssh/id_rsa.pub root@10.1.1.1(上线机器)  可以ssh root@10.1.1.1检查是否能连上。
## 3、Jenkins变量
## 4、Jenkins 参数化构建过程 添加参数
1，文本参数：选择版本: 名称 git_version  默认值：v1.1 描述
2，选项参数: 选择部署或回退: 名称 deploy_env  选项：deploy   rollback  描述

3,源码管理里，将master改成 ${git_version}
### 5、gitlab加版本
git pull origin master
git tag -a "v1.1" -m "ttt"
git push origin v1.1
## 修改文件
git add .
git commit =m "v1.2"
git push origin master
git tag -a "v1.2" -m "v1.2"
git push origin v1.2
```
```sh
##
Webserver="10.1.1.1 10.2.2.2"
Sdir=/opt
Ddir=/code

### deploy.sh
#1 将项目打包
get_code(){
    cd &{WORKSPACE} && \
    tar czf /${Sdir}/web-${DATE}.tar.gz  ./*
}
#2 scp拷贝到集群
scp_web_server(){
    for hosts in $Webserver:
    do
    scp /opt/web-${DATE}.tar.gz root@{hosts}:/opt
    ssh  root@{hosts} "mkdir -p /${Ddir}/web-${DATE} &&\
                        tar -xf /${Sdir}/web-${DATE}.tar.gz -C /${Ddir}/web-${DATE}
                        rm -rf /${Ddir}/web && \
                        ln -s /${Ddir}/web-${DATE} /${Ddir}/web"
    done
}

deploy(){
    get_code
    scp_web_server
}
deploy
```

```sh
### deploy_tag_rollbak.sh

DATE=${date +%Y-%m-%d-%H-%M-%S}
Webserver="101.1.1.1 10.2.2.2"
Sdir=/opt
Ddir=/code
## Name=$DATE-${git_version}
#1 将项目打包
get_code(){
    cd &{WORKSPACE} && \
    tar czf /${Sdir}/web-${DATE}-${git_version}.tar.gz  ./*
}
#2 scp拷贝到集群
scp_web_server(){
    for hosts in $Webserver:
    do
    scp /opt/web-${DATE}-${git_version}.tar.gz root@{hosts}:/opt
    ssh  root@{hosts} "mkdir -p /${Ddir}-${git_version}/web-${DATE} &&\
                        tar -xf /${Sdir}/web-${DATE}-${git_version}.tar.gz -C /${Ddir}/web-${DATE}-${git_version}
                        rm -rf /${Ddir}/web && \
                        ln -s /${Ddir}/web-${DATE}-${git_version} /${Ddir}/web"
    done
}
rollback(){
rollback_file=$(ssh root@10.1.1.1 "find /code/ -maxdepth 1 -type d -name "web-&-${git_version}"") ##需要保证只能查出一条，不能有重复部署（同一个版本）的问题，使用git_commit和git_previous_successful_commit变量判断
for hosts in $Webserver:
do
ssh  root@{hosts} "rm -rf /${Ddir}/web && \
                   ln -s ${rollback_file} /${Ddir}/web"
done
}
deploy(){
    get_code
    scp_web_server
}



if [ $deploy_env == "deploy" ];then
    if [ ${$GIT_COMMIT} == ${GIT_PREVIOUS_SUCCESSFUL_COMMIT} ]; then
        echo "你已经部署过此版本${GIT_COMMIT}"
        exit 1
0    else
        deploy
    fi
elif [ $deploy_env == "rollback" ];then
    rollback
fi
```

```sh
mvn package -Dmaven.test.skip=true


get_code(){
    cd &{WORKSPACE}
}
#2 scp拷贝到集群
scp_web_server(){
    for hosts in $Webserver:
    do
    scp target/*.war  root@{hosts}:/opt/ROOT-${DATE}-${git_version}.war
    ssh  root@{hosts} "mkdir -p /${Ddir}-${busi}-${DATE}-${git_version} &&\
                        unzip  /opt/ROOT-${DATE}-${git_version}.war  -d  /${Ddir}-${busi}-${DATE}-${git_version}
                        rm -f /${Ddir}/ROOT && \  ##删除此前已存在的软链接
                        ln -s /${Ddir}-${busi}-${DATE}-${git_version}   /${Ddir}/ROOT  && \
                        ###重启"
    done
}
rollback(){
rollback_file=$(ssh root@10.1.1.1 "find /code/ -maxdepth 1 -type d -name "web-&-${git_version}"") ##需要保证只能查出一条，不能有重复部署（同一个版本）的问题，使用git_commit和git_previous_successful_commit变量判断
for hosts in $Webserver:
do
ssh  root@{hosts} "rm -f /${Ddir}/web && \
                   ln -s ${rollback_file} /${Ddir}/web
                     ###重启 "
done
}
deploy(){
    get_code
    scp_web_server
}



if [ $deploy_env == "deploy" ];then
    if [ "${GIT_COMMIT}" == "${GIT_PREVIOUS_SUCCESSFUL_COMMIT}" ]; then
        echo "你已经部署过此版本${GIT_COMMIT}"
        exit 1
0    else
        deploy
    fi
elif [ $deploy_env == "rollback" ];then
    rollback
fi
```