---
layout: post
title:  "Motion JPEG in ASP.Net Core"
date:   2019-11-23 21:44:38 +0200
tags: [aspnetcore, code]
---

# Background

I have recently setup some IP cameras and a network video recorder (NVR) at home. The cameras and NVR are accessible via a built-in web interface but viewing the images/video requires a web browser plugin to be installed. With HTML5 and modern browsers these plugins are completely unnecessary and only make me feel uneasy from a secuirty point of view.

To fix this I have built a small web site for viewing the camera images from any web browser. I'm using ASP.Net Core on a Raspberry Pi. For now, it only shows static images (the IP cameras have a useful endpoint to HTTP GET a JPEG snapshot) but a future goal is to use the camera’s H.264/5 video stream via RTSP.

As an intermediate step I want the static images to automatically update every few seconds without having to hit refresh in the browser. One solution is to use java script to periodically reload the image in the browser, the other (discussed here) is for the web server to send a stream of JPEG images aka [motion JPEG (MJPEG)]( https://en.wikipedia.org/wiki/Motion_JPEG).


# How Motion JPEG over HTTP works

A standard HTML `img` tag is used to place the image on the web page:

```html
<img src="/Home/LiveViewCamera/2">
```

The browser uses an `HTTP GET` to retrieve the image. 
The server responds with the data of the first JPEG image. 
The HTTP connection is kept open and used to send the data for the next JPEG after a short delay. 
This repeats until the browser or server closes the connection. 
To send multiple images in this way the special MIME type `Content-Type: multipart/x-mixed-replace` is used. This indicates that the response contains multiple parts, where each part replaces the previous part.

Here is an example HTTP session:

Browser requests the image:

```http
GET /Home/LiveViewCamera/2 HTTP/1.1
Host: localhost:5000
```
Server response:
```http
HTTP/1.1 200 OK
Date: Sun, 23 Nov 2019 11:10:52 GMT
Content-Type: multipart/x-mixed-replace;boundary=MotionImageStream
Server: Kestrel
Transfer-Encoding: chunked

--MotionImageStream
Content-Type: image/jpeg
Content-Length: 268265

<JPEG image data for 1st image>
--MotionImageStream
Content-Type: image/jpeg
Content-Length: 267213

<JPEG image data for 2nd image>
```
The MIME header and JPEG image data sections are repeated for subsequent images.


# Generating MJPEG responses in ASP.Net Core 

We can use the following MVC controller implementation to return an MJPEG stream:

```csharp
public IActionResult LiveViewCamera(string id)
{
    var source = new CameraImageSource(id);

    var first = true;
    return new MjpegStreamContent(async cancellationToken => {
        if (first)
            first = false; // no delay for first image
        else
            await Task.Delay(2000, cancellationToken); // 2 second delay between image updates
        return await source.GetImageAsync(cancellationToken);
    },
    source.Dispose);
}
```

Here `CameraImageSource` is a wrapper for an `HttpClient` that provides a `GetImageAsync` to get the latest JPEG image from the IP camera.
The `MjpegStreamContent` class contains all the logic to generate the MJPEG response.
Its constructor takes a delegate that returns the data for a single image. This delegate allows us to decouple the image retrieval code from the MJPEG response generation code.

`MjpegStreamContent` is implemented as follows:

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using System.Text;
using System.Threading;

public class MjpegStreamContent : IActionResult
{
    private static readonly string _boundary = "MotionImageStream";
    private static readonly string _contentType = "multipart/x-mixed-replace;boundary=" + _boundary;
    private static readonly byte[] _newLine = Encoding.UTF8.GetBytes("\r\n");

    private readonly Func<CancellationToken, Task<byte[]>> _onNextImage;
    private readonly Action _onEnd;

    public MjpegStreamContent(Func<CancellationToken, Task<byte[]>> onNextImage, Action onEnd)
    {
        _onNextImage = onNextImage;
        _onEnd = onEnd;
    }

    public async Task ExecuteResultAsync(ActionContext context)
    {
        context.HttpContext.Response.ContentType = _contentType;

        var outputStream = context.HttpContext.Response.Body;
        var cancellationToken = context.HttpContext.RequestAborted;

        try
        {
            while (true)
            {
                var imageBytes = await _onNextImage(cancellationToken);

                var header = $"--{_boundary}\r\nContent-Type: image/jpeg\r\nContent-Length: {imageBytes.Length}\r\n\r\n";
                var headerData = Encoding.UTF8.GetBytes(header);
                await outputStream.WriteAsync(headerData, 0, headerData.Length, cancellationToken);
                await outputStream.WriteAsync(imageBytes, 0, imageBytes.Length, cancellationToken);
                await outputStream.WriteAsync(_newLine, 0, _newLine.Length, cancellationToken);

                if (cancellationToken.IsCancellationRequested)
                    break;
            }
        }
        catch (TaskCanceledException)
        {
            // connection closed, no need to report this
        }
        finally
        {
            _onEnd();
        }
    }
}
```
`ExecuteResultAsync` contains a loop that gets the image data using the `_onNextImage` delegate and then writes that data to the response steam with the needed MIME headers.
The loop continues until the `HttpContext.RequestAborted` cancellation token is set (this happens when the connection drops or is closed by the browser).
The `_onEnd` delegate allows for disposing of any resources used for this request e.g. the `CameraImageSource`.

