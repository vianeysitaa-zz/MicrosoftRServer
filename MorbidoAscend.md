
| layout |  title  | author | author-link |  date  |  categories  |  excerpt  |  language  |  verticals  |
|--------|:-------:|:------:|:-----------:|:------:|:-------------:|:--------:|:----------:|:-----------:|
| post | Mórbido Xamarin App | Vianey Juárez Araujo | [@Vianeysitaa](https://twitter.com/VIANEYsitaa) | 2017-02-03 | Mobile Application Development with Xamarin | blue | Microsoft and Mórbido are working together to provide to their fans a new channel for horror an sci-fi movies through a Xamarin app that offers video streaming, podcast, news, and more. | English | Media and Entretainment |

# Solution Overview #

>Mórbido will offer podcast, streaming videos, news and different content for all of Mórbido fans through a Xamarin App that will be available for Android users.

## Key Technologies Used ##
*	Xamarin Froms
*	Visual Studio 2015
*	Media Services
*	SQL Server Azure
*	Azure Blob Storage
*	Azure Virtual Machines

## Core Project Team ##

*	Vianey Juarez Araujo (@VIANEYsitaa) – Technical Evangelist, Microsoft
*	Ricardo Pons (@RicardoPonsDev), Senior Developer, Mórbido 

## Customer Profile ##
>Mórbido is a multiplatform content generator. Started as a film festival but now their services extended to a website, TV network, a film distributor, a radio show, social media pages and a printed magazine.
Mórbido's content revolves around horror, sci-fi and fantasy and generates information on a daily basis, on all its platforms for over 5 million people from all over Latam.

>Mórbido URL: [Mórbido Web Page](http://www.morbidofest.com) 

## Problem statement ##
>“The constant and dynamic transformation of the entertainment industry, coupled with the insatiable interest of our fans to consume all kinds of content at all times, everywhere and in all possible devices and being our project a multiplatform generator of content and events throughout Latin America, led us to the conclusion that a mobile app was the only real option we had to concentrate everything, then we started  developing an application that had the capacity and strength to satisfy our current needs and allow us to continue growing.” (Pablo Guisa Koestinger, Mórbido CEO).

>Mórbido needed a mobile app because they have different audience channels, like the magazine, TV, movies, web page, social media, etc. and they need this app to gather all of this channels. With it they will start collecting all the information from their users (or as they call them, fans) because by now, they don’t have a way to know how many of them they have. With this information, they will be able to offer specific promotions, discounts, or advertise their users.

# Solution, steps and delivery #
## Areas of improvement ##

>Currently, Mórbido doesn’t have a way to count how many of their fans are cross-consuming their products. For example, they don’t have a way to know how many users who buy the magazine, also are viewing the TV channel. Through this app, Mórbido will have a way to know more about their fans and collect info about them.
>In a feature stage of the app, Mórbido plans to add metrics on it to measure how many time does a user spend on the app, the most visited section, the most watched content, etc.

## Challenges ##
>One of the challenges encountered while developing the app, was related to video and audio streaming. Mórbido uses Smooth Streaming protocol to deliver content. The Xamarin native player is not compatible with it.
>The video streaming was overcome by playing video through Rox Xamarin Video. This component allows the app to progressively play video from Azure Media Services. Also, this player has play and pause controls. 
>To use this component, we need to install it from NuGet using the following command.

```shell
Install -Package Rox.Xamarin.Video
```

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/1-NuGetRox.png)

>Implement the player into the project is really simple. Once we get the video URL from the backend web service, we just have to create a view to build the player. Then we just assign the URL by binding.

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/2-BindVideoURL.png)

>To play audio, we had to implement XamarinMediaManager component. In order to be able to play a podcast within the app, first we need to get the podcast URL. 

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/3-PodcastURL.png)

>Once we get the URL, we need to add a specific format for Android (m3u8 format).

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/4-StreamingFormat.png)

>In this way, the player now can play the podcast. 
>The next step is to implement the device native player, and assign the audio file it will play.

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/5-ImplementAudioPlayer.png)

>It is important to mention that all of the timing and playback indicators of the file being played must be carried manually in the ViewModel podcast.
## Code Snippets ##
>Mórbido app connects to the backend through HTTP requests. In order to make it secure, Mórbido implemented OAuth to be able to get the required info in JSON format, so then it could be deserialized and pass it to the app in a clear way.
>In the next code snippet, it is shown how this backend call is made. The user token is sent within the service call, and in this way, be sure it is a secure request.

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/0-OAuth.png)

## Architecture Diagram ##

![alt tag](http://aminespinoza.com/ascend/MorbidoAscend/Architecture diagram.jpg)

## General Lessons ##
* It was really difficult to find information regarding to streaming integration with Xamarin. We expect this documentation can help other developers to solve this quickly.
* Downloading NuGet packages sometimes crashed the application, but it seems to be more stable now.

## Opportunities Going forward ##
>This applicattion will be launched in the second week of March for Android, and it is now in scope to launch the applicattion for iOS in a few months later.
>Also, another opportunity is realated to store metrics from users. In a future stage of the application, they are planning to collect metrics from the users on the app, so they can know how many time they spend on each section, the most visited section, etc., and capitalize this infomration.

## Conclusion ##
> The impact of this app is related to Mórbido fans, and Mórbido itself. Mórbido will start collecting information of their fans into a database, so they can know how many people they are reaching, the different countries they are from, and start offering more content. The fans will be impacted because they can watch Mórbido movies without a TV channel subscription, listen to the podcast and news related to the horror and sci-fi movies.


## References ##
* [Xamarin Media Manager](https://github.com/martijn00/XamarinMediaManager)
* [Rox Xamarin Video](https://www.nuget.org/packages/Rox.Xamarin.Video/)