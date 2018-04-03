---
layout: post
title: Music exploration and visualization with Spotify and Shiny.
date: 2018-04-03 13:32:20 +0300
description: # Add post description (optional)
img: spotify-sim.gif # Add image post (optional)
tags: [spotify, data-visualization, shiny, machine-learning, R]
---

A while ago [this post](http://www.rcharlie.com/post/fitter-happier/) became viral among two groups that I am part of: R users and Radiohead fans. I took that, my willingess to improve my Shiny skills and my personal interest in understanding and comparing music to create a Shiny app that process music data using Spotify's API.

The code can be found on [my github](https://github.com/joelcponte/shiny-app-spotify) and the app can be accessed joelcponte.shinyapps.io/spotifyapp.

## Great, but what can I do with it?

Select one or multiple artists. In the first tab, find their most (and least) similar songs. Heard a song you liked and wanna know other songs from that artists that are similar to it? Tipe its name in the search of the "Most similar songs" box and rank them accordngly.

You can also see a scatterplot of pca-reduction to 2 or 3 dimensions of the songs features (such as energy and instrumentalness) to see where each song lies on that reduced space. If you pick one artist, the points will be colored by albums, and if you pick multiple artists, they will be colored by artists. You can see for example how different the albums of an artist are among themselves or how multiple artists are more or less similar. In the gif that opens this post we compared Radiohead and Oasis and the scatterplot indicates that Radiohead songs tend to have more "Acousticness", "Instrumentalness" and "Danceability" whereas Oasis songs tend to have more "Liveness", "Loudness" and "Energy".



In the second tab, you can find energetic and calm playlists generated with the artists and albums you selected.


## How was the playlist recommender created?

First, I downloaded playlists with the keywords "running" and "relax" through Spotify's api and labelled the songs accordingly. Spotify provides a set of features for all of their songs which I used to create a training set where the Spotify features were the inputs and "running" and "relax" the outputs. A logistic regression model was trained to predict to which playlist new songs are most likely to belong.


## Next steps

I few things that in mind:

- Add lyris information.
- Make it possible to export the playlists to Spotify.
- Playlist: retrieve songs similar to a list of songs.
- And some other which may be secret by now.