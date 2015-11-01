---
layout: post
title: Custom error pages with Spring Boot
categories: articles
tags: [spring-boot]
comments: true
---

If you are like me ,and loathe XML configurations with a passion , and have been using Spring MVC for a while or deciding which framework to use for your next project , then you are in for a treat. Spring Boot is a great web framework based on the philosophy of convention over configuration , backed by the goodness of the entire Spring ecosystem ,and does not mandate a single line of XML. How about that ! 

One of the regular requirement for any website / web app is to built custom error pages. These pages can be plain white pages that only show the error message with its Http error code (which is what are going to do ) or some fancy page displaying the error message.


## Heading

~~~
# Heading 1

## Heading 2

### Heading 3

#### Heading 4
~~~



## Body text

**This is strong**.

This is figure

![Elaphurus davidianus](https://i.imgur.com/Mdc4szJl.jpg "Père David's deer")

*This is emphasized*.

 53 = 125. Water is H<sub>2</sub>O. 

The New York Times <cite>(That’s a citation)</cite>. 

<u>Underline</u>. 


<del>HTML and CSS are our tools</del>. 

## Blockquotes

> Justification:
> This species is listed as Extinct in the Wild, as all populations are still under captive management. The captive population in China has increased in recent years, and the possibility remains that free-ranging populations can be established some time in the near future. When that happens, its Red List status will need to be reassessed. 

> I follow up the quest. Despite of day and night and death and hell.
> <center> -- Idylls of the King, Tennyson </center>



## List Types

### Ordered Lists

1. Item one
   1. sub item one
   2. sub item two
   3. sub item three
2. Item two

### Unordered Lists

* Item one
* Item two
* Item three

## Tables

| Header1 | Header2 | Header3 |
|:--------|:-------:|--------:|
| cell1   | cell2   | cell3   |
| cell4   | cell5   | cell6   |
| cell7   | cell8   | cell9   |


| Kingdom | Phylum  | Class | Order | Family |
|:------:|:------:|:------:|:------:|:------:| 
|ANIMALIA|CHORDATA|MAMMALIA|CETARTIODACTYLA|CERVIDAE|


## Code Snippets

Syntax highlighting via Pygments

{% highlight css %}
#container {
  float: left;
  margin: 0 -240px 0 0;
  width: 100%;
}
{% endhighlight %}


#### highlight with line number.

{% highlight ruby linenos  %}
def foo
  puts 'foo'
end
{% endhighlight %}


## $$\LaTeX$$ 

you can use latex with <q>double $$ </q>

$$e^{i\pi}+1=0$$


## \<q\> tag

here is a \<q\> q tag \</q\>


here is a <q> q tag </q>
