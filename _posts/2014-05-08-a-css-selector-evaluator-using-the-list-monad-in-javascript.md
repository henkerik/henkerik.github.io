---
layout: post
title: "Building a CSS selector evaluator using the list monad in Javascript"
description: ""
category: 
tags: [monad,list,javascript,CSS]
---
{% include JB/setup %}


In this post I will explain how to write a monadic CSS selector evaluator. The CSS specification defines selectors, which indicate to which DOM elements a given list of style declarations needs to be applied. For example, the selector:

{% highlight css %}
    #test > li
{% endhighlight %}

selects the two li elements in the following document:

{% highlight html %}
<body id="body">
  <p>P1</p>
  <ul id="test">
    <li>A</li>
    <li>B</li>
   </ul>
   <p>P2</p>
</body>
{% endhighlight %}

How do we write a function which takes as an input a CSS selector and returns a list of the selected DOM elements? Let's start by writing a function which selects the child elements of a given element based on a predicate function. I will give an implementation Javascript, but I will write a type signature for each function using Scala notation. 

{% highlight javascript %}
// selectByPredicate: (Element => Bool) => Element => List[Element]
function selectByPredicate (f) {
  return function (e) {
    var result = new Array();
    for (var i = 0; i < e.children.length; i++)
      if (f(e.children[i])) result.push(e.children[i]);
    return result;
  }
}
{% endhighlight %}

By using this general select function we are able to write specialized select functions. We may for example write a function which selects child elements which have a specific id or a specific tagname:

{% highlight javascript %}
// selectById: String => Element => List[Element]
function selectById (id) {
  return selectByPredicate(function (e) { 
    return e.id == id; 
  });
}

// selectByTagName: String => Element => List[Element]
function selectByTagName (name) {
  return selectByPredicate(function (e) { 
    return e.tagName == name.toUpperCase(); 
  });
} 

function selectByClassName (name) {
  return selectByPredicate(function (e) { 
    return e.class == name.toUpperCase(); 
  });
}
{% endhighlight %}

Now, we are able to translate the individual parts of the CSS selector to 
a function by partially applying our select functions:

CSS Selector Part | Partial Applied Selector Function | Remaining Type
--- | --- | ---
`#test` | selectById('test')       | Element => List[Element]
`li`    | selectByTagName('li')    | Element => List[Element]
`.foo`  | selectByClassName('foo') | Element => List[Element]

Our next job is finding a method to chain these partially applied selector functions together. Notice how the remaining type of our partially applied functions line up. However, one quickly sees that simply using function composition is not going to work here: the result of a select function is a list of element while the input of a select function is just a single element. 

This looks remarkabily similar to the problem we faced while composing functions which returned an Option. Today, we apply the same trick. In Javascript we may implement flatMap on lists as following:

{% highlight javascript %}
// flatMap[A,B]: List[A] => (A => List[B]) => List[B]
Array.prototype.flatMap = function (f) {
  return this.map(f).flatten(); 
}

// flatten[A]: List[List[A]] => List[A]
Array.prototype.flatten = function () {
  return this.reduce(function (left, right) {
    return left.concat(right)
  }, new Array());
}
{% endhighlight %}

Now, we are able to compose the individual select function together and write one function which selects the elements given by the CSS selector `#test > li`:

{% highlight javascript %}
// xs contains the result of the CSS selector: #test > li
var body = document.getElementById('body');
var xs = [body].flatMap(
           selectById('test')
         ).flatMap(
           selectByTagName('li')
         );
{% endhighlight %}

This may still look a bit difficult to understand, but imagine if Javascript supported for-comphrensions. In that case we would have been able to write:

{% highlight javascript %}
// xs contains the result of the CSS selector: #test > li
var body = document.getElementById('body');
var xs = for {
  e1 <- selectById ('test')(body)
  e2 <- selectByTagName ('li')(e1)
} yield e2
{% endhighlight %}

Looks quite elegant right? For those interested, a working example is available here.