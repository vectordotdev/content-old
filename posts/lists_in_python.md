# Lists in Python
_This article is the first in our Data Structures in Python series._

Lists are containers that can hold multiple pieces of information. Lists are commonly used to hold strings such as a list of names or numbers or the number of attendees for a class. They are an ordered collection of values that can store any data type.

## Why not store everything in an individual variables
It would be a pain to store everything individually like this:

```python
Ramone1 = 'Johnny Ramone'
Ramone2 = 'Dee Dee Ramone'
Ramone3 = 'Joey Ramone'
Ramone4 = 'Tommy Ramone'
```

## Enter lists
Lists are a data structure that allow us to hold multiple values. Lists are are created by placing items inside of `[ ]` . The values in lists are comma separated.

An empty list looks like this:

```python
the_ramones[]
```

Let's take a look, at the example from before as but reformatted as a list:

```python
the_ramones = ['Johnny Ramone', 'Dee Dee Ramone', 'Joey Ramone', 'Tommy Ramone']
```
## How long is my list?
We can see how long the list is by calling `len()` on it:

```python
print(len(the_ramones))
```
For this list we would get back the number 4 since there are four elements of the list above.

## Accessing individual parts of the list.
Slicing in Python allows us to access individual parts of the list to grab a subset. With Python, we start counting at zero so if we ran this code we would return the first element of the list 'Johnny Ramone':

```python
print(the_ramones[0])
```

We can slice the list at an index of one to return 'Dee Dee Ramone'

```python
print(the_ramones[1])
```

To return just the first three elements of the list we  can use the following:

```python
print(the_ramones[0:3])
```
We can also simplify the code to just return everything after the 0 index:

```python
print(the_ramones[0:])
```
## Pop it off
To remove the last item from a list you can use `.pop()`.

We can use `.pop()` on our list from earlier. If we now print the list we can see that we have removed the last item of the lis. In this we case removed Tommy Ramone from `the_ramones`.

```python
the_ramones.pop()
print(the_ramones)
```

## What if I don't want the last list item but another place?
We can specify the index to remove whatever we'd want using `del`.

In this example we have removed Dee Dee Ramone from the list:

```python
del the_ramones[1]
print(the_ramones)
```

## What about removing more than one item?
If you wanted to remove more than one element in a list can also use a range:

```python
del the_ramones[0:2]
print(the_ramones)
```

## Adding one item to a list
`.append()` allows us to add one item to the end of the list. If wanted to Dee Dee back to the list we can use the following code:

```python
the_ramones.append('Dee Dee Ramone')
print(the_ramones)
```

## What if I want to add more than one item to the end of my list.
`.extend()` allows us to add a list of extra list elements to our list. If we wanted to add some other members to `the_ramones` we can do this as so:

```python
the_ramones.extend(['Marky Ramone', 'Richie Ramone', 'Elvis Ramone'])
print(the_ramones)
```

## Changing list item
We can use slicing to change list items as well by setting the value equal to a new value.

In this example we can add C. J. Ramone to the end of our list.

```python
the_ramones[4] = 'C. J. Ramone'
print(the_ramones)
```

## Sorting things out
To sort a list we can use `.sort()`, which works for numerical data or strings:

```python
votes = [5, 4, 6, 3, 9, 1, 2]
votes.sort()
print(votes)

the_ramones.sort()
print(the_ramones)
```

## We can also reverse a list
To reverse the order of the list we can use `.reverse()`.

```python
print(votes.reverse())
print(the_ramones.reverse())
```

## Combining two lists together
We can use the `+` operator to add lists together.

```python
thin_lizzy = ['Eric Bell', 'Eric Wrixon', 'Phil Lynott' 'Brian Downey']

supergroup = the_ramones + thin_lizzy
print(supergroup)
```

## Splitting up a string
You can use .split() to take items from a string and turn it into a list.

```python
fruits = 'Apples, Oranges, Pears, Bananas'
fruit_basket = fruits.split(',')
```

## Looping through a list
The basic syntax of a for loop in Python is as follows:

```
for item in list:
  do something
```

If we wanted to print out each fruit in our `fruit_basket` list we created we can use the following syntax:
```python
for fruit in fruit_basket:
  print(fruit)
```

## Why lists
Lists are one of the four built-in data structures in Python. In future articles, we will be discussing tuples, sets, and dictionaries. You will want to use lists when you need to store an ordered collection of items.

_Jessica Garson is a Adjunct Professor at NYU. She’s also previously the organizer of the Tech Lady Hackathon, and  the organizer of Hack && Tell DC. She is a frequent teacher at DC's Hear Me Code, beginner friendly classes for women by women and has run programming classes at DC area public libraries. Jessica also runs a self-published programming magazine called What's my Function. In 2015, she was named by DC FemTech as one of the most powerful Women in Tech. In 2017, one of her projects nominateher.org got named best web/mobile product of the year by Technical.ly DC._
