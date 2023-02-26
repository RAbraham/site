---
categories:
- markdown
date: '2020-04-25'
description: Brython is a Python implementation which runs on the browser 
layout: post
title: Brython. On replacing JavaScript with Python for front-end development
toc: true

---

This article was originally posted at `https://blog.rajivabraham.com/posts/brython`

# Purpose.
This blog article will:
 * give a brief introduction to using [Brython](https://brython.info/), a Python implementation for front-end development on the browser

The entire project is [here](https://github.com/RAbraham/brython-blog)

# Introduction
Jealous of JavaScript programmers, a cabal of Python programmers secretly met to discuss the future of Python in this apocalyptic world. JavaScript was everywhere and eating Python's lunch. With Node.js, JavaScript had invaded Python territory and ended its dominance as everyone's favorite language after Ruby(not very dominant then, is it? Mr. Author). It was time to make a thrust into the heart of JavaScript land: The Browser.

# Don't forget your history(and future)
The cabal were not the only gentlemen concerned with this dilemma. The author of [Transcrypt](https://www.transcrypt.org/) believed in poison and espionage. He decided to write a Python compiler which compiled to JavaScript code. Like good poison, there was no trace of Python. It looked very promising. But the author of this boring post thought he found a bug but later found out it wasn't so. Unwisely, he relegated Transcrypt to a side plot and must keep it there for now.

Others wanted to learn from history. Just immigrate the entire family. At least, that's what [Pyodide](https://hacks.mozilla.org/2019/04/pyodide-bringing-the-scientific-python-stack-to-the-browser/) thought of doing. Their strategy was to create an enclave on the side with a full Python Interpreter which can run Python code. Thus you could run any Python code including most of the data science stack which contains C language bindings (e.g. Numpy, Pandas). 

This looks very promising too. But on initial lazy tests by this author, the initial page load was a bit slow(The real reason was that the author also could not find an easy way to make this work and was happy to not pursue it any further.)

So the cabal decided to do what every cabal is supposed to do i.e. create another Python to JavaScript compiler but this time, compile it to JavaScript when the page loads(unlike Transcrypt which compiles to JavaScript ahead of time). Thus, the fellowship of Brython was formed. One snake to rule them all. 

# Hello World
Let's code up the customary 'Hello World'

The Brython paratroopers(compiler) is here.
```
<script type="text/javascript"
       src="https://cdn.jsdelivr.net/npm/brython@3.8.9/brython.min.js">
</script>
``` 
We activate it on page load
```
<body onload="brython()">
...
</body>
```
Within the `body` tag above, we write the Brython Code:
```
<script type="text/python">
from browser import document

document <= "Hello World"
</script>
```

We just add `Hello World` to the document element. Hmmm. That was easy.

In complete form, it's shown below.

```
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
    <script type="text/javascript"
        src="https://cdn.jsdelivr.net/npm/brython@3.8.8/brython.min.js">
    </script>
</head>

<body onload="brython()">

<script type="text/python">
from browser import document

document <= "Hello World"
</script>
</body>
</html>
```

This will simply print "Hello World" to the page. 

# Calculator
Let's now make a calculator(code courtesy: Brython). The full code is [here](https://github.com/RAbraham/brython-blog/blob/master/calculator.html)


![alt text](https://raw.githubusercontent.com/RAbraham/brython-blog/master/brython-calculator.png "Calculator in Brython")


Yes, you were right. We do need a table. Let's make one.
```
from browser import document, html
calc = html.TABLE()
```
Let's add the first row only. Just the display box(we'll name it `result`) and `C`.
```
calc <= html.TR(html.TH(html.DIV("0", id="result"), colspan=3) +
                html.TD("C"))
 
```
Yes, I'm not very sure of this `<=` syntax either. But hey, for such a lovely library, I'll settle for it too :).

Let's now add the number pad
```
lines = ["789/", "456*", "123-", "0.=+"]
calc <= (html.TR(html.TD(x) for x in line) for line in lines)
``` 
Finally, we add `calc` to the `document`
```
document <= calc
```

Now that's all good. How do we make it work? First, we need to capture a reference to the `result` element to manipulate it when the number pad is pressed.

```
result = document["result"] # direct access to an element by its id
``` 
Next, we need to update the `result` whenever any element in the number pad is clicked. Let's make an event handler. We'll trust the Brython developers that this code works. Notice the manipulation of `result` based on the button you clicked.

```
def action(event):
    """Handles the "click" event on a button of the calculator."""
    # The element the user clicked on is the attribute "target" of the
    # event object
    element = event.target
    # The text printed on the button is the element's "text" attribute
    value = element.text
    if value not in "=C":
        # update the result zone
        if result.text in ["0", "error"]:
            result.text = value
        else:
            result.text = result.text + value
    elif value == "C":
        # reset
        result.text = "0"
    elif value == "=":
        # execute the formula in result zone
        try:
            result.text = eval(result.text)
        except:
            result.text = "error"

``` 
Finally, we associate the event handler above to the `click` event of all buttons.

```
for button in document.select("td"):
    button.bind("click", action)

```
See, how easy it is when someone else writes the code :P. But seriously, Brython is a wonderful work of engineering and perhaps the best display of programmer love for their beloved Python language. Please support the developers, at least with a star on their Github [repo](https://github.com/brython-dev/brython)!


# For the advanced reader
* One can also integrate third party libraries like Vue.js as shown [here](https://brython.info/gallery/test_vue.html).
* A great in depth explanation of the concepts can be found [here](https://anvil.works/blog/python-in-the-browser-talk) 




