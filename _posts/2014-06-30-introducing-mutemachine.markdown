---
layout: post
title:  "Introducing MuteMachine!"
color-class: "green"
---
Any user of [The Hype Machine](http://hypem.com) probably knows how awesome it is to get a fresh batch of new music every week from all around the internet. If you're like me you started using the service and never stopped. While the music selection is mostly great, you've probably also noticed times when that one rogue track totally kills your vibe. Maybe its a track that's already been to the top of the charts before, or maybe you just aren't as big a fan of Lana Del Rey's new album as everyone else is. Whatever the reason, we all want the same thing, for the site to know which songs we don't like and just automatically skip to the next one.

![MuteMachine]({{ site.baseurl }}images/mutemachine-promo.png)

Enter [MuteMachine](https://chrome.google.com/webstore/detail/mutemachine/jccdmapgmmonababhfbflfoekpphbkhg). MuteMachine is a chrome extension which adds a mute button next to the favorite button of any track on hypem. When you click that button, MuteMachine remembers that song and will always skip to the next song whenever that one tries to play. It works on any song on hypem, including ones you've previously favorited and have since gotten sick of. [Download it now and start muting away!](https://chrome.google.com/webstore/detail/mutemachine/jccdmapgmmonababhfbflfoekpphbkhg). 

![MuteMachine in action]({{ site.baseurl }}images/mutemachine-screenshot.png)

If you're interested in checking out the source code for MuteMachine it's open source and available on my [github](http://github.com/akashbad/mutemachine). It's a pretty straightforward extension but there are a couple of tricks I'm using to get it to work.

Firstly, I'm using the trick described by [David Walsh here](http://davidwalsh.name/detect-node-insertion) to detect when new songs are added to the UI. It essentially binds some custom css to the elements that trigger a javascript listenable event whenever they show up:

**CSS**
{% highlight css %}
.favdiv {
  -webkit-animation-duration: 0.001s;
  -webkit-animation-name: favoriteInserted;
}

@-webkit-keyframes favoriteInserted {
  from { clip: rect(1px, auto, auto, auto); }
  to { clip: rect(0px, auto, auto, auto); }
}
{% endhighlight %}

**JS**

{% highlight js %}
var insertListener = function(event){
  if(event.animationName == "favoriteInserted") {
    insertMuteButton(event.target);
  }
}
document.addEventListener("webkitAnimationStart", insertListener, false);
{% endhighlight %}

The second interesting thing I had to do was figure out how to detect that the song was changing and trigger a skip track event. I decided to listen to the top player bar for changes, but it fires about 4 events every song change. That coupled with the fact that instantly triggering a skip event wouldn't register because hypemachine was still in the process of loading and not accepting input. The solution to both was to use javascript timeouts to either wait until I was ready to trigger my event or to repeatedly trigger a skip track until it was registered.

**JS**
{% highlight js %}

var event_timer;
$("#player-nowplaying").on("DOMSubtreeModified", function(event)
  {
        if (event_timer) clearTimeout(event_timer);
            event_timer = setTimeout(skip_song, 100);
  });

var skip_song = function() {
  var song_id = $("#player-nowplaying").find("a[href^='/track/']").attr("href")
.replace("/track/","");
  if(mute_list.hasOwnProperty(song_id)){
    document.dispatchEvent(muteMachineEvent);
    setTimeout(skip_song, 1000);
  }
}
{% endhighlight %}

You may have noticed that in the previous code block, I am skipping a track by dispatching an event to the document. Weird right? Well that brings me to the last interesting part of this (relatively simple) chrome extension, triggering a hypemachine event from my extension. In order to do so, I actually inject a small amount of javascript directly into the page (not in my normal chrome extension content script) which is listening to a custom event. This code actually calls the "nextTrack()" method in the hypem javascript, so I can tap right into their native behavior. Once this script is injected, all that I needed to do is call it by dispatching my custom event.

**JS**
{% highlight js %}
var skip_track_code = "document.addEventListener('muteMachineEvent', function(e)
{nextTrack();});"
var script = document.createElement('script');
script.textContent = skip_track_code;
(document.head||document.documentElement).appendChild(script);
script.parentNode.removeChild(script);
{% endhighlight %}

Thanks for reading! Throw me an upvote or follow the conversation over at [HackerNews]().
