title: Disappeared Trio-Encryption
date: 2015-12-04 01:06:48
tags:
 - crytology
 - encryption
 - Aliyun
---
This was a challenge posted by Aliyun, which is the cloud computing business of Alibaba Group, for the "shopping carnival" on 11/11 every year, which date also known as "Single Day" in China.

Here is the original chanllenge:
{% img /2015/12/04/Disappeared-Trio-Encryption/question.jpg 300  "Question" %}

There were three hints:
1. Colorful anagram, green and red stand on opposite side
2. Ancient encryption technique, invented 300 years ago
3. In programmers' world, 1+1 != 2

<!-- more -->
When looking at the second hint, we can almost certain that that was not any fancy encryption algorithm because the history of computer is only around 60 years. That should give us a lot information, back to that age, the most popular encryption method would be [substitution cipher](https://en.wikipedia.org/wiki/Substitution_cipher), like `simple substitution` or `polygraphic cipher`. What's more, by taking a look at the question, space is also given, which is another import factor that tells us length of each word.

Noted that letter frequency analysis would not necessarily solve this problem because polygraphic cipher like `Vigenère cipher` could mask letter frequency with different key length. However, with modern computing power, and know word length, in theory we should be able to use brute force cracking this by dictionary lookups easily.

But why not use more intelligence instead of computer power? :)

With hint 1, all the green and red letters from questin string are:
```
kearyodd
```

Feels like an anagram. I googled some anagram solver, however, I didn't get any reasonable result. It does look like a lot with `keyboard` with a **b** instead of **d**. This sounds a little wobble, but if we think **b** as a rotated **d** it's good enough, heh.

Alright, so we have our first clue: `keyboard`.

We can try this in two way:
1. keyboard simple substitution cipher, or
2. key for Vigenère cipher

First one is not hard to understand, a keyboard substitution is just keyboard order to alphabetical order: qwert -> abcde etc.
Second one would require some effort, take a look at the following Vigenère cipher:
{% img /2015/12/04/Disappeared-Trio-Encryption/vigenere_cipher.png 300 Vigenère Cipher %}
Say if you want to encrypt "apple" with key "dog", you just need repeat "dog" to same length of input text and lookup each letter from the graph:
```txt Vigenère Cipher
TEXT: 			apple
KEY:  			dogdo
ENCRYPTION:		ddvos
```
It turned out it was just a simple keyboard substitution. Here's the decoded text:

{% blockquote %}
In the room there are four identical basketballs and two identical footballs. Now if you want to put them in one line, how many solutions are there? Tips:please change the form of the number you get.The Programmers!
{% endblockquote %}

Now we are getting really close. We successfully decrypted the first level encryption, which is text encryption. Now it's a basic probability question. Two ways to tackle this:
1. place 4 basketball in a row, then insert footballs between them, with cases two footballs are adjacent and seperated by basketball. C(5,2) + C(5,1) = 15
2. place 6 basketball in a row, pick either two as football. C(6,2) = 15

Second level encryption revealed. Anwser is 15.

It's getting really obvious now. With the tip from decrypted message above, it's asking you change form of the number iin programming world: decimal -> binary.

so: 15 -> 1111.

Awesome, the final anwser is 1111, which is the date for the shopping carnival.


