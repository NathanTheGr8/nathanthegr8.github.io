---
layout: article
title:  "Exploring Microsoft Graph API"
key: post20241203
tags: [microsoft,graph,api,intune,azure]
---
{:refdef: style="text-align: center;"}
![Graph API Logo](/assets/images/graph-api/graph_api_logo.png){:.rounded}
{: refdef}

Graph api is clearly the future for Microsoft toolkits, but it can be confusing to find the endpoints (urls) and examples for given tasks. In this post we will learn how to "reverse engineer" a graph command from the clicking buttons on a web page.

<!--more-->

### Intro
Microsoft Graph exposes REST APIs and for almost all of Microsoft products. All graph apis start with the single endpoint https://graph.microsoft.com



### Using the Dev Console
In most browsers you can launch a dev console that shows source code, network stats, and other advanced info about a web page or performance of the browser. The dev console will be useful to us by showing what graph url a page in Intune is calling.

1. Launch the dev console
    * Navigating menus you can do Browser Menu > More Tools > Developer Tools
    * most browsers you can hit the f12 key and launch the console
2. Open the network tab
    1. Along the top of the dev console you will see tab. Find the tab called network. It will have a wifi like logo. ![Console Title Bar](/assets/images/graph-api/ConsoleTitleBar.png)
    2. Once you open the network tab it will start in a recording mode. Navigating to another page or clicking a button will generate a row for the assocated network call the web page made. 
    ![Network Capture Table](/assets/images/graph-api/NetworkCaptureTable.png)
    Pages can make a lot of calls so it is best to start and stop the recording to capture only actions you are trying to investigate. To stop a recording press the red square surrounded by a circle. Top clear a recording click the circle with a diagonal line through it. ![Network Start Stop](/assets/images/graph-api/Start-Stop-Network.png)


### Graph X-ray
[Graph Xray](https://graphxray.merill.net/) is a extension/app made by [Merill Fernando](https://merill.net/) and others that is like a [Fiddler](https://www.telerik.com/blogs/how-to-test-your-api-with-fiddler) for Microsoft. It trivializes the process that we went through above. It even will convert the graph commands to several languages.

1. Install X-ray for your browser. Currently it only supports chrome and edge
    * [Chrome Extension](https://chromewebstore.google.com/detail/graph-x-ray/gdhbldfajbedclijgcmmmobdbnjhnpdh)
    * [Edge Extension](https://microsoftedge.microsoft.com/addons/detail/graph-xray/oplgganppgjhpihgciiifejplnnpodak)

2. Launch the Dev Console by typing f12. In the window look along the top for a "+" button



### Graph Explorer
Another useful tool for testing/exploring graph api is the [graph explorer](https://developer.microsoft.com/en-us/graph/graph-explorer). It is a website create by Microsoft where you can try queries against a test tenant or your production environment. 




### Extras

[Graph Documentation](https://learn.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0)