---
title: "Code and Data in a large Machine Learning project"
categories: blog
layout: post
output: html_document
date: "2019-03-18 14:00:00"
---

We did a large machine learning project at work recently. It involved two data scientists, two backend engineers and a data engineer, all working on-and-off on the R code during the project. The project had many interesting and new aspects to me, among them are doing data science in an agilish way, how to keep track of the different model versions and how to deal with directories, data and code on different machines. I planned to do a series of write-ups this summer, describing each of them, but then this happened 

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Let me know if you write this up somewhere and I could summarize and/or link to it. I think it would be good to have an overview of different approaches to the Path Problem.</p>&mdash; Jenny Bryan (@JennyBryan) <a href="https://twitter.com/JennyBryan/status/1101158865897283585?ref_src=twsrc%5Etfw">February 28, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


Compliant as I am, here is already the story on the latter topic.

We knew upfront that the model we were trying to create would take many iterations of improvement before it was production worthy. This implied that we were to create a lot of code and a lot of data files. If not organized properly we could easily drawn in the ocean we were about to create. We had a large server at our disposal that could do the heavy lifting. But, because we sometimes needed all the cores for training for a prolonged period of time we also worked on our local machines.

The server was our principal machine for building the project, because it had a lot more RAM and cores than our local machines and because it was the central place where data was stored (more about that later). The first challenge we had to overcome was how we could work on the server simultaneously on different aspects of the project. Ideally every different exploration and adjustment to the model went in its own git branch so we could use all the best practices of software development, like doing code reviews before merging to master. Working in parallel on the same machine on different aspects made this really hard to do. Then, a DevOps we discussed our challenges with came with the simplest solution ever. Just give every user his own code folder on the same machine, just like every user has a code folder on its own machine. All of a sudden everything worked smoothly, such a simple solution proved to be turning point in the project.

On to the organization of the code itself. From past experiences we knew that reproducibility of results was absolutely vital, both for the quality of the model and for the retention of our mental health. Therefore, we decided that from day one we would use the R package structure to develop the code. This has two major advantages over placing scripts in regular Rstudio projects. First, it will not build if you place R code in the scripts that is not a function or a method. Thereby enforcing writing code that is independent of the state of the user's machine. Second, by using `devtools::load_all()` you have all functionality at your disposal at every step of the analysis. You don't have to load or run certain scripts first, before you can go to work.

But what about doing explorative analysis? You cannot get to much insight by just writing functions. Well, R packages already have a very convenient solution in place in the form of Vignettes. These are normally used to write examples on how the package should be used, for example this [one for dplyr](https://cran.r-project.org/web/packages/dplyr/vignettes/dplyr.html). One way to write a Vignette is in a Rmarkdown file, a format ideal for data exploration because it allows for mixing text with code. We were very strict about the code quality in the R scripts, but the Vignettes are called the *Feyerabend files* (after the epistemologist who claimed that anything goes as valid science). You can mess about here as much as you want as long as the results and insights are subsequently transferred to the R scripts. This allowed for very quick hypothesis testing.

Then finally data. Our principal data source was the company data base. Since the queries to produce an analysis set took a long time to run, we needed to store the results locally. A couple of smaller data sets, such as the IDs of all the cases in the train set, were used so regularly that it was most convenient to have them at our immediate disposal at all times. We included them in the R package as data files. (Just like packages from CRAN have datasets shipped with them). However, most files were too large to hold in memory all the time, and we certainly not wanted to have them in version control. As mentioned, each user had its own code folder on the server, and sometimes we had to work locally as well. While syncing code was easy, using version control, syncing data was hard if everybody kept data inside his own folder, but did not check it in. On the server this was relatively easy to overcome, by using a single data folder outside the code folders. To make sure we could also sync the data locally we made strict arrangements about the creation of data files. Every single one of them had to be produced by a function in the `R` folder of the package. This included all queries to the data base, although this caused some overhead the reproducibility and clarity it gave us made it well worth it. We put the code that produced the data in version control, not the data itself. Data files could then be created on every system independently. Finally, saving and loading the data in a uniform way. How did we deal with that what keeps Jenny awake at night? In the `utils` file of the package functions for writing and reading were created. Before loading or saving, the functions check the name of the system and the name of the user. It would then load from  or save to the folder belonging to the system or the user. Here is an example of the structure for saving as .Rds files.


```r
save_as_rds <- function(file, 
                        filename) {
  
  node <- Sys.info()["nodename"]
  user <- Sys.info()['user']
  
  if (node == "server_node_name") {
    path <- "path/to_the_data/on_the/server"
  } else if (user == "user1") {
    path <- "path/for/user1"
  } else if (user == "user2") {
    path <- "user2/has_data/stored/here"
  }
  
  file_path <- file.path(path, filename)
  saveRDS(file, file_path)
}
```

Every user added their name and local path to these functions. Throughout the code we only used these functions, so we were never bothered with changing directories.

Every project is different, but I think the challenges to developing a complicated model with a team are universal. Hopefully, these practical solutions can help you when you find yourself in such a situation. Of course I am very interested in your best practices. Post a reply or send an email.
