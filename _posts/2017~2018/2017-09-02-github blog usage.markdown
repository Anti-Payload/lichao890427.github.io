---
layout: post
title: Github博客用法
categories: Miscellaneous
description: Github博客用法
keywords: 
---

# Jekyll github博客用法

## Jekyll语言模板
Jekyll 使用 Liquid模板语言，支持所有标准的 Liquid 标签 和 过滤器 。 Jekyll 甚至增加了几个过滤器和标签，方便使用。

输出日期`site.time | date_to_xmlschema` => {{ site.time | date_to_xmlschema }}

CGI转码`"foo,bar;baz?" | cgi_escape` => {{ "foo,bar;baz?" | cgi_escape }}

URI转码`"'foo, bar \\baz?'" | uri_escape` => {{ "'foo, bar \\baz?'" | uri_escape }}

统计字数`page.content | number_of_words` => {{ page.content | number_of_words }}

## 高亮语法

```Jekyll
{\% highlight ruby linenos \%}
def foo
  puts 'foo'
end
{\% endhighlight \%}
```

{% highlight ruby linenos %}
def foo
  puts 'foo'
end
{% endhighlight %}

## 数学公式

???.github.io\_includes\header.html加入

```Javascript
	<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
		MathJax.Hub.Config({
			tex2jax: {
				displayMath: [['$','$'],["\\(","\\)"]],
				inlineMath: [['\\[','\\]'], ['$$','$$']],
			},
		});
	</script> 
```

`$\sqrt{x^{2}+\sqrt{y}}$` => $\sqrt{x^{2}+\sqrt{y}}$

