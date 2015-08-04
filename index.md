---
layout: home
---

<div class="index-content blog">
    <div class="section">
        <ul class="artical-cate">
            <li class="on"><a href="/"><span><h2>技术博文</h2></span></a></li>
            <li style="text-align:center"><a href="/opinion"><span><h2>胡言乱语</h2></span></a></li>
            <li style="text-align:right"><a href="/project"><span><h2>往期归档</h2></span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.blog %}
            <li>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
        <div class="introduction">
            <div class="photo"> 
                <img src="/images/lambda.png" width="100%" />
            </div>
            <br/>
            <h3>Lambda | iOS | Unix | C++</h3>
            <!-- 微博 -->
            <a href="http://weibo.com/208234305/" target="_blank" title="微博">
                <i class="fa fa-weibo fa-2x"></i>
            </a>&nbsp
            <!-- twitter -->
            <a href="http://twitter.com/feimengspirit/" target="_blank" title="Twitter">
                <i class="fa fa-twitter fa-2x" class="before"></i>
            </a>&nbsp
            <!-- github -->
            <a href="https://github.com/feimengspirit" target="_blank" title="Github">
                <i class="fa fa-github fa-2x"></i>
            </a>&nbsp
            <!-- 邮件 -->
            <a href="mailto:feimengspirit@gmail.com" title="邮件联系">
                <i class="fa fa-envelope fa-2x"></i>
            </a>
        </div>
    </div>
</div>
