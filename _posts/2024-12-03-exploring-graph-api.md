---
layout: article
title:  "Exploring Microosft Graph API"
key: post20241203
tags: [microsoft,graph,api,intune,azure]
---
{:refdef: style="text-align: center;"}
![Graph API Logo](/assets/images/graph-api/graph_api_logo.png){:.rounded}
{: refdef}

Graph api is clearly the future for Microsoft toolkits, but it can be confusing to find the endpoints and examples for given tasks. Learn how to "reverse engineer" a graph command from the web.

<!--more-->

### Intro
Microsoft Graph exposes REST APIs and for almost all of Microsoft products. All graph apis start with the single endpoint https://graph.microsoft.com



### Using the Dev Console
In most browsers you can launch a dev console that shows source code, network stats, and other advanced info about a web page or performace of the browser. The dev console will be useful to us by showing what graph url a page in Intune is calling.

1. Launch the dev console
    * Navigating menus you can do Browser Menu > More Tools > Developer Tools
    * most browsers you can hit the f12 key and launch the console
2. Open the network tab
    1. Along the top


### Graph X-ray
[Graph Xray](https://graphxray.merill.net/) is a exention/app made by Merill Fernando and others that is like a Fiddler for Microsoft. It trivializes the process that we went through above. It even will convert the graph commands to several langugages.

1. Install X-ray for your browser. Currently it only supports chrome and edge

2. Launch the Dev Console by typing f12. In the window look along the top for a "+" button



### Graph Explorer





### Extras

[Graph Documentation](https://learn.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0)