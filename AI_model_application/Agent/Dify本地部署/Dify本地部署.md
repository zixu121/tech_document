## 1.安装docker应用
进入docker官网:https://www.docker.com/ 选择适合自己版本的docker应用下载。

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309422716-a47033d4-edb3-4b66-b500-86cc04a13c34.png)

安装完成之后就可以直接看到桌面上docker应用了

## 2.安装git应用（可选）
进入git官网中：https://git-scm.com/ ，选择自己电脑对应的版本下载

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309422794-3d2df563-a16c-4f5f-8dde-f1454750b028.png)

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309422870-eeafb251-e5cf-4fae-ad64-e6d759008b89.png)

## 3.克隆Dify代码仓库
输入：https://github.com/langgenius/dify进入github中找到dify对应的仓库

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309422759-fc2df2b3-9e4a-44d1-a042-9b1011dff6cc.png)

在文件管理器中创建一个空的文件夹

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309422745-1243834a-175b-4d32-91dd-4c7df215d4af.png)

进入该文件后，右键空白地方选择鼠标选中的地方会打开一个git命令窗口

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423238-a81da7e3-ecf7-41a8-8c06-a1c3381b4ee6.jpeg)

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423366-6e332c8b-629d-40bd-9700-4274300e69f3.png)

输入以下命令进行dify项目拉取，等待完成

```plain
git clone https://github.com/langgenius/dify.git
```

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423395-7a8ab37d-fa2e-47d8-be98-31a711877189.png)

**如果git下载的比较慢，可以直接下载zip文件。**

打开文件夹管理器，进入dify源码的目录

```plain
cd dify/docker
```

复制配置文件，手动将.env.example复制一份成.env文件

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423450-aace9df2-e011-4636-85f8-6470d1669085.png)

接下来打开cmd命令行，进入dify/docker文件夹中，输入下面命令加载项目到docker中

```plain
docker-compose up -d
```

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423504-133dcfcc-000a-4c1d-b0a2-691b9f3b7b33.png)

完成后，会在docker中启动

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423927-c5787ff3-74c6-49d3-91b0-662309ed47d1.png)

## 4.初始化dify设置
打开http://localhost/install 网址设置管理员账号，会让你输入对应的邮箱、用户名和密码

接下来登录账户即可，至此dify已经本地部署完毕。

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309423937-d1b668f2-026c-499d-a7e7-54e78b2548c2.png)

## 5.配置对应的大模型
点击设置，选择模型供应商

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309424025-0c803c98-387f-4775-928b-7e0d6b89935f.png)

可以配置目前大部分的模型厂家，也可以部署自己本地ollama部署的模型，只需要选择对应的厂商安装即可。

设置对应大模型厂商的key

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309424032-3c6c2e6d-22df-4476-b840-6ca6e20fec02.png)

设置本地embedding模型，使用的是ollama本地部署的模型

<!-- 这是一张图片，ocr 内容为： -->
![](./images/1768309424220-f542a196-9ff1-40f7-a8cd-b070acb88343.png)

