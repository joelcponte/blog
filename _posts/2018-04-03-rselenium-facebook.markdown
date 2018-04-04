---
layout: post
title: Scraping and Visualizing my Network of Friends With RSelenium and D3.js
date: 2018-04-03 13:32:20 +0300
description:  # Add post description (optional)
img: network_of_friends.jpg # Add image post (optional)
tags: [R, scraping, RSelenium, data-visualization, D3.js]
---

***tl;dr: Visualization in d3.js that shows conexions among Facebook friends and indentifies groups of friends***

Years ago, when Facebook's API was more permissive, I remember of using an app that provided a visualization of my network of friends. I intended to take snapshots of the network over time to see its growth as I met more people, but (un?)fortunately Facebook's API stopped allowing users to retrieve the list of their mutual friends with their friends, which was crucial information to create that visualization. For this reason that app inevitably died.

In this post I will show how I recreated that visualization. I used R and the package RSelenium to scrape the data from Facebook and D3.JS to create a visualization.

## Strategy

The strategy for this project was the following:

1. Go to my own "Friends" page to scrape the necessary information about them. (quantity, names, ids and usernames).
2. Loop through this list of friends and get their list of mutual friends by scraping their "Friends" page.

It sounds simple but Facebook has a lot of little details that we must take into consideration:
- The "Friends" page of your friends will change depending on their privacy settings.
- Everyone on Facebook have ids, but most users have also usernames. The links to their pages depend on whether they have usernames or not.
- When you visit a "Friends" page, not all friends are immediately shown. More and more friends are loaded as you scroll down.
- Some people who deleted their Facebook will still appear in your friends list, but their profile and "Friends" page will be unnacessible.

## R scrapping

