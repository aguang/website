[`Dendextend`](https://cran.r-project.org/web/packages/dendextend/vignettes/dendextend.html)
is a pretty great package for clustering, coloring and labeling
dendrograms, and plays really nicely with `hclust` and `cutree`.
Specifically, it will color branches based on coloring results from
cutree all the way up to the branch nodes rather than just the leaves
with a single command `color_branches(dendrogram, k=n)` where
`dendrogram` is the dendrogram and `n` is the number of clusters. In
most cases this is all a user would want.

    test <- mtcars[, c("mpg", "disp")] %>%
                 dist() %>%
                 hclust(method = "ave")
    c <- cutree(test, k=4)
    dend <- as.dendrogram(test)
    d1 <- color_branches(dend, k=4)
    plot(d1)

![A dendrogram with branches colored by cutree with
k=4.](recoloring-dendrogram_files/figure-markdown_strict/d1-1.png)

Recently I was asked how I could change the colors from `dendextend` so
that they could match up with the numerical ordering of the clusters
from `cutree`. Specifically, the user was pulling out the labels in each
cluster, rendering the labels as text in an RShiny app, and and wanted
the color of othe output to match the cluster colors for easier visual
identification. So if you picked cluster 4 from the `cutree` results
they wanted the color to be what cluster 4 was colored as in the
dendrogram.

The funny thing is that the output of `cutree` follows the indexing of
the labels, while the output of `get_leaves_branches_col` (which returns
the color for each label) follows the indexing of the order, so they
won’t necessarily match up. And the numerical ordering of the clusters
doesn’t match up with the order of the colors either. See below for an
example.

    which(c==4)

    ##  Cadillac Fleetwood Lincoln Continental   Chrysler Imperial 
    ##                  15                  16                  17

    col <- get_leaves_branches_col(d1)
    col

    ##  [1] "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B"
    ##  [8] "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B" "#CC476B"
    ## [15] "#CC476B" "#CC476B" "#767F00" "#767F00" "#767F00" "#009681" "#009681"
    ## [22] "#009681" "#009681" "#009681" "#7866D8" "#7866D8" "#7866D8" "#7866D8"
    ## [29] "#7866D8" "#7866D8" "#7866D8" "#7866D8"

    test$label[test$order]

    ##  [1] "Lotus Europa"        "Toyota Corolla"      "Fiat X1-9"          
    ##  [4] "Fiat 128"            "Honda Civic"         "Datsun 710"         
    ##  [7] "Porsche 914-2"       "Toyota Corona"       "Volvo 142E"         
    ## [10] "Merc 230"            "Merc 240D"           "Ferrari Dino"       
    ## [13] "Mazda RX4"           "Mazda RX4 Wag"       "Merc 280"           
    ## [16] "Merc 280C"           "Chrysler Imperial"   "Cadillac Fleetwood" 
    ## [19] "Lincoln Continental" "Pontiac Firebird"    "Camaro Z28"         
    ## [22] "Ford Pantera L"      "Hornet Sportabout"   "Duster 360"         
    ## [25] "Valiant"             "Dodge Challenger"    "AMC Javelin"        
    ## [28] "Maserati Bora"       "Hornet 4 Drive"      "Merc 450SLC"        
    ## [31] "Merc 450SE"          "Merc 450SL"

We can see that Cadillac Fleetwood, Lincoln Continental, Chrysler
Imperial which are all in cluster 4, have the colors “\#767F00” and are
indexed at 15, 16, 17 in the original data.

There are two ways to match up colors with text output. The first is to
match up each label/leaf with the color assigned to it. The second is to
recolor the dendrogram by the numerical ordering of the clusters. The
first method is probably better for the most part since it doesn’t mess
with the dendrogram itself, but to each their own. Sometimes someone
wants both ways to do it.

Output labels with assigned color from `color_branches`
-------------------------------------------------------

    # labels to colors
    labs_to_col <- test$labels[test$order]
    names(labs_to_col) <- col

    # color of cluster 4
    clust4_col <- col[which(test$labels[test$order] %in% test$labels[c==4])] %>% unique

So here we can output the labels in cluster 4 with the color
`clust4_col=` \#767F00: <span style="color: #767F00;">Cadillac
Fleetwood, Lincoln Continental, Chrysler Imperial</span>. You can see
this color is the same as in the plot for `d1`.

Reorder assigned color in dendrogram by numerical order of clusters
-------------------------------------------------------------------

Doing it this way is a bit quirky not only in that you’re basically just
permuting the colors, but also because the `color_branches` method
doesn’t do what you think it will. For some reason it only colors the
leaves, not any of the parent branches as well. I’m not sure why this
is. Using `set` with `branches_k_color` somehow works.

Both of these work by using `uniq_col[c]` to reorder the colors based on
the numerical assignment of the clusters, then providing the colors as
the indexing of the order.

    uniq_col <- unique(col)
    d2 <- color_branches(dend, col=uniq_col[c][order.dendrogram(dend)])
    plot(d2)

![If you use color\_branches you will only have the leaves
colored.](recoloring-dendrogram_files/figure-markdown_strict/unnamed-chunk-1-1.png)

    d3 <- set(d1, "branches_k_color", value=unique(uniq_col[c][order.dendrogram(dend)]), k=4)
    plot(d3)

![If you use set you will color the whole branch as
well.](recoloring-dendrogram_files/figure-markdown_strict/unnamed-chunk-2-1.png)

So now cluster 4 is actually color 4 (`uniq_col[4]=` \#7866D8) in
`uniq_col`: <span style="color: #7866D8;">Cadillac Fleetwood, Lincoln
Continental, Chrysler Imperial</span>.

Other applications
------------------

I’m sure more will come to mind, but an immediate one is let’s say you
have a specific set of colors that you’d like to use for the cluster
branch colors in your dendrogram. You can set it using
`set("branches_k_color", value=color_vector, k=n)` where `color_vector`
has length `n`.
