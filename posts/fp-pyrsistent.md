---
categories:
- markdown
date: '2020-03-22'
description: Use Pyrsistent to create immutable values in Python for better code maintenance.
layout: post
title:  Reading code easily with immutable values(Pyrsistent).
toc: true

---

###### TLDR: 
Use data structures that don't change once created(using Pyrsistent) to make it easy to understand and maintain code.

#### Purpose.
When a data structure(e.g. dict) once created, does not change, it allows us to read code with more confidence.

For e.g. Let's say you have a `customer` variable in your code and you are tracking it's value by reading the code. What do you reason is the value of `customer` below? 
```python
customer = dict(name="Rajiv", age=40)
some_function(customer)
print(customer)
```
In the above code, we can't say. In Python, for most default data structures like `dict`,  it is possible that `some_function` could have changed the value of `customer`. So, we have to dig in and read the code of `some_function` to be fully sure. If the code of `some_function` was below:
```python
def some_function(a):
    a1 = a
    a1['name'] = 'NewRajiv' # Changing the values. blasphemy
    # do something with a1
``` 
then `print(customer)` would display `{'name': 'NewRajiv', 'age': 40}`. 

If you are lucky, `some_function` does not pass it forward to other functions! Or else, you would have to dig in and read those functions too :). Now that would suck. Unless, it is the intention that the `customer` field should be mutated but in most cases, one does not expect it to be so(in other languages,naming conventions are used to indicate if that is the case).  A knowledgeable programmer may make a copy(via the `copy.deepcopy()`) and work on the copy to prevent her code from affecting the client code but I have not been that  knowledgeable programmer :) 

What if we could use a data structure that once created, cannot be changed i.e. it is immutable. Let's check out a library called [pyrsistent](https://github.com/tobgu/pyrsistent) that gives us such data structures. 

```python
from pyrsistent import m # m is like a dictionary

customer1 = m(name='Rajiv', age=40)
customer2 = customer1.set(name='NewRajiv')
print(customer1) # pmap({'age': 40, 'name': 'Rajiv'})
print(customer2) # pmap({'age': 40, 'name': 'NewRajiv'})
```

When we specify a different value('NewRajiv'), a copy is created with that new value and assigned to `customer2`. `customer1` still retains the value it was first assigned. Now, let's go back to our previous code example and modify it a bit for `pyrsistent`

```python
from pyrsistent import m # m is like a dictionary

def some_function(a):
    a1 = a.set('name', 'NewRajiv')
    # do something with a1 

customer = m(name="Rajiv", age=40)
some_function(customer)
print(customer)


```
`print(customer)` would display `{'name': 'Rajiv', 'age': 40}`, the value set in our code. So, we can safely reason about our code and what it's doing without worrying about it changing inside `some_function`. We don't have to even look into `some_function` in this case. Trust me, when you can't run that snippet of code to see what the actual values are, this feature makes life so easy :).


`pyrsistent` also has support for other common data structures(i.e. lists, sets) and much much more. Most of these `pyrsistent` data structures are drop in replacements for their Python counterparts when it comes to accessing the data.

From the pyrsistent docs:
```python
from pyrsistent import v  # like a list

a = v(1, 2, 3)
b = a.append(4)

print(b[1])  # 2
print(b[1:3])  # pvector([2, 3])
print([2 * x for x in b])  # [2, 4, 6, 8]
```
### On Speed and Memory

I simplified(ok, I lied) when I said that `pyrsistent` makes a copy of the data structure. Such a practice would be a waste of memory and time if we copy over every huge data structure. `pyrsistent` mitigates that to a great extent by not just blindly copying data structures and then making the modifications. It tries to be intelligent by `sharing the common parts` between a original data structure and the new modified copy to save on memory and time.

