+++
title = "Extracting authorship from a git repository"
date = "2023-08-03"
+++

When moving code from one project to another, original authorship and attribution is easily lost in the process.
Fortunately, if you're using [git][git], this information travels with the repository and anyone who had a part in publishing code is identifiable.
In this post, we are concerned with identifying the authors who were involved with the [ENCORE][encore] analysis module in MDAnalysis and giving proper attribution in the new [mdaencore][mdaencore] MDAKit `AUTHOR.md`.

While GitHub *does* show the historical information we want, this takes is much easier accomplished on the command line with a local copy of the repository.
From within the original code base, at the same level as the `encore` analysis module, we can run the [`git log`](https://git-scm.com/docs/git-log) command
```bash
$ git log --pretty="%an <%ae>|%ai" encore/ \
      | sort -k2 -t'|' \
      | awk -F"|" '!_[$1]++'
```
and then output the contents to a file or `less` for easier viewing.
This will show all authors (`%an`) along with their email (`%ae`) and their commit times (`%ai`).
We can separate the author information from the times with a `|`, so we can later [`sort`](https://man7.org/linux/man-pages/man1/sort.1.html) the output by date using the pipe symbol as a delimiter.
This is then passed to [`awk`](https://tldp.org/LDP/abs/html/awk.html) to remove duplicate authors (based on their name and email combination) but maintaining chronological order.
Unfortunately, as is common with git histories, many authors will have multiple email addresses and can show up multiple times in a history.
In this case it's best to just pick the most reliable contact address.

The output of the above command yields a list of authors (with real emails removed for scraping reasons)

```
Matteo Tiberti <XYZ@ABC.COM>|2016-01-26 15:43:49 +0000
Tone Bengtsen <XYZ@ABC.COM>|2016-02-16 13:40:41 +0100
Wouter Boomsma <XYZ@ABC.COM>|2016-02-20 23:05:27 +0100
Wouter Boomsma <XYZ@ABC.COM>|2016-09-11 22:36:33 +0200
Richard Gowers <XYZ@ABC.COM>|2016-10-13 14:40:48 +0100
Oliver Beckstein <XYZ@ABC.COM>|2016-10-27 23:26:26 -0700
Wouter Boomsma <XYZ@ABC.COM>|2016-11-03 09:31:43 +0100
Max Linke <XYZ@ABC.COM>|2016-11-30 18:03:45 +0100
Max Linke <XYZ@ABC.COM>|2016-12-30 22:23:16 +0100
Jonathan Barnoud <XYZ@ABC.COM>|2017-01-12 22:14:39 +0100
richardjgowers <XYZ@ABC.COM>|2018-04-24 13:47:05 -0400
Tyler Reddy <XYZ@ABC.COM>|2018-05-02 15:32:20 -0600
zeman <XYZ@ABC.COM>|2018-06-11 11:28:45 +0200
Matthijs Tadema <XYZ@ABC.COM>|2019-07-16 14:56:55 +0200
Matthijs Tadema <XYZ@ABC.COM>|2019-07-17 14:12:10 +0200
Rocco Meli <XYZ@ABC.COM>|2020-02-15 08:39:12 +0000
shfrz <XYZ@ABC.COM>|2020-03-19 14:56:59 +0530
Irfan Alibay <XYZ@ABC.COM>|2020-06-15 08:13:38 +0100
Philip Loche <XYZ@ABC.COM>|2021-02-04 10:57:15 +0100
Paarth Thadani <XYZ@ABC.COM>|2021-04-07 20:06:14 +0530
Aya Alaa <XYZ@ABC.COM>|2022-03-23 17:06:12 +0200
Estefania Barreto-Ojeda <XYZ@ABC.COM>|2022-07-25 02:28:09 -0600
chrispfae <XYZ@ABC.COM>|2023-02-24 23:16:15 +0100
```

The repeated authors in this case used different email addresses between their commits.
This list of authors can then be used to populate an `AUTHORS.md` file like the one in [mdaencore][mdaencoreauthors].

[git]: https://git-scm.com/
[ENCORE]: https://docs.mdanalysis.org/2.5.0/documentation_pages/analysis/encore.html
[mdaencore]: https://github.com/MDAnalysis/mdaencore
[mdaencoreauthors]: https://github.com/MDAnalysis/mdaencore/blob/main/AUTHORS.md