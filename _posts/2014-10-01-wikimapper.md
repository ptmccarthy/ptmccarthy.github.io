---
layout: post
title: WikiMapper 1.0
---

WikiMapper is available for free in the [Google Chrome Store](https://chrome.google.com/webstore/detail/wikimapper/feiheebgoilmbkaddngcoocjbogfchlb?hl=en)!

For quite some time I've thought it would be cool to be able to see how I end up reading about bizarre and seemingly random things after spending some time on Wikipedia. If you're like me, you probably know what I mean. You start out by reading about a pertinent, sensible topic; something that you're working on at work or as a hobby, or something related to current events or politics. But as you read that first page, you start clicking on links to other articles, saving them in new tabs for reading after you (maybe) finish what you started with. As the minutes go by and new tabs lead to even more new tabs, you find yourself reading about [Agriculture in Uzbekistan](http://en.wikipedia.org/wiki/Agriculture_in_Uzbekistan) and [Customs and Traditions of the Royal Navy](http://en.wikipedia.org/wiki/Customs_and_traditions_of_the_Royal_Navy). And you wonder, "how did I end up here?"

I'm obviously not the first person to have had this thought, as beautifully depicted in this classic XKCD comic:

[![XKCD 214](/public/img/the_problem_with_wikipedia.png)](http://xkcd.com/214)

Earlier this year I started hacking together a little Google Chrome extension, dubbed *WikiMapper*, to answer exactly that question. What started out quick and dirty slowly evolved into a more presentable, more functional extension that this past week I declared to be the "1.0" version.

This was my first forray into the [Chrome API](https://developer.chrome.com/extensions/getstarted) and also my first introduction to the data visualization framework [d3.js](http://d3js.org/). They're topics deserving of their own blog posts so for now I will just say that I have enjoyed working with both of them.

WikiMapper is open-source and is hosted on [GitHub](https://github.com/ptmccarthy/wikimapper).
