+++
keywords = [
  "hugo",
  "github",
]
author = "asdfsx"
date = "2016-12-06T19:29:11+08:00"
title = "使用hugo在github上搭建blog"
tags = [
  "hugo",
  "github",
]
draft = false
type = "post"
topics = [
  "topic 1",
]
description = "hugo的使用"

+++

安装golang的开发环境  

> brew install golang  
> cat "export GOPATH=/root/gocode" >> ~/.bash_profile  
> cat "export PATH=${GOPATH}/bin:$PATH" >> ~/.bash_profile

安装hugo  

> go get -v github.com/spf13/hugo

```
源码会下载到$GOPATH/src下
可执行文件在$GOPATH/bin下
如果没有生成可执行文件，就使用go install手动安装一下hugo
```

创建一个站点  

> hugo new site localhost

```
执行命令的目录下会生成一个新目录localhost
该站点的所有文件都保存在该目录下
目录结构如下
.
├── archetypes
├── config.toml
├── content
├── data
├── layouts
├── static
└── themes
```

给localhost下载一个theme  

> cd localhost/themes  
> git clone https://github.com/enten/hyde-y.git'

修改localhost站点的配置, 用下边的值覆盖config.toml  

> vi config.toml

```
# hostname (and path) to the root eg. http://spf13.com/
baseurl = "http://asdfsx.github.io"

# Site title
title = "asdfsx"

# Copyright
copyright = "(c) 2016 asdfsx."

# Language
languageCode = "en-EN"

# Metadata format
# "yaml", "toml", "json"
metaDataFormat = "toml"

# Theme to use (located in /themes/THEMENAME/)
theme = "hyde-y"

# Pagination
paginate = 10
paginatePath = "page"

# Enable Disqus integration
#disqusShortname = "your_disqus_shortname"

#[permalinks]
#    post = "/:year/:month/:day/:slug/"
#    code = "/:slug/"

[taxonomies]
    tag = "tags"
    topic = "topics"

[author]
    name = "yourname"
    email = "yourname@example.com"

#
# All parameters below here are optional and can be mixed and matched.
#
[params]
    # You can use markdown here.
    brand = "asdfsx"
    topline = "sam's bodega"
    footline = "code with <i class='fa fa-heart'></i>"

    # Sidebar position
    # false, true, "left", "right"
    sidebar = "left"

    # Text for the top menu link, which goes the root URL for the site.
    # Default (if omitted) is "Home".
    home = "home"

    # Select a syntax highight.
    # Check the static/css/highlight directory for options.
    highlight = "default"

    # Google Analytics.
    googleAnalytics = "Your Google Analytics tracking code"

    # Sidebar social links.
    github = "enten/hugo-boilerplate" # Your Github profile ID
    bitbucket = "" # Your Bitbucket profile ID
    linkedin = "" # Your LinkedIn profile ID (from public URL)
    googleplus = "" # Your Google+ profile ID
    facebook = "" # Your Facebook profile ID
    twitter = "" # Your Twitter profile ID
    youtube = ""  # Your Youtube channel ID
    flattr = ""  # populate with your flattr uid
    flickr = "" # Your Flickr profile ID
    vimeo = "" # Your Vimeo profile ID

    # Sidebar RSS link: will only show up if there is a RSS feed
    # associated with the current page
    rss = true

[blackfriday]
    angledQuotes = true
    fractions = false
    hrefTargetBlank = false
    latexDashes = true
    plainIdAnchors = true
    extensions = []
    extensionmask = []
```

修改侧边栏的菜单, 用下边的覆盖data/Menu.toml

> vi data/Menu.toml

```
[about]
    Name = "About"
    URL = "/about/"
[tags]
    Name = "Tags"
    URL = "/tags"
```

给localhost站点创建一个文章  

> cd localhost  
> hugo new post/first.md  

```
在localhost目录下执行该命令。文章会在content目录里生成
```

启动hugo server  

> hugo server --theme=blackburn --buildDrafts --watch

生成静态资源  

> hugo --theme=blackburn --baseUrl=https://asdfsx.github.io/


```
注意 baseurl里用了https。因为github现在全站使用https。
如果不指定https，会用http来生成内部资源的链接。
当加载css、js时可能会发生 “Blocked loading mixed active content” 的问题。
导致页面显示异常
```

经过上边一步，静态资源生成，并保存在public目录下，接下来把public下的内容都上传到github上就好了

首先在GitHub 上创建一个repository,名字为: username.github.io  
然后将public下的内容都传到该项目里

> cd public  
> git init  
> git add .  
> git commit -m "first commit"  
> git remote add origin https://github.com/asdfsx/asdfsx.github.io   
> git push -u origin master  

完成以上步骤以后就可以访问https://asdfsx.github.io

