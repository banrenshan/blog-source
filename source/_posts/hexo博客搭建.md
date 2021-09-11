---
title: hexo博客搭建
tags: 
 - hexo
categories:
 - 工具
---



# 1.下载hexo

1. 安装git
2. 安装nodejs
3. 安装hexo: `npm install -g hexo-cli`

# 2.初始化项目

1. 创建项目： `hexo init blog`
2. 安装依赖： 进入blog目录，执行 `npm install`
3. 安装主题：`git clone https://github.com/Shen-Yu/hexo-theme-ayer.git themes/ayer`
4. 修改根目录下面的_config.yml，指定theme为ayer
5. 启动项目：`hexo s`

# 3.配置gitee

1. 安装git插件：`npm install hexo-deployer-git --save`

2. 配置_config.yaml文件：

```yaml
deploy:
  type: git
  repository: https://gitee.com/hanziqi/blog.git
  branch: master
```



# 4.其他配置

1. 支持标签：`hexo new page tags`,创建的文件(source\tags\index.md)内容：

   ```shell
   ---
   title: 标签
   type: "tags"
   layout: "tags"
   ---
   ```

   

2. 支持分类：`hexo new page categories`,创建的文件(source\categories\index.md)内容如下：

   ```shell
   ---
   title: 分类
   type: "categories"
   layout: "categories"
   ---
   ```

3. 支持搜索：`npm install hexo-generator-searchdb --save`

4. 支持rss订阅： `npm install hexo-generator-feed --save`

5. 支持图片：`npm install https://github.com/CodeFalling/hexo-asset-image --save`,然后修改配置文件：

   ```yaml
   post_asset_folder: true
   ```

