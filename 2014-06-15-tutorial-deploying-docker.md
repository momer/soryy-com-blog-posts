# Tutorial: Docker in Production [Part 1]

## Overview

Docker's recent 1.0 release brought significant attention to the project. A lot 
of the excitement was from people who

  1. Had been tracking either the LXC and/or docker projects for a while
  2. Are close to dev-ops virtualization or deployment systems
  3. Started singing it not knowing what it was, And they'll continue singing
   it forever just because...

To address people in the C group, a lot of the comments around social media
 [0][1] directed people to pages like Docker's [What is Docker?](http://www
 .docker.com/whatisdocker/) overview, or the
  [Linux Containers Homepage](https://linuxcontainers.org/). Those are great
   resources, and you should definitely check it out if you're in the C group, 
   but aren't quite sure what all the excitement is about yet.

Sometimes, though, powerful concepts are best understood when you can see their 
potential in action; so, this will be a small tutorial in which we tackle three 
projects:

1.  Create a basic Docker container, reviewing foundational concepts along the way
2.  Create a more complex Docker container, using advanced networking and volume features
3.  Deploy our complex Docker container onto a live server

[0] [https://news.ycombinator.com/item?id=7868791]([HN] Docker 1.0)
[1] [http://www.reddit.com/r/programming/comments/27pjea/its_here_docker_10/] ([Reddit] It’s Here: Docker 1.0)