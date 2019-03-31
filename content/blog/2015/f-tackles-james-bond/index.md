---
title: "F# tackles James Bond"
date: 2015-11-18
draft: false
type: blog
description: "Earlier this week I read an interesting blog post on visualizing box office success of individual James Bond films using R and ggplot2. In this blog post I'm looking at how to do similar analysis in F#, using the HTML type provider and XPlot."
image: bond-bubble-zoom.png
featured_image: bond-bubble-zoom.png
featured_image_alt: "James Bond box office visualization"
---


Earlier this week I read an interesting article on visualizing box office success of 
James Bond films using R and ggplot2 by Christoph Safferling (
you can find it [here](http://opiateforthemass.es/articles/james-bond-film-ratings/)).
The blog post shows how to pull information from Wikipedia and visualize 
the budget, box office and rating of each film - all this using R. 
While reading the blog post, I couldn't help wondering how would a similar 
analysis look in F# using the HTML type provider from the
[F# Data](http://fsharp.github.io/FSharp.Data/index.html) library. 


<!-- more -->

I was looking for a good excuse to play with the HTML type provider for some time so I decided
to replicate the analysis. And since I saw the 
latest James Bond film Spectre recently, I was also curious how do the individual films differ 
from each other quantitatively. 

The original [blog post](http://opiateforthemass.es/articles/james-bond-film-ratings/) 
looks at how does the box office success of each film compare to its aggregated rating from 
Rotten Tomatoes. All the data are available on Wikipedia in a single 
[article](https://en.wikipedia.org/w/index.php?title=List_of_James_Bond_films&oldid=688916363), so 
all that is needed is to download the data, clean them and visualize them.



## Accessing data with HTML type provider

To download the data from Wikipedia, the original blog post used `rvest` package in R. As I'm using
the HTML type provider in F# to do the same task, the first step is to use the URL of the article
to initialize the type provider. The type provider
then looks at the website and generates statically typed access to all the tables that appear on that website:

{{< highlight FSharp >}}
#load "packages/FsLab/FsLab.fsx"
open FSharp.Data
open XPlot.GoogleCharts

let bondUrl = "https://en.wikipedia.org/w/index.php?title=List_of_James_Bond_films&oldid=688916363"
type BondProvider = HtmlProvider<(*[omit:(...)]*)"https://en.wikipedia.org/w/index.php?title=List_of_James_Bond_films&oldid=688916363"(*[/omit]*)>

let bondWiki = BondProvider.Load(bondUrl) 
{{< /highlight >}}

Accessing the actual data from the Wikipedia article was so quick and easy that I had to create a gif to 
show explicitly how nice and straightforward it is:

![Bond type provider](bondProvider.gif)

One of the main advantages of the HTML type provider is that we can access the tables on the website using
their original headings. The original R version of the code accesses the tables using the
order in which the tables appear on the website so we would need to know the index of the correct table. 

When accessing individual items in the table, we can similarly see all the labels from the table's header and 
directly access all the elements in each row. No need to parse or clean the data manually.

Using the type provider, I pulled the following information about each film: the title, year, the actor, the budget and 
the box office. Both the budet and the box office are adjusted to the equivalent of 2005 
prices to make them comparable between different years. 
Again, the type provider does most of the parsing, except for a few tweaks that the original Wikipedia 
table used to make alphabetic sorting of items in the table easier. Because of this, I had to manually 
adjust some of titles and names of the actors.

{{< highlight FSharp >}}
let boxOffice = 
	let allBoxOffice = 
		[| for row in bondWiki.Tables.``Box office``.Rows ->
			row.Title, row.Year, row.Budget2, 
			row.``Box office 2``, row.``Bond actor`` |]
	allBoxOffice.[1..allBoxOffice.Length-3]
	|> Array.map (fun (titleRaw, yr, bdgt, bo, actorRaw) -> 
		let actor = actorRaw.[actorRaw.Length/2 + 1 .. ]
		let title = 
			match titleRaw |> Seq.tryFindIndex ((=) '!') with
			| Some(idx) -> titleRaw.[idx+1 ..]
			| None -> titleRaw
		title, int yr, float bdgt, float bo, actor)
{{< /highlight >}}		

To keep things simple, I returned all the values as a simple array of tuples containing all the relevant 
information. This could be modified to a proper data frame using [Deedle](http://bluemountaincapital.github.io/Deedle/). 

As the next step, I downloaded the Rotten tomatoes ratings, which are placed in a different table called 
"Reception and accolades" on the same Wikipedia page:

{{< highlight FSharp >}}
let rating = 
	let allRatings = 
		[| for row in bondWiki.Tables.``Reception and accolades``.Rows ->
			row.Film, row.``Rotten Tomatoes`` |]
	allRatings.[0..allRatings.Length-2]
	|> Array.map (fun (title, r) -> 
		title, r.[0..r.IndexOf('%')-1] |> float )
{{< /highlight >}}		

Here I had to strip off some additional information like the percent sign and the total number
of reviews that Wikipedia added into each cell of the table.

## Visualization

Now we are ready to do some charting. I'm using the [XPlot](https://tahahachana.github.io/XPlot/) library 
which provides a nice API for Google Charts (and also plot.ly). To follow the visualization used in the original
blog post, I decided to create a bubble chart. This type of plot in Google Charts 
takes up to four parameters for each data point:
to define its horizontal and vertical position in the plot, and to specify the colour and size of each bubble. 

{{< highlight FSharp >}}
let options =
	Options(
		title = "Bond fims - rating and box office",
		hAxis = Axis(title = "Year"),
		vAxis = Axis(title = "Box office (millions $)"),
		bubble = Bubble(textStyle=TextStyle(color="transparent")),
		colors = [| "red"; "gold" |] )

Array.map2 (fun (title, yr, bdgt, bo, actor) (_, rt) -> 
				title + " (" + actor + ")", yr, bo, rt, bdgt ) boxOffice rating
|> Chart.Bubble
|> Chart.WithLabels(["Title"; "Year"; "Box office"; "Rating"; "Budget"])
|> Chart.WithOptions(options)
{{< /highlight >}}

<script type="text/javascript" src="https://www.google.com/jsapi"></script>
<script type="text/javascript">
    google.load("visualization", "1", {packages:["corechart"]})
    google.setOnLoadCallback(drawChart);
function drawChart() {
    var data = new google.visualization.DataTable({"cols": [{"type": "string" ,"id": "Title" ,"label": "Title" }, {"type": "number" ,"id": "Year" ,"label": "Year" }, {"type": "number" ,"id": "Box office" ,"label": "Box office" }, {"type": "number" ,"id": "Rating" ,"label": "Rating" }, {"type": "number" ,"id": "Budget" ,"label": "Budget" }], "rows" : [{"c" : [{"v": "Dr. No (Sean Connery)"}, {"v": 1962}, {"v": 448.8}, {"v": 96}, {"v": 7}]}, {"c" : [{"v": "From Russia with Love (Sean Connery)"}, {"v": 1963}, {"v": 543.8}, {"v": 96}, {"v": 12.6}]}, {"c" : [{"v": "Goldfinger (Sean Connery)"}, {"v": 1964}, {"v": 820.4}, {"v": 96}, {"v": 18.6}]}, {"c" : [{"v": "Thunderball (Sean Connery)"}, {"v": 1965}, {"v": 848.1}, {"v": 86}, {"v": 41.9}]}, {"c" : [{"v": "You Only Live Twice (Sean Connery)"}, {"v": 1967}, {"v": 514.2}, {"v": 72}, {"v": 59.9}]}, {"c" : [{"v": "On Her Majesty's Secret Service (George Lazenby)"}, {"v": 1969}, {"v": 291.5}, {"v": 82}, {"v": 37.3}]}, {"c" : [{"v": "Diamonds Are Forever (Sean Connery)"}, {"v": 1971}, {"v": 442.5}, {"v": 67}, {"v": 34.7}]}, {"c" : [{"v": "Live and Let Die (Roger Moore)"}, {"v": 1973}, {"v": 460.3}, {"v": 66}, {"v": 30.8}]}, {"c" : [{"v": "The Man with the Golden Gun (Roger Moore)"}, {"v": 1974}, {"v": 334}, {"v": 46}, {"v": 27.7}]}, {"c" : [{"v": "The Spy Who Loved Me (Roger Moore)"}, {"v": 1977}, {"v": 533}, {"v": 79}, {"v": 45.1}]}, {"c" : [{"v": "Moonraker (Roger Moore)"}, {"v": 1979}, {"v": 535}, {"v": 60}, {"v": 91.5}]}, {"c" : [{"v": "For Your Eyes Only (Roger Moore)"}, {"v": 1981}, {"v": 449.4}, {"v": 74}, {"v": 60.2}]}, {"c" : [{"v": "Octopussy (Roger Moore)"}, {"v": 1983}, {"v": 373.8}, {"v": 42}, {"v": 53.9}]}, {"c" : [{"v": "A View to a Kill (Roger Moore)"}, {"v": 1985}, {"v": 275.2}, {"v": 35}, {"v": 54.5}]}, {"c" : [{"v": "The Living Daylights (Timothy Dalton)"}, {"v": 1987}, {"v": 313.5}, {"v": 69}, {"v": 68.8}]}, {"c" : [{"v": "Licence to Kill (Timothy Dalton)"}, {"v": 1989}, {"v": 250.9}, {"v": 76}, {"v": 56.7}]}, {"c" : [{"v": "GoldenEye (Pierce Brosnan)"}, {"v": 1995}, {"v": 518.5}, {"v": 77}, {"v": 76.9}]}, {"c" : [{"v": "Tomorrow Never Dies (Pierce Brosnan)"}, {"v": 1997}, {"v": 463.2}, {"v": 57}, {"v": 133.9}]}, {"c" : [{"v": "The World Is Not Enough (Pierce Brosnan)"}, {"v": 1999}, {"v": 439.5}, {"v": 51}, {"v": 158.3}]}, {"c" : [{"v": "Die Another Day (Pierce Brosnan)"}, {"v": 2002}, {"v": 465.4}, {"v": 58}, {"v": 154.2}]}, {"c" : [{"v": "Casino Royale (Daniel Craig)"}, {"v": 2006}, {"v": 581.5}, {"v": 95}, {"v": 145.3}]}, {"c" : [{"v": "Quantum of Solace (Daniel Craig)"}, {"v": 2008}, {"v": 514.2}, {"v": 65}, {"v": 181.4}]}, {"c" : [{"v": "Skyfall (Daniel Craig)"}, {"v": 2012}, {"v": 879.8}, {"v": 93}, {"v": 158.1}]}]});
    var options = {"colors":["red","gold"],"hAxis":{"title":"Year"},"legend":{"position":"none"},"title":"Bond fims - rating and box office","vAxis":{"title":"Box office (millions $)"},"bubble":{"textStyle":{"color":"transparent"}}} 
    var chart = new google.visualization.BubbleChart(document.getElementById('7079c2e6-91fe-4ff3-96c1-f778b22874df'));
    chart.draw(data, options);
}
</script>

<div id="7079c2e6-91fe-4ff3-96c1-f778b22874df" style="width: 100%; height: 500px;"></div>

In the resulting plot, the x axis represents the year and the y axis shows the box office success. 
The size of each bubble represents the budget of each film, and finally the colours show the Rotten Tomatoes
rating where lighter colours mean higher rating. Also, hover over the individual bubbles to see all the 
information about the films including their title and the James Bond actors. 

This plot shows that the first few James Bond films with Sean Connery were very successfull, 
with small budgets, high aggregated rating and growing revenues. The revenue was gradually decreasing
over the Roger Moore and Timothy Dalton era. The box office again improved after 1995 with Pierce
Brosnan, and the budgets increased as well. Two of the most recent films with Daniel Craig, Casino
Royal and Skyfall, also reached similarly high aggregated ratings as some of the older Sean Connery films. 
Unfortunately, there are no available data on Spectre at the moment so we don't know how does it
compare to the previous films. 

From the source code perspective, this all took about 45 lines of F# code, 
compared to 125 lines of R code that were used in the original blog 
post to produce a similar bubble chart with ggplot2 
([see the original R code](https://github.com/safferli/james_bond_films/blob/master/007.R)). 
The difference in 
size is mainly because the R code parses all the data in the Wikipedia table manually to create a 
clean data frame. In F#, the HTML type provider already does most of the work that is required to get
the data into a usable shape. 

I also tried to replicate the resulting ggplot figure from the original blog post to test the capabilities 
of the F# [RPRovider](http://bluemountaincapital.github.io/FSharpRProvider/), which is a mechanism of 
accessing functions from R within F#. Using ggplot2 from F# with RProvider is a little more verbose than in 
R, but the full F# code still came out shorter than the the full R code (125 lines in F# compared to 187 lines in R). 
The code I used to generate the bubble chart and the full ggplot2 chart is available as a 
[gist](https://gist.github.com/evelinag/0ce68655f2aae1ecabcb).

![James Bond with ggplot](bond-ggplot.png)

## Summary 

Overall, I was very happy with the whole F# HTML type provider experience. It allows accessing most of the
elements from the webpage using their real human-readable names and it does a great job parsing the data from 
the tables. Compared to the R version, there is less work left to be done to get the data into a good
shape and prepare it for visualisation. 

Finally, here is the source code that I used and some other relevant links:

* [Source code](https://gist.github.com/evelinag/0ce68655f2aae1ecabcb)
* This blog post uses FsLab, you can download a template [here](http://fslab.org/download/)
* [Blog post that inspired all this](http://opiateforthemass.es/articles/james-bond-film-ratings/)




