---
layout: post
title: "吐槽一下这个框架文章目录导航栏的实现和愚蠢的我"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - 杂
---


今天发现，[这个](https://github.com/Huxpro/huxpro.github.io)博客框架，也就是你现在看的这个，会把 h1..h6 的标题都生成目录导航栏，而我希望只生成 h1..h4 的目录。很简单的需求，只要我看懂作者怎么实现的就能改了。

检查了一下网页里目录对应的 div 标签都带着`.*catalog.*`属性，然后到项目里`js/`文件夹里查。只在`/js/hux-blog.js`里找到这一个函数与`catalog`有关。虽然我不太熟JS但是也看得出来这是在浏览文章时 highlight 对应章节标题的。~~（因为有注释）~~

    function() {
        var currentTop = $(window).scrollTop(),
            $catalog = $('.side-catalog');

        //check if user is scrolling up by mouse or keyborad
        if (currentTop < this.previousTop) {
            //if scrolling up...
            if (currentTop > 0 && $('.navbar-custom').hasClass('is-fixed')) {
                $('.navbar-custom').addClass('is-visible');
            } else {
                $('.navbar-custom').removeClass('is-visible is-fixed');
            }
        } else {
            //if scrolling down...
            $('.navbar-custom').removeClass('is-visible');
            if (currentTop > headerHeight && !$('.navbar-custom').hasClass('is-fixed')) $('.navbar-custom').addClass('is-fixed');
        }
        this.previousTop = currentTop;

        //adjust the appearance of side-catalog
        $catalog.show()
        if (currentTop > (bannerHeight + 41)) {
            $catalog.addClass('fixed')
        } else {
            $catalog.removeClass('fixed')
        }
    });

查了半天，能 Google 出来的除了 Jekyll 官方的教程和文档之外，大部分有相关内容的博客都是大同小异，即通过 kramdown 给标题生成Id，然后就能通过 Id 获取元素并进行修改，大部分甚至还没官网说得清楚。可是我没有找到修改元素和添加内容的代码。（Toc, Table of contents，有些博文老用这个缩写，刚看到的时候我是一脸懵逼）

> kramdown supports the automatic generation of the table of contents of all headers that have an ID set. 

根据[kramdown文档](https://kramdown.gettalong.org/converter/html.html)，默认的`toc_levels`是1..6，而`_config.yml`中并没有设置这一属性，应该使用的是默认值。

        markdown: kramdown
        kramdown:
        input: GFM     
        syntax_highlighter_opts:
            span:
            line_numbers: false
            block:
            line_numbers: true
            start_line: 1

好嘛，那我就加上`toc_levels: 1..4`应该就行了吧？

但是事实上并没什么用，该跑出来的 h6 还是跑出来了。

我只好去原作者 github 的 issue 找找看有没有提出过类似需求或者问题的人。很遗憾没有。跟目录导航栏相关的 issue 不多，十条左右，其中一个是希望能够对每个级别标题在目录导航栏里能够区分显示的，那个改 css 样式完成了，跟我没什么关系。

大致看了一遍，除了找到作者最初提出要加入目录导航栏功能的 issue 之外没什么收获。在那个 issue 里，作者提到可能会在1.6版本里实现这个功能。

> @BruceZhaoR @blue20080 Hope you guys love it!
Check React-vs-Angular2 or ios9-safari-web to get the exciting preview! This new feature would be released in v1.6 few days later, hopefully.  
~~顺便吐槽一句你们这个issue里的明明都是国人，聊着聊着变英语了怎么回事~~

好嘛，这是逼我找那个版本的代码来看嘛，来吧。

![release history](/img/in-post/post-catalog/img1-release-log.jpg)

诶？v1.6 被吃了吗？点开 v1.7 几个版本好像真的没有关于 catalog 的修改。

本来这时候我差不多想放弃这个改动的打算了，多大点事干要跟自己过不去嘛。但是我又想起来作者这条评论发布时间是2016.5.20，后面几个关于 catalog 的 issue 差不多是从 2016.10 往后出现的，也就是这段时间里作者初步加上了这个功能。还是看一看这段时间的提交记录吧

![emoji](/img/in-post/post-catalog/img3.jpg)

但是看看我找到了什莫！作者又把 v1.6 吐出来了！

![commits](/img/in-post/post-catalog/img2-commits.jpg)

也就改动五个文件而已嘛……等等，这都还没往下拉呢，看这是什莫！修改记录第一条就是这个！居然是在`footer.html`里面的！`a = P.find('h1,h2,h3,h4,h5,h6');`这一句把选定了 h1 到 h6 的 header，我找到项目里对应的位置，把不想要的部分去掉，再在本地运行看一下，果然是我想要的效果。

    <!-- create the directory and set the animatescroll. failed to adjust the scroll event in hux-blog.js so the scroll animation looks odd. -->
    <script type="text/javascript">
        function createCatalog (selector) {
            var P = $('div.post-container'),a,n,t,l,i,c;
            a = P.find('h1,h2,h3,h4,h5,h6');
            a.each(function () {
                n = $(this).prop('tagName').toLowerCase();
                i = "#"+$(this).prop('id');
                t = $(this).text();
                c = $('<a href="'+i+'" class="d_anchor" rel="nofollow">'+t+'</a>');
                c = $('<a href="'+i+'" rel="nofollow">'+t+'</a>');
                l = $('<li class="'+n+'_nav"></li>').append(c);
                $(selector).append(l);
            });
            return true;    
        }


好了，终于折腾完了。什么嘛，居然放在这种地方。我记得我明明 ctrl+F 在几个 html 文件里都搜过一次当时没有结果呀。

话说回来，git 不是能够包含查找特定内容的提交记录吗。应该是`git log -Scatalog`很快就能查到我想要的记录了，或者一早把几个文件夹的文本内容搜索一遍，也不用在github和各种博客中挣扎了(vscode，Ctrl+Shift+F在文件夹中搜索)。

还是自己蠢，git 不熟，命令行操作也不熟，前端也不会（学不会不想学），但是 b 事还多……自己的锅自己背。

The End

------

