---
title: hexo博文包含图片的坑
top: false
cover: false
toc: true
mathjax: true
date: 2019-12-29 10:52:53
password:
summary:
tags:
categories: tool
---

<div align="middle"><iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=414414&auto=0&height=66"></iframe></div>

# hexo博文包含图片的坑

### 网上有很多关于这个的教程,主要的总结如下

- ①修改博客目录下的_config_yml的post_asset_folder为true

```yaml
post_asset_folder: true
```

- ②安装hexo-asset-image插件

```shell
npm install hexo-asset-image --save
```

- ③hexo new  file_name 时会在source/_post/下生成file_name的文件夹,将需要使用的图片放置在里面,然后使用相对路径引入

```html
![用于图片加载失败时显示的内容](file_name/image_name)
```

- 如此博客中的图片最后会和.md文件一起生成到public\2019\12\27\file_name中,这样在hexe g 时,可以看到命令窗口会打印修改后的路径,如下

```html
Start processing
update link as:-->/2019/12/27/first/1577523021175.png
update link as:-->/2019/12/27/first/1577523021175.png
```

### 我遇到的问题

```html
Start processing
update link as:-->.io//2019/12/27/first/1577523021175.png
update link as:-->.io//2019/12/27/first/1577523021175.png
```

- 经过一番搜寻,发现hexo-asset-image会将图片的地址修改,具体的源码信息可见\node_modules\hexo-asset-image\index.js,打开后内容如下:

```javascript
'use strict';
var cheerio = require('cheerio');
function getPosition(str, m, i) {
  return str.split(m, i).join(m).length;
}

hexo.extend.filter.register('after_post_render', function(data){
  var config = hexo.config;
  if(config.post_asset_folder){
    var link = data.permalink;
    var beginPos = getPosition(link, '/', 3) + 1;
    var appendLink = '';
    // In hexo 3.1.1, the permalink of "about" page is like ".../about/index.html".
    // if not with index.html endpos = link.lastIndexOf('.') + 1 support hexo-abbrlink
    if(/.*\/index\.html$/.test(link)) {
      // when permalink is end with index.html, for example 2019/02/20/xxtitle/index.html
      // image in xxtitle/ will go to xxtitle/index/
      appendLink = 'index/';
      var endPos = link.lastIndexOf('/');
    }
    else {
      var endPos = link.lastIndexOf('.') ;
    }
    link = link.substring(beginPos, endPos) + '/' + appendLink;

    var toprocess = ['excerpt', 'more', 'content'];
    for(var i = 0; i < toprocess.length; i++){
      var key = toprocess[i];

      var $ = cheerio.load(data[key], {
        ignoreWhitespace: false,
        xmlMode: false,
        lowerCaseTags: false,
        decodeEntities: false
      });
      $('img').each(function(){
        if ($(this).attr('src')){
          // For windows style path, we replace '\' to '/'.
          var src = $(this).attr('src').replace('\\', '/');
          if(!(/http[s]*.*|\/\/.*/.test(src)
            || /^\s+\//.test(src)
            || /^\s*\/uploads|images\//.test(src))) {
            // For "about" page, the first part of "src" can't be removed.
            // In addition, to support multi-level local directory.
            var linkArray = link.split('/').filter(function(elem){
              return elem != '';
            });
            var srcArray = src.split('/').filter(function(elem){
              return elem != '' && elem != '.';
            });
            if(srcArray.length > 1)
            srcArray.shift();
            src = srcArray.join('/');

            $(this).attr('src', config.root + link + src);
            console.info&&console.info("update link as:-->"+config.root + link + src);
          }
        }else{
          console.info&&console.info("no src attr, skipped...");
          console.info&&console.info($(this));
        }
      });
      data[key] = $.html();
    }
  }
});

```

- 通过查看源码发现里面有对生成博客图片的地址修改:

```javascript
link = link.substring(beginPos, endPos) + '/' + appendLink;
```

- 通过排查发现图片的路径的endPos为:

```javascript
var endPos = link.lastIndexOf('.') ;
```

- 我打印data.permalink得到

```html
http://tigerLuHai.github.io/2019/12/27/first/
```

如此在截取字符串的时候就会多出四个字符  **.io/**

最后发现这段代码的作用就是要将data.permalink中路径的https://tigerLuhai.gituhub.io/去掉,因为在后面部署到github时使用相对路径访问会重新加上这个前缀,如果这里有就会重复,导致地址为https://tigerLuhai.gituhub.io/http://tigerLuHai.github.io/2019/12/27/first/的现象.

明白了需求就可以修改代码为

```javascript
var endPos = link.lastIndexOf('/') ;
```

这样就可以正常部署了.