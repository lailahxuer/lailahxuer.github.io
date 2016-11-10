---
layout: post
title:  "Markdown Example"
date:   2016-11-10 10:00:00
categories: temp
---

`code span`

    public class Main {
        public static void main(String[] args) {
            System.out.println("this is a markdown code block.");
        }
    }

{% highlight java %}
// highlighting using Liquid tag
public class Main {
	public static void main(String[] args) {
		System.out.println("this is not a markdown syntax.");
	}
}
{% endhighlight %}

Inline link to [Rouge wiki](https://github.com/jneen/rouge/wiki/List-of-supported-languages-and-lexers).

Reference-style link to [Pygments’ Lexers page][pygments_lexer].

[pygments_lexer]:http://pygments.org/docs/lexers/
