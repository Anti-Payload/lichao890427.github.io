---
layout: page
title: About
description: 莫以恶小而不为，莫以善小而为之
keywords: 
comments: true
menu: 关于
permalink: /about/
---

逆向、调试、安卓、IOS、python、机器学习



## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
