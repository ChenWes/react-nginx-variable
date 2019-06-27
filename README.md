## 处理nginx包裹的react应用程序中变量的问题

## 第一步：创建应用

```
npx create-react-app react-nginx-variable
cd react-nginx-variable
```

## 第二步：创建.env文件
通过命令创建.env文件，并写入内容
```
touch .env
echo "API_URL=https//default.dev.api.com" >> .env
```


## 第三步：写入env.sh脚本
env.sh脚本内容
```
#!/bin/bash

# Recreate config file
rm -rf ./env-config.js
touch ./env-config.js

# Add assignment 
echo "window._env_ = {" >> ./env-config.js

# Read each line in .env file
# Each line represents key=value pairs
while read -r line || [[ -n "$line" ]];
do
  # Split env variables by character `=`
  if printf '%s\n' "$line" | grep -q -e '='; then
    varname=$(printf '%s\n' "$line" | sed -e 's/=.*//')
    varvalue=$(printf '%s\n' "$line" | sed -e 's/^[^=]*=//')
  fi

  # Read value of current variable if exists as Environment variable
  value=$(printf '%s\n' "${!varname}")
  # Otherwise use value from .env file
  [[ -z $value ]] && value=${varvalue}
  
  # Append configuration property to JS file
  echo "  $varname: \"$value\"," >> ./env-config.js
done < .env

echo "}" >> ./env-config.js
```

并修改package.json中的运行脚本为
```
    "dev": "chmod +x ./env.sh && ./env.sh && cp env-config.js ./public/ && react-scripts start",
    "test": "react-scripts test",
    "eject": "react-scripts eject",
    "build": "sh -ac '. ./.env; react-scripts build'"
```


## 第四步：修改public/index.html
index.html增加脚本标签引用脚本
```
<script src="%PUBLIC_URL%/env-config.js"></script>
```

## 第五步：修改src/App.js
增加代码显示环境变量内容
```
<p>API_URL: {window._env_.API_URL}</p>
```


## 第六步：本地运行应用
使用命令测试应用
```
yarn dev
```


## 第七步：使用Azure DevOps打包并发布镜像
azure-pipelines.yml内容
```
# Docker image
# Build a Docker image to deploy, run, or push to a container registry.
# Add steps that use Docker Compose, tag images, push to a registry, run an image, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- development

pool:
  vmImage: 'Ubuntu-16.04'

variables:
  imageName: 'workbench:$(build.BuildNumber)'

steps:
- script: docker build -f Dockerfile -t $(imageName) .
  displayName: 'docker build'
- script: | 
    docker login -u $(dockerId) -p $(pswd)
    docker tag $(imageName) $(dockerId)/$(imageName)  
    docker push $(dockerId)/$(imageName)
  displayName: 'push docker image'
```


## 第八步：使用docker-compose运行容器
docker-compose.yml内容
```
version: "3.2"
services:
  web:
    image: weschen/workbench:20190627.2
    ports:
      - "5000:80"
    environment:
      - "API_URL=weschen.production.example.com"

```
在服务器中使用命令布署应用
```
docker-compose up -d
```