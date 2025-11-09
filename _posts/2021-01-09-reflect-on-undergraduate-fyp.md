---
layout: post
title: Looking back at my undergraduate final year project
published: false
---

As I was putting up my new website, I looked back at my final year project as an undergraduate student. I realized I have forgotten most of the technical details and found many programming mistakes that I made in the implementation. It has been an interesting exercise to go retro and reflect a bit on the project So I decided to write a short blog post about it.

## Background
I was interested in machine learning and distributed systems at the time. After consulting with my advisor, I decided on a project that would give me exposure to both. The project was to develop a distributed sparse support vector machine (SVM) algorithm for feature selection. There are a lot of keywords in this sentence. So let's break it down word by word:
- `distributed` - the algorithm distributes the data and the work to a coordinated set of machines
- `SVM` - a classic classification algorithm with nice bounds
- `feature selection` - the process of selecting from a dataset a subset of features that are most important for a problem (think regression or classification)
- `sparse` - select only a small subset of the features (the standard technique is to regularize the l0 or l1 norm of the weight vector)

The gist of the project was to implement a single machine feature selection algorithm called [Feature Generating Machine (FGM)](https://jmlr.csail.mit.edu/papers/volume15/tan14a/tan14a.pdf) in a distributed fashion, on top of the [Husky](http://www.cse.cuhk.edu.hk/proj-h/index.html) system \[[^1]\]. The most challenging parts of the project were to understand the mathematical formulation of the problem and coming up with a decomposition of the data and the work in a distributed setting whilst keeping remote communication costs small. If you are interested in learning more, you can find my report [here](/assets/distributed-feature-selection.pdf).

## Reflection

Looking back at the code, which can be found [here](https://github.com/xwkuang5/husky/blob/fyp/examples/dfgm.cpp). I realized I did not know much about software design and had just put everything in a single file, rendering a 2209-lines-of-code beast. The file was supposed to be composed of several main modules: a dual-coordinate descent SVM solver, a distributed SVM solver, a distributed feature selection solver, and some system boilerplates. These modules are smashed into the same file without proper interfaces defined between them. 

Not to make any excuses but I think that is a problem shared by many research codes that I have seen in my short time as a student researcher. Students are typically swamped with coursework when at the same time they need to validate many research ideas via fast prototyping. Often an idea is tested and abandoned within a few days, and spending the time to think through the software abstraction and interface just does not feel like it is worth the time. I remembered struggling through thousands of lines of heavy math C++/matlab codes with very few comments and swearing that I would do a better job at design and documentation when implementing my project. I guess I failed to keep my promise after all.

## Learnings

Despite forgetting most of the technical details of the project, I think a few key learnings did strike and persist:
1. Understanding the single machine setting to the core is of utmost importance to designing and implementing any distributed algorithms.
2. Decoupling and decomposition are very useful techniques for designing distributed algorithms.

## Footnote

[^1]: Husky is a general distributed computing system developed at CUHK by my advisor and his group. I learned a lot about distributed processing systems during that time when helping out the project as a contributor.