We begin by starting RSelenium as usual and loging in Facebook. If you are not familiar with RSelenium I recommend [taking a look here](https://ropensci.org/tutorials/rselenium_tutorial/) and [here](http://rpubs.com/johndharrison/RSelenium-Docker).

{% highlight r%}
remDr <- remoteDriver(remoteServerAddr = "XXXXX", port = 4445L, browserName = "firefox")
remDr$open()

remDr$setTimeout(type = "page load", milliseconds = 99999999999)
remDr$navigate("http://www.facebook.com")

## log in
login = "XXXXXX" # PUT LOGIN HERE
password = "XXXXXX" # PUT PASSWORD HERE

## go to login field and fill
txtfield <- remDr$findElement(using = 'css selector', "#email")
txtfield$sendKeysToElement(list(login))

## go to password field and fill
txtfield <- remDr$findElement(using = 'css selector', "#pass")
txtfield$sendKeysToElement(list(password))

## click to log in
wxbutton <- remDr$findElement(using = 'css selector', "#u_0_2")
wxbutton$clickElement()
{% endhighlight %}

To find the css selectors, I had the help of the Google Chrome extension [SelectorGadget](http://selectorgadget.com/). It didn't work always so sometimes I had to inspect the page manually to find the CSS selectors.

Once logged in, we must go to our own Friends page to retrieve the necessary information about our Friends.

{% highlight r%}
## go to your profile page
wxbutton <- remDr$findElement(using = 'css selector', "#userNav .noCount")
wxbutton$clickElement()

## go to your friends page
wxbutton <- remDr$findElement(using = 'css selector', 
    '#fbTimelineHeadline [data-tab-key="friends"]')
wxbutton$clickElement()

#get number of friends
page = read_html(remDr$getPageSource()[[1]])
n_friends = html_nodes(page, "._3d0") %>% html_text
n_friends = as.numeric(n_friends[1])
{% endhighlight %}

![alt text]({{site.baseurl}}/assets/img/my_friends.jpg)

Only a few of your friends are shown when you load this page. The rest of your friends only show up if you scroll down, so we have to tell selenium to scroll down the page until we can see as many friends as the number of friends that we know that we have:


{% highlight r%}
friends = 1
# scroll down page down to the bottom
while(length(friends) < n_friends) {
  
  webElem <- remDr$findElement("css", "body")
  webElem$sendKeysToElement(list(key = "end"))
  
  #get number of friends shown now
  page = read_html(remDr$getPageSource()[[1]])
  friends = html_nodes(page, ".fcb a")
  
  #collected friends should be increasing
  print(paste0("collected friends: ", length(friends), " < total friends: ", n_friends))
}
# select only friends (the css selector gets other elements once all friends are found)
friends = friends[1:n_friends]
{% endhighlight %}


Now that the page shows all of our friends, we can collect information about them.

We don't know which friends have usernames or not. For friends who have usernames, we retrieve something close to:

{% highlight r%}
friends[532] %>% as.character()
[1] "<a data-hovercard-prefer-more-content-show=\"1\" data-hovercard=\"/ajax/hovercard/ \
  user.php?id=ID_HERE&amp; extragetparams=%7B%22hc_location%22%3A%22friends_tab%22%7D\" \
  data-gt='{\"engagement\":{\"eng_type\":\"1\",\"eng_src\":\"2\",\"eng_tid\":\"1437964297\" \
  ,\"eng_data\":[]},\"coeff2_registry_key\":\"0406\",\"coeff2_info\":\" \
  AasBsWStelnLVpVZEdT6HQTiPfwTiAYdkcrKSeqdhMm4ZZPrVgYGvuD20JFAuSLmOiYKgtyMMmiyOHkVxeGLURYV\" \
  ,\"coeff2_action\":\"1\",\" coeff2_pv_signature\":\"2045669656\"}' href=\" \
  https://www.facebook.com/USERNAME_HERE?fref=pb&amp; hc_location=friends_tab\">User Name</a>"
{% endhighlight %}

For friends who don't have usernames, we stored in the variable `friends` something like:

{% highlight r%}
[1] <a role="button" ajaxify="/ajax/friends/inactive/dialog?id=ID_HERE" \
  rel="dialog" href="#">User Name</a>
{% endhighlight %}

We now try to extract both usernames and ids using regular expressions and we say that the user has an id and not an username if the string retrieved is too big. The trick here is the following: we use the code `sub(".*facebook\\.com/([A-Za-z0-9\\.]*)\\?fref.*", "\\1", x)` to get the username. If the user has an username, it will be properly retrieved. If the user doesn't, it will fail to match the regex so it will retrieve the whole id string, which is big. After some analysis I found the value of 50 to be a good threshold. That is, it the retrieved friend username has more than 50 characters, it means that it is not an username, and therefore the user doesn't have one.





{% highlight r%}
## retrieve ids and usernames
friends_names = html_text(friends)
friends_ids = sapply(friends, function(x) return(sub(".*id=([0-9]*).*", "\\1", x)))
friends_usernames = sapply(friends, 
    function(x) return(sub(".*facebook\\.com/([A-Za-z0-9\\.]*)\\?fref.*", "\\1", x)))
friends_usernames[nchar(friends_usernames)>50] = NA
{% endhighlight %}

Now we define a function to help us retrieve mutual friends page. It will depend on wether the user does or doesn't have an username.


{% highlight r%}
fb_mutual_friends_page <- function(string, type = "username") {

  if (type == "username") {
    return(paste0("https://www.facebook.com/", string, "/friends_mutual"))
  }
  if (type == "id") {
    return(paste0("https://www.facebook.com/profile.php?id=", 
        string, "&sk=friends_mutual&pnref=lhc"))
  } else {
    print("Choose a valid type")
    return(NULL)
  }
}
{% endhighlight %}

And create a data frame to organize the collected information about the friends:


{% highlight r%}
friends_df = data.frame(name = friends_names,
                        friends_usernames = friends_usernames,
                        friends_ids = friends_ids,
                        link_id = fb_mutual_friends_page(friends_ids, "id"),
                        link_username = fb_mutual_friends_page(friends_usernames),
                        friends_ids = friends_ids,
                        friends_usernames = friends_usernames,
                        stringsAsFactors = F)

# define link independently of having or not usernames
friends_df$link = friends_df$link_id
friends_df$link[!is.na(friends_usernames)] = friends_df$link_username[!is.na(friends_usernames)]
friends_df$link_username[is.na(friends_df$friends_usernames)] = NA
{% endhighlight %}


Finally, it's time to get the mutual friends list. We loop through all friends, going to their "link" column in the data frame to visit their own "Friends" page. Once we finish scrolling down, we add the mutual friends to a list and move to the next iteration.

{% highlight r%}
#loops though all friends and collect your mutual friends with them
for (i in 1:n_friends) {
    cat("Scrapping friend ", i, "out of ", n_friends, "...\n")
    remDr$navigate(friends_df$link[i])
    current_page = read_html(remDr$getPageSource()[[1]])
    n_friends_now = html_nodes(current_page, ".fsl.fwb.fcb") %>% html_text %>% length()
    mutual_friends = html_nodes(current_page, "[name='Mutual Friends'] ._3d0") %>%
        html_text %>% 
        as.numeric
    mutual_friends = mutual_friends[1]
    
    #check if friend has deleted page
    deleted_text = tryCatch({html_nodes(current_page, ".uiHeaderTitle") %>% 
        tail(1) %>%
        html_text()}, error = function(e) {'page not deleted!'})
        
    if ( deleted_text == "Sorry, this content isn't available right now") next
    #check if friend has no mutual friendship
    if ( is.na(mutual_friends) ) next
    
    # scroll down to load all mutual friends
    while (n_friends_now < mutual_friends) {
      webElem <- remDr$findElement("css", "body")
      webElem$sendKeysToElement(list(key = "end"))
      n_friends_now = html_nodes(current_page, ".fsl.fwb.fcb") %>% html_text %>% length()
      current_page = read_html(remDr$getPageSource()[[1]])
    }
    
    mutual_friends = html_nodes(current_page, ".fcb a")
    mutual_friends = mutual_friends[1:n_friends_now]
    mutual_friends_ids = sapply(mutual_friends, function(x) return(sub(".*id=([0-9]*).*", "\\1", x)))
    mutual_friends_ids_all[[i]] <- mutual_friends_ids
}
{% endhighlight %}


## Preparing data for visualization

At this point we should have a list that looks like:

{% highlight r%}
> mutual_friends_ids_all %>% head(3)
[[1]]
 [1] "xxxxx" "xxxxx" "xxxxx" 


[[2]]
 [1] "xxxxx"  "xxxxx"  "xxxxx"  "xxxxx"  "xxxxx"  "xxxxx" 


[[3]]
 [1] "xxxxx" "xxxxxx"
{% endhighlight %}


It would be nice to have some clusterization of friends to encode as color for the visualization and to do that we will use the package `igraph` and a label propagation algorithm. *Disclaimer: I didn't study anything about the available methods to do this so please note that I am just applying a function that I found somewhere and there are probably better ways to do this.*

We need to transform a little the data to learn the clusters:


{% highlight r%}
N = nrow(friends_df)
friends_matrix = matrix(0,N,N,dimnames = list(friends_df$friends_ids, friends_df$friends_ids))
for (i in 1:N) {
  friends_matrix[i,friends_df$friends_ids %in% mutual_friends_ids_all[[i]]] = 1
}

colnames(friends_matrix) = friends_df$name
rownames(friends_matrix) = friends_df$name

ga.data <- do.call(rbind, lapply(1:N, function(i) {
  do.call(rbind, lapply(1:N, function(j) {
    if(friends_matrix[i,j]==1) return(data.frame(from=friends_df$name[i], to=friends_df$name[j]))
  }))
}))

g <- graph.data.frame(ga.data, directed=FALSE)

wc <- label.propagation.community(g)
{% endhighlight %}
<p></p>
{% highlight r%}
> head(ga.data, 2)
  from            to
1 xxxxxx          zzzzzzz
2 xxxxxx          yyyyyyy
{% endhighlight %}
<p></p>
{% highlight r%}
> wc
IGRAPH clustering label propagation, groups: 10, mod: 0.61
+ groups:
  $`1`
   [1] "xxxxxx"           "yyyyyy"                  "zzzzzzz"             "wwwwww"                   
  + ... omitted several groups/vertices
 {% endhighlight %}


Great! With the links and clusters in hand, we just have to save the data in a way our d3.js script will read.


{% highlight r%}
links = ga.data
names(links) = c("source", "target")
nodes = data.frame(name = NULL, group = NULL)
for (i in 1:length(wc)) {
  nodes = rbind(nodes, data.frame(wc[[i]], rep(i, length(wc[[i]]))))
}
names(nodes) = c("name", "group")


n_links = do.call("rbind", mutual_friends_ids_all %>% lapply(length))
n_links = cbind(friends_df[,"name", drop = F], n_links)
nodes = merge(nodes, n_links, by = "name", sort = F)
names(nodes)[1] = "id"
#in case there's a mistake and someone appears in links but not nodes...
links = links[links$source %in% nodes$id,]
links = links[links$target %in% nodes$id,]
{% endhighlight %}
<p></p>
{% highlight r%}
> head(links,2)
  source         target
1 xxxxx      zzzzzz
2 xxxxx      yyyyyy
{% endhighlight %}
<p></p>
{% highlight r%}
> head(nodes,2)
  id group n_links
1 xxxxx     1      22
2 yyyyy     1      31
{% endhighlight %}


And save to json
{% highlight r%}
json_nodes = rjson::toJSON(unname(split(nodes, 1:nrow(nodes))))
json_links = rjson::toJSON(unname(split(links, 1:nrow(links))))
write_file(paste0('{\n"nodes": ', json_nodes, ', \n', '"links": ', json_links, "\n}"),
           "data.json")
{% endhighlight %}


## D3.js visualization

For this I [got a ready-to-go visualization](https://bl.ocks.org/heybignick/3faf257bbbbc7743bb72310d03b86ee8) and made some changes. More specifically, I made it possible to zoom in and out, encoded the size of the circles with the number of mutual friends and changed the force. See the code below and the result [**here**]({{site.baseurl}}/assets/d3js/vis.html). For the whole code checkout [my github repository](https://github.com/joelcponte/rselenium-facebook-mutualfriends-scraper).

I found my network to be very interesting. Some background about myself: I'm from a city in Brazil called Fortaleza. Although more than 2.5 million people live there, it is common to say that everyone in their 20's knows each other. That seems to be the case (at least in my network), as shown in the big blue cluster in the middle. It contains close friends, friends from middle and highschool and completely unrelated people that I met through many different ways. Close to that there is a gray cluster that contains mostly family members. Other two small clusters are in red and orange, which are respectively ex-roomates/landlord and coworkers. Then, there are four clusters far from the middle: people from home university in Brazil, people from the exchange that I did in Canada, people from my master's in the US and the very small of people from my current master's in France.

<!-- <iframe src="{{site.baseurl}}/assets/d3js/vis.html" width="100%" height="500" marginwidth="0" marginheight="0" scrolling="no" frameBorder="0"></iframe> -->

{% highlight javascript%}
<!DOCTYPE html>
<meta charset="utf-8">
<style>

.links line {
  stroke: #999;
  stroke-opacity: 0.6;
}

.nodes circle {
  stroke: #fff;
  stroke-width: 1.5px;
}

text {
  font-family: sans-serif;
  font-size: 10px;
}

</style>
<svg width="1920" height="1080"></svg>
<script src="https://d3js.org/d3.v4.min.js"></script>
<script>

var svg = d3.select("svg"),
    width = +svg.attr("width"),
    height = +svg.attr("height");

var color = d3.scaleOrdinal(d3.schemeCategory20);

var simulation = d3.forceSimulation()
    .force("link", d3.forceLink().id(function(d) { return d.id; }))
    .force("charge", d3.forceManyBody().strength(-100))
    .force("center", d3.forceCenter(width / 2, height / 2));

//add encompassing group for the zoom 
  var g = svg.append("g")
    .attr("class", "everything");
  //add zoom capabilities 
  var zoom_handler = d3.zoom()
      .on("zoom", zoom_actions);

  zoom_handler(svg);     

d3.json("temp.json", function(error, graph) {
  if (error) throw error;
  console.log("asdaSD")
  debugger;
  var link = g.append("g")
      .attr("class", "links")
    .selectAll("line")
    .data(graph.links)
    .enter().append("line")
      .attr("stroke-width", 0.5);

  var node = g.append("g")
      .attr("class", "nodes")
    .selectAll("g")
    .data(graph.nodes)
    .enter().append("g")
    
  var circles = node.append("circle")
      .attr("r", function(d) {return 2*Math.sqrt(d.n_links);})
      .attr("fill", function(d) { return color(d.group); })
      .call(d3.drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended));

  var lables = node.append("text")
      .text(function(d) {
        return d.id;
      })
      .attr('x', 6)
      .attr('y', 3);

  node.append("title")
      .text(function(d) { return d.id; });

  simulation
      .nodes(graph.nodes)
      .on("tick", ticked);

  simulation.force("link")
      .links(graph.links);



  function ticked() {
    link
        .attr("x1", function(d) { return d.source.x; })
        .attr("y1", function(d) { return d.source.y; })
        .attr("x2", function(d) { return d.target.x; })
        .attr("y2", function(d) { return d.target.y; });

    node
        .attr("transform", function(d) {
          return "translate(" + d.x + "," + d.y + ")";
        })
  }
});
function zoom_actions(){
    g.attr("transform", d3.event.transform)
}
function dragstarted(d) {
  if (!d3.event.active) simulation.alphaTarget(0.3).restart();
  d.fx = d.x;
  d.fy = d.y;
}

function dragged(d) {
  d.fx = d3.event.x;
  d.fy = d3.event.y;
}

function dragended(d) {
  if (!d3.event.active) simulation.alphaTarget(0);
  d.fx = null;
  d.fy = null;
}

</script>
{% endhighlight %}
