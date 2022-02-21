---
layout: post
title: "Style Guide"
sitemap: false
---

"Come with me, keep quiet, an' keep yerself covered with that cloak," said Hagrid. "We won' take Fang, he won' like it. . . "Listen, Hagrid, I can't stay long. . . . I've got to be back up at the castle by one o'clock -" But Hagrid wasn't listening; he was opening the cabin door and striding off into the night.

# Headers
{% highlight markdown %}
# H1
## H2
### H3
#### H4
##### H5
###### H6
{% endhighlight %}

# H1
## H2
### H3
#### H4
##### H5
###### H6

# Text formatting
{% highlight markdown %}
- **Bold**
- _Italics_
- ~~Strikethrough~~
- <ins>Underline</ins>
- <sup>Superscript</sup>
- <sub>Subscript</sub>
- Abbreviation: <abbr title="HyperText Markup Language">HTML</abbr>
- Citation: <cite>&mdash; Bill</cite>
{% endhighlight %}

- **Bold**
- _Italics_
- ~~Strikethrough~~
- <ins>Underline</ins>
- <sup>Superscript</sup>
- <sub>Subscript</sub>
- Abbreviation: <abbr title="HyperText Markup Language">HTML</abbr>
- Citation: <cite>&mdash; Bill</cite>

# Lists
{% highlight markdown %}
1. Ordered list item 1
2. Ordered list item 2
3. Ordered list item 3

* Unordered list item 1
* Unordered list item 2
* Unordered list item 3
{% endhighlight %}

1. Ordered list item 1
2. Ordered list item 2
3. Ordered list item 3

* Unordered list item 1
* Unordered list item 2
* Unordered list item 3

# Links
{% highlight markdown %}
Check out this example project on [GitHub](https://github.com/).
{% endhighlight %}

Check out this example project on [GitHub](https://github.com/).

# Images
{% highlight markdown %}
![Image with caption](https://www.fillmurray.com/g/600/400 "Image with caption")
_This is an image with a caption_
{% endhighlight %}

![Image with caption](https://www.fillmurray.com/g/600/400 "Image with caption")
_This is an image with a caption_

# Code and Syntax Highlighting
Use back-ticks for `inline code`. Multi-line code snippets are supported too through [Pygments](https://github.com/mvdbos/kramdown-with-pygments).

{% highlight js %}
// Sample javascript code
var s = "JavaScript syntax highlighting";
alert(s);
{% endhighlight %}

Adding `linenos` to the Pygments tag enables line numbers.

{% highlight js linenos %}
// Sample javascript code
var s = "JavaScript syntax highlighting";
alert(s);
{% endhighlight %}

# Blockquotes
{% highlight markdown %}
> "Keep quiet, an' keep yerself covered with that cloak," said Hagrid.
{% endhighlight %}

> "Keep quiet, an' keep yerself covered with that cloak," said Hagrid.

# Horizontal Rule & Line Break
{% highlight markdown %}
Use `<hr>` for horizontal rules

<hr>

and `<br>` for line breaks.

<br>
{% endhighlight %}

Use `<hr>` for horizontal rules

<hr>

and `<br>` for line breaks.

<br>

_The end_
