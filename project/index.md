---
layout: home
---

<div class="index-content project">
    <div class="section">
        <ul class="artical-cate">
            <li><a href="/"><span><h2>技术博文</h2></span></a></li>
            <li style="text-align:center"><a href="/opinion"><span><h2>胡言乱语</h2></span></a></li>
            <li class="on" style="text-align:right"><a href="/project"><span><h2>其他</h2></span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.project %}
            <li>
                <h2>
                    <a href="{{ post.url }}">{{ post.title }}</a>
                </h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
        <div class="introduction">
            <h2 style="font-size:30px" = "">Why so serious?</h2>
            <h4>正如树叶荣枯，鼎膨代谢，终随秋风凋落，人生风景亦如斯。</h4>
            <!-- 微博 -->
            <a href="http://weibo.com/208234305/" target="_blank">
                <i class="fa fa-weibo fa-2x"></i>
            </a>
            <!-- twitter -->
            <a href="http://twitter.com/feimengspirit/" target="_blank">
                <i class="fa fa-twitter fa-2x" class="before"></i>
            </a>
            <!-- github -->
            <a href="https://github.com/feimengspirit" target="_blank">
                <i class="fa fa-github fa-2x"></i>
            </a>
            <!-- 邮件 -->
            <a href="mailto:feimengspirit@gmail.com" title="邮件联系">
                <i class="fa fa-envelope fa-2x"></i>
            </a>
            </div>
        </div>
    </div>
</div>
