+++
date = "2016-08-07T12:00:00"
draft = false
tags = ["star wars", "social networks" , "R", "rstats"]
title = "Analysing the social Star Wars network in The Attack of the Clones with R"
math = true
summary = """
"""

+++


This is a free adaptation of two (very) clever analyses made by others:

* [The Star Wars Social Network by Evelina Gabasov](http://evelinag.com/blog/2015/12-15-star-wars-social-network) in which program F# was mostly used to analyse the Star wars social networks 

* [Analyzing networks of characters in 'Love Actually' by David Robinson](http://varianceexplained.org/r/love-actually-network/) in which R was used to analyse the links between the characters of the movie Love Actually.

The aim here is to try and reproduce Evelina's analysis using R only, using David's contribution plus several tweaks I found here and there on the internet. The R code and data are available on my [GitHub page](https://github.com/oliviergimenez/starwars_network).

_Disclaimer_: The original blog posts are awesome and full of relevant details, check them out! My objective here was to teach myself how to manipulate data using trendy R packages and do some network analyses. Some comments below have been copied and pasted from these blogs, the credits entirely go to the authors Evelina and David. Last but not least, my code comes with mistakes probably. 

# Read and format data

First, read in data. I found the movie script in doc format [here](theforce.net/timetales/ep2se.doc), which I converted in txt format for convenience. Then, apply various treatments to have the data ready for analysis. I use the old school way for modifying the original dataframe. [Piping](https://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf) would have made the code more readable, but I do not feel confident with this approach yet.


```r
# load convenient packages
library(dplyr)
library(stringr)
library(tidyr)

# read file line by line 
raw <- readLines("attack-of-the-clones.txt")

# create data frame
lines <- data_frame(raw = raw) 

# get rid of leading and trailing white spaces
# http://stackoverflow.com/questions/2261079/how-to-trim-leading-and-trailing-whitespace-in-r
trim <- function (x) gsub("^\\s+|\\s+$", "", x)
lines <- mutate(lines,raw=trim(raw))

# get rid of the empty lines
lines2 <- filter(lines, raw != "")

# detect scenes: begin by EXT. or INT.
lines3 <-  mutate(lines2, is_scene = str_detect(raw, "T."),scene = cumsum(is_scene)) 

# drop lines that start with EXT. or INT.
lines4 <- filter(lines3,!is_scene)

# distinguish characters from what they say
lines5 <- separate(lines4, raw, c("speaker", "dialogue"), sep = ":", fill = "left",extra='drop')

# read in aliases (from Evelina's post)
aliases <- read.table('aliases.csv',sep=',',header=T,colClasses = "character")
aliases$Alias
```

```
##  [1] "BEN"           "SEE-THREEPIO"  "THREEPIO"      "ARTOO-DETOO"  
##  [5] "ARTOO"         "PALPATINE"     "DARTH SIDIOUS" "BAIL"         
##  [9] "MACE"          "WINDU"         "MACE-WINDU"    "NUTE"         
## [13] "AUNT BERU"     "DOOKU"         "BOBA"          "JANGO"        
## [17] "PANAKA"        "NUTE"          "KI-ADI"        "BIBBLE"       
## [21] "BIB"           "CHEWIE"        "VADER"
```

```r
aliases$Name
```

```
##  [1] "OBI-WAN"        "C-3PO"          "C-3PO"          "R2-D2"         
##  [5] "R2-D2"          "EMPEROR"        "EMPEROR"        "BAIL ORGANA"   
##  [9] "MACE WINDU"     "MACE WINDU"     "MACE WINDU"     "NUTE GUNRAY"   
## [13] "BERU"           "COUNT DOOKU"    "BOBA FETT"      "JANGO FETT"    
## [17] "CAPTAIN PANAKA" "NUTE GUNRAY"    "KI-ADI-MUNDI"   "SIO BIBBLE"    
## [21] "BIB FORTUNA"    "CHEWBACCA"      "DARTH VADER"
```

```r
# assign unique name to characters
# http://stackoverflow.com/questions/28593265/is-there-a-function-like-switch-which-works-inside-of-dplyrmutate
multipleReplace <- function(x, what, by) {
  stopifnot(length(what)==length(by))               
  ind <- match(x, what)
  ifelse(is.na(ind),x,by[ind])
}
lines6 <- mutate(lines5,speaker=multipleReplace(speaker,what=aliases$Alias,by=aliases$Name))

# read in actual names (from Evelina's post)
actual.names <- read.csv('characters.csv',header=F,colClasses = "character")
actual.names <- c(as.matrix(actual.names))
# filter out non-characters
lines7 <- filter(lines6,speaker %in% actual.names)

# group by scene
lines8 <- group_by(lines7, scene, line = cumsum(!is.na(speaker))) 

lines9 <- summarize(lines8, speaker = speaker[1], dialogue = str_c(dialogue, collapse = " "))

# Count the lines-per-scene-per-character
# Turn the result into a binary speaker-by-scene matrix
by_speaker_scene <- count(lines9, scene, speaker)
by_speaker_scene
```

```
## # A tibble: 447 x 3
## # Groups:   scene [321]
##    scene    speaker     n
##    <int>      <chr> <int>
##  1    26      PADME     1
##  2    27      PADME     1
##  3    29      PADME     1
##  4    48      PADME     1
##  5    50      PADME     2
##  6    66 MACE WINDU     1
##  7    67 MACE WINDU     1
##  8    69       YODA     1
##  9    70 MACE WINDU     1
## 10    74       YODA     1
## # ... with 437 more rows
```

```r
library(reshape2)
speaker_scene_matrix <-acast(by_speaker_scene , speaker ~ scene, fun.aggregate = length)
dim(speaker_scene_matrix)
```

```
## [1]  19 321
```

# Analyses

## Hierarchical clustering


```r
norm <- speaker_scene_matrix / rowSums(speaker_scene_matrix)
h <- hclust(dist(norm, method = "manhattan"))
plot(h)
```

![](/img/starwars_network_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

## Timeline

Use tree to give an ordering that puts similar characters close together

```r
ordering <- h$labels[h$order]
ordering
```

```
##  [1] "MACE WINDU"  "YODA"        "SHMI"        "QUI-GON"     "PLO KOON"   
##  [6] "LAMA SU"     "OBI-WAN"     "BAIL ORGANA" "JAR JAR"     "POGGLE"     
## [11] "ANAKIN"      "PADME"       "CLIEGG"      "BERU"        "OWEN"       
## [16] "SIO BIBBLE"  "RUWEE"       "JOBAL"       "SOLA"
```

This ordering can be used to make other graphs more informative. For instance, we can visualize a timeline of all scenes:

```r
scenes <-  filter(by_speaker_scene, n() > 1) # scenes with > 1 character
scenes2 <- ungroup(scenes)
scenes3 <- mutate(scenes2, scene = as.numeric(factor(scene)),
           character = factor(speaker, levels = ordering))
library(ggplot2)
ggplot(scenes3, aes(scene, character)) +
    geom_point() +
    geom_path(aes(group = scene))
```

![](/img/starwars_network_files/figure-html/unnamed-chunk-4-1.png)<!-- -->

Create a cooccurence matrix (see [here](http://stackoverflow.com/questions/13281303/creating-co-occurrence-matrix)) containing how many times two characters share scenes

```r
cooccur <- speaker_scene_matrix %*% t(speaker_scene_matrix)
heatmap(cooccur)
```

![](/img/starwars_network_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

## Social network analyses

### Graphical representation of the network

Here the nodes represent characters in the movies. The characters are connected by a link if they both speak in the same scene. And the more the characters speak together, the thicker the link between them.  
 

```r
library(igraph)
g <- graph.adjacency(cooccur, weighted = TRUE, mode = "undirected", diag = FALSE)
plot(g, edge.width = E(g)$weight)
```

![](/img/starwars_network_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

Compute standard network features, degree and betweeness.


```r
degree(g)
```

```
##      ANAKIN BAIL ORGANA        BERU      CLIEGG     JAR JAR       JOBAL 
##          12           1           4           4           4           4 
##     LAMA SU  MACE WINDU     OBI-WAN        OWEN       PADME    PLO KOON 
##           1           5           6           4          12           0 
##      POGGLE     QUI-GON       RUWEE        SHMI  SIO BIBBLE        SOLA 
##           1           1           4           1           0           4 
##        YODA 
##           4
```

```r
betweenness(g)
```

```
##      ANAKIN BAIL ORGANA        BERU      CLIEGG     JAR JAR       JOBAL 
##   42.600000    0.000000    1.750000    0.500000   22.000000    0.000000 
##     LAMA SU  MACE WINDU     OBI-WAN        OWEN       PADME    PLO KOON 
##    0.000000   18.366667   15.000000    5.250000   55.133333    0.000000 
##      POGGLE     QUI-GON       RUWEE        SHMI  SIO BIBBLE        SOLA 
##    0.000000    0.000000    0.700000    0.000000    0.000000    5.000000 
##        YODA 
##    3.366667
```

To get a nicer representation of the network, see [here](http://tagteam.harvard.edu/hub_feeds/1981/feed_items/1388531) and the formating from igraph to d3Network. Below is the code you’d need:

```r
library(d3Network)
library(networkD3)
sg <- simplify(g)
df <- get.edgelist(g, names=TRUE)
df <- as.data.frame(df)
colnames(df) <- c('source', 'target')
df$value <- rep(1, nrow(df))
# get communities
fc <- fastgreedy.community(g)
com <- membership(fc)
node.info <- data.frame(name=names(com), group=as.vector(com))
links <- data.frame(source=match(df$source, node.info$name)-1,target=match(df$target, node.info$name)-1,value=df$value)

forceNetwork(Links = links, Nodes = node.info,Source = "source", Target = "target",Value = "value", NodeID = "name",Group = "group", opacity = 1, opacityNoHover=1)
```

The nodes represent characters in the movies. The characters are connected by a link if they both speak in the same scene. The colors are for groups obtained by some algorithms.
