<!-- 
.. title: nikola+gitpage搭建博客
.. slug: nikolagitpageda-jian-bo-ke
.. date: 2013/03/18 19:46:37
.. tags: nikola, github
.. link: 
.. description: 
-->

本来博客是在CSDN上的<http://blog.csdn.net/digimon>，但最近CSDN在我这一直无法登录，已经在Evernote上积压了好几篇博客，搞烦了，决定自己搭个博客。  
由于目前是没有收入来源的屌丝，没钱去买个域名、弄个VPS，只能想免费的招了。WordPress这货屏蔽GAE的UA，导致没法申请博客，最后想到了gitpage。虽然gitpage也有随时被墙的风险，但是我的博客主要记录技术内容，作为程序猿翻个伟大的防火墙还是没问题的吧;-)  

gitpage只支持静态页面，网上搜了一下，生成静态页面的工具还真不少，好像专门为此设计的一样，套餐呐有木有。  
Python的就有不少: [What's the best available static blog/website generator in Python?](http://www.quora.com/Whats-the-best-available-static-blog-website-generator-in-Python)  
还有更多别的语言支持的工具: [32 Static Website Generators For Your Site, Blog Or Wiki](https://iwantmyname.com/blog/2011/02/list-static-website-generators.html)  

最终选择了nikola，也是因为Quora上那篇文章的推荐。  
nikola官网：<http://nikola.ralsina.com.ar/>  

详细安装和配置的教程就不赘述了，可以参考这两篇文章：  
* [The Nikola Handbook](http://nikola.ralsina.com.ar/handbook.html)（官网教程）  
* [用 nikola 写静态博客](http://rca.is-programmer.com/2013/2/5/using-nikola-write-static-blog.37513.html)  

下面说一下安装和配置nikola时要注意的问题。  
1. 官网上的[安装指导](http://nikola.ralsina.com.ar/handbook.html#installing-nikola)中，如果你是Ubuntu用户的话，[Mako](http://makotemplates.org/)一定要用pip安装或自己去下，Ubuntu官方源上的版本太旧，运行会出问题。  
2. 喜欢用markdown的童鞋除了更改nikola生成的站点根目录下的从conf.py之外，新建博文的时候要注意加上`-f markdown`参数：`nikola new_post -f markdown`  
3. 喜欢用markdown的童鞋还得注意一下(==!)，官方0.5.4有个bug。在`/usr/local/lib/python2.7/dist-packages/nikola/plugins/compile_markdown/__init__.py`文件中，在73~81行的字符串前加个字母u。  

        with codecs.open(path, "wb+", "utf8") as fd:
            if onefile:
                fd.write('<!-- \n')
                fd.write('.. title: {0}\n'.format(title))
                fd.write('.. slug: {0}\n'.format(slug))
                fd.write('.. date: {0}\n'.format(date))
                fd.write('.. tags: {0}\n'.format(tags))
                fd.write('.. link: \n')
                fd.write('.. description: \n')
                fd.write('-->\n\n')
            fd.write("\nWrite your post here.")  
		
改成  

	    with codecs.open(path, "wb+", "utf8") as fd:
            if onefile:
                fd.write(u'<!-- \n')
                fd.write(u'.. title: {0}\n'.format(title))
                fd.write(u'.. slug: {0}\n'.format(slug))
                fd.write(u'.. date: {0}\n'.format(date))
                fd.write(u'.. tags: {0}\n'.format(tags))
                fd.write(u'.. link: \n')
                fd.write(u'.. description: \n')
                fd.write(u'-->\n\n')
            fd.write(u"\nWrite your post here.")

因为不加u就是str对象，写入的是utf-8编码的文件，当title等参数里出现非英文字母的时候会出先解码异常。（马上提交个pull request去）  
4. 想要快速将页面发布至github，请参考[Deploying Nikola to Github Pages](http://robertfw.github.com/posts/deploying-nikola-to-github-pages.html)  
