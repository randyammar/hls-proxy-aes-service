# HLS Proxy service for AES encrypted HLS stream
[![Build Status](https://sdesyllas.visualstudio.com/hls-proxy-aes-service/_apis/build/status/hls-proxy-aes-service-ASP.NET%20Core%20(.NET%20Framework)-CI)](https://sdesyllas.visualstudio.com/hls-proxy-aes-service/_build/latest?definitionId=2)

# [See a live demo](http://spyros-hls-proxy.azurewebsites.net/hlsplayer.html)

Azure Media Services provides capability for customers to generate an AES encrypted HLS stream with Token authorization configured on the AES key retrieval. However, as we know, Safari handles HLS playlist and key retrieval within the native stack and there is no easy way for developers to intercept the key request and add in Token into the 2nd level HLS Playlist. Here is a proposed solution if you do some magic on your authentication module to make this work. Below is a diagram to illustrate how this solution works: 

![](http://acom.azurecomcdn.net/80C57D/blogmedia/blogmedia/2015/03/03/architecture_thumb.png)

# Basic Usage
Let's say that the streaming HLS Url in Azure media services looks like this :
https://amssamples.streaming.mediaservices.windows.net/830584f8-f0c8-4e41-968b-6538b9380aa5/TearsOfSteelTeaser.ism/manifest(format=m3u8-aapl)&webtoken=Bearer%3deyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1cm46bWljcm9zb2Z0OmF6dXJlOm1lZGlhc2VydmljZXM6Y29udGVudGtleWlkZW50aWZpZXIiOiI5ZGRhMGJjYy01NmZiLTQxNDMtOWQzMi0zYWI5Y2M2ZWE4MGIiLCJpc3MiOiJodHRwOi8vdGVzdGFjcy5jb20vIiwiYXVkIjoidXJuOnRlc3QiLCJleHAiOjE3MTA4MDczODl9.lJXm5hmkp5ArRIAHqVJGefW2bcTzd91iZphoKDwa6w8

Instead of using the Azure Media Services Url directly provide it as a parameter in this proxy servce:
http://spyros-hls-proxy.azurewebsites.net/api/ManifestLoad?playbackUrl=https://amssamples.streaming.mediaservices.windows.net/830584f8-f0c8-4e41-968b-6538b9380aa5/TearsOfSteelTeaser.ism/manifest(format=m3u8-aapl)&webtoken=Bearer%3deyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1cm46bWljcm9zb2Z0OmF6dXJlOm1lZGlhc2VydmljZXM6Y29udGVudGtleWlkZW50aWZpZXIiOiI5ZGRhMGJjYy01NmZiLTQxNDMtOWQzMi0zYWI5Y2M2ZWE4MGIiLCJpc3MiOiJodHRwOi8vdGVzdGFjcy5jb20vIiwiYXVkIjoidXJuOnRlc3QiLCJleHAiOjE3MTA4MDczODl9.lJXm5hmkp5ArRIAHqVJGefW2bcTzd91iZphoKDwa6w8

Then you can use the hls.js or other player to test your video


# What this service does
This service is responsible to inject the token for every request to the video chunks that are defined inside the manifest.

Let’s say the streaming URL looks like this: https://amssamples.streaming.mediaservices.windows.net/830584f8-f0c8-4e41-968b-6538b9380aa5/TearsOfSteelTeaser.ism/manifest(format=m3u8-aapl). Azure Media Services will return the top Playlist to the Authentication system. The top playlist looks like this:

```
#EXTM3U 
#EXT-X-VERSION:4 
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="AAC_und_ch2_96kbps",URI="QualityLevels(92405)/Manifest(AAC_und_ch2_96kbps,format=m3u8-aapl)" 
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="AAC_und_ch2_56kbps",DEFAULT=YES,URI="QualityLevels(53017)/Manifest(AAC_und_ch2_56kbps,format=m3u8-aapl)"
#EXT-X-STREAM-INF:BANDWIDTH=1092766,RESOLUTION=384x288,CODECS="avc1.4d4015,mp4a.40.2",AUDIO="audio"
QualityLevels(960870)/Manifest(video,format=m3u8-aapl)
#EXT-X-STREAM-INF:BANDWIDTH=1607960,RESOLUTION=480x360,CODECS="avc1.4d401e,mp4a.40.2",AUDIO="audio"
QualityLevels(1464974)/Manifest(video,format=m3u8-aapl)
#EXT-X-STREAM-INF:BANDWIDTH=62343,CODECS="mp4a.40.2",AUDIO="audio"
QualityLevels(53017)/Manifest(AAC_und_ch2_56kbps,format=m3u8-aapl) 
```
Modify the top playlist, so the player (Safari in this case) will ping proxy server instead of our key services directly, and add token into the playlist

```
#EXTM3U
#EXT-X-VERSION:4
#EXT-X-ALLOW-CACHE:NO
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-TARGETDURATION:10
#EXT-X-KEY:METHOD=AES-128,URI="https://test.keydelivery.mediaservices.windows.net/?kid=a99263cd-43b3-490a-a4d6-ea04d4645fb7&token=[PUT_YOUR_Token]
#EXT-X-PROGRAM-DATE-TIME:1970-01-01T00:00:00Z
#EXTINF:3.947392,no-desc
http://test.origin.mediaservices.windows.net/fc63efd5-93b0-435e-b4ca-50142cdbcc54/Video_asset_name.ism/QualityLevels(92405)/Fragments(AAC_und_ch2_96kbps=0,format=m3u8-aapl)
#EXT-X-ENDLIST
```

Safari now will send  request based on the URL provided in URI parameter to retrieve 2nd level playlist which contains the key information. Since we changed the playlist to point to our proxy server, the request will come to proxy server

```
#EXTM3U
#EXT-X-VERSION:4
#EXT-X-ALLOW-CACHE:NO
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-TARGETDURATION:10
#EXT-X-KEY:METHOD=AES-128,URI="https://test.keydelivery.mediaservices.windows.net/?kid=a99263cd-43b3-490a-a4d6-ea04d4645fb7&token=[PUT_YOUR_Token]
#EXT-X-PROGRAM-DATE-TIME:1970-01-01T00:00:00Z
#EXTINF:3.947392,no-desc
http://test.origin.mediaservices.windows.net/fc63efd5-93b0-435e-b4ca-50142cdbcc54/Video_asset_name.ism/QualityLevels(92405)/Fragments(AAC_und_ch2_96kbps=0,format=m3u8-aapl)
#EXT-X-ENDLIST
```

more information regarding this solution can be found in this blog post : 
https://azure.microsoft.com/en-gb/blog/how-to-make-token-authorized-aes-encrypted-hls-stream-working-in-safari/