Let's take an example(Credit: Wikipedia: Persistent Data Structures). Ah, you now are exposed to what this is really called.  This concept is called `persistent data structures` or `functional data structures`. 
 
 NOTE: The below example is just to explain the concepts and such a binary search tree is not part of `pyrsistent`.  

Let's say you had a binary search tree(`xs`) which was a persistent data structure:

![alt text](https://upload.wikimedia.org/wikipedia/commons/thumb/9/9c/Purely_functional_tree_before.svg/696px-Purely_functional_tree_before.svg.png "Binary Search Tree")

Now if you added a node `e` to that data structure, e.g. `ys = insertNode(xs, e)` A naive implementation would copy the data structure and then insert `e` at the appropriate location. In a persistent data structure approach, it would be:

![alt text](https://upload.wikimedia.org/wikipedia/commons/thumb/5/56/Purely_functional_tree_after.svg/876px-Purely_functional_tree_after.svg.png "Persistent Binary Search Tree")

Since `e` falls into the right side of the tree(i.e. the tree with `g` as the root), the tree with root `b` is not affected and hence can be reused. You can see a arrow from `d'`  to `b` indicating that.

This reuse saves memory space, and saves time not done copying as it merely uses pointers to refer to the unchanged data.

Note: just because a data structure is being reused does not mean modifications to one can affect another. They are immutable and hence cannot be modified. If you do modify, a new structure is created like above.

Note on Note: `pyrsistent` tries to be as fast as possible and has comparable speeds to the norm for most cases. The complexity of most operations are well described in their docs.
### Nested Transformations

What if we have to update a nested value in a data structure while maintaining immutability. `pyrsistent` has a method `transform` for that. 
How I would normally do it
```python
import copy
m4 = dict(a=1, b=6, c=[1, 2])
# I want to update c[1] to 17
m4_new = copy.deepcopy(m4) 
m4_new['c'][1] = 17
```

From their docs,

```python
from pyrsistent import m  # m is like a dictionary
from pyrsistent import v # m is like a list
m4 = m(a=5, b=6, c=v(1, 2))
m4_new = m4.transform(('c', 1), 17)
print(m4_new) # pmap({'a': 5, 'c': pvector([1, 17]), 'b': 6})

``` 

### Updating dictionaries

One thing I do very often is merging dictionaries. For e.g., I may have to construct my configuration taking the the following sources with the earliest being the highest priority.
 * Environment variables
 * File configuration
 * Default configuration

How I would normally do it.

```python
default_conf = dict(database_url='dev_url', user='postgres', port=5432)
# Imagine file_conf below was extracted from a file
file_conf = dict(user='test_user', port=5433)
# Imagine env_conf below was constructed from environment variables
environment_conf = dict(database_url='test_url')
final_conf = {**default_conf, **file_conf, **environment_conf}

print(final_conf) # {'database_url': 'test_url', 'user': 'test_user', 'port': 5433}

```

That's great for 99% of the cases I would think :). But for the sake of discussion, perhaps if you had HUGE dictionaries(e.g. merging all the data you scrapped illegally from some website ;) ), that would be some duplication of data in memory.
In `pyrsistent`:
```python
from pyrsistent import m
default_conf = m(database_url='dev_url', user='postgres', port=5432)
# Imagine file_conf below was extracted from a file
file_conf = m(user='test_user', port=5433)
# Imagine env_conf below was constructed from environment variables
environment_conf = m(database_url='test_url')
final_conf = default_conf + file_conf + environment_conf

print(final_conf) # pmap({'database_url': 'test_url', 'user': 'test_user', 'port': 5433})
``` 

I hope this is enough to get you started in a better coding experience :). There are many other wonderful features in `pyrsistent` like having the above behaviour for records(`PRrecord`) and clases(`PClass`) and many more advanced features. I'll leave that for another post.

So head out to  [pyrsistent](https://github.com/tobgu/pyrsistent) and check it out. And if you like it, don't forget to star! It's a wonderful piece of engineering whose authors that we should applaud and support.



