---
layout: post
title: "A B C of grep"
date: 2012-11-09 00:00:00
preview: a greenhorn's second introduction to the grep program
---

With grep, you can not only return the matching line for your string or regex, but also the context in which that string occurs. This is achieved by the simple use of the flags, -A <N> -B <N> and -C <N>, which stand for "after", "before" and "around", respectively, and <N> signifies the number of lines returned.

As a quick example, I'll print to the terminal my scss codeblock style settings:

    $ grep code -A 10 sass/_typo.scss
    pre {
      margin:1em;
      padding:1em;
      overflow-x:scroll;
      background-color:rgba(232,239,251,1)
    }
    code {
      @include font($base-size*0.8,$base-height*0.8,black,('Anonymous Pro', 'Bitstream Vera Sans', 'Monaco', Courier, mono));
    }

No need to open the file!
