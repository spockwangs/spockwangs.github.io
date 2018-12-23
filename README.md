## Introduction

这个repository有2个分支：source用于存放源代码，master分支存放根据源代码生成的用于部署的HTML和CSS.

## Install dependencies

```
$ gem install bundler
$ bundle install
$ rake install
```

## How to write a post

1. 进入source分支
```
$ git checkout source

2. 新建post
```
$ rake new_post["title"]
```
生成的新文章在`source/_posts/`下，用编辑器打开开始写文章。

3. 写完后可用本地浏览器预览
```
$ rake preview
```

4. 预览OK后发布至Github
```
$ rake generate
$ rm -rf _deploy/*
$ cp -r public/* _deploy
$ git co master
$ cp -r _deploy/* .
$ git ci -a -m xxx
$ git push
```
这会把生成好的文件复制到master分支并push给Github.

5. 最后把本地编辑的源文件提交到Github
```
$ git co source
$ git commit -a -m "xxx"
$ git push
```
