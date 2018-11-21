---
title: "Streaming video from AWS S3 to the browser using a Flask API"
layout: post
date: 2018-11-14
image: /assets/images/markdown.jpg
headerImage: false
tag:
- aws
- S3
- Flask
- video
- stream
- Safari
- python
star: true
category: blog
author: petter
description: A demo on streaming video content from S3 to both Safari and Chrome
---
## What we will do
Let's assume we have a media service, hosting images and video through our website. What we will do here is just do a
basic example of how you could actually host a solution like this in AWS, using S3 for media storage. Usually, with 
hosting media, you would probably store the URL or path of the media in a separate database, but I let you figure that
part out, and here we will only focus on the streaming. So maybe in real life, the endpoint would take a URL of an image
on the www, which we have saved in S3. But in this case I will just assume that whoever is requesting the content
already knows the S3 key/path to the resource (which is not very likely...).

## Let's get "coding"!
So, this application is pretty straight forward, except for one thing: different browsers request content in different
ways through HTTP requests. Or, to be more specific - all browsers requests content in the same way except Safari. 
Hard to imagine, right? I guess it has something to do with the iPhone back in the days (when we were on 2G and older)
wanted to know before hand that the content it was collecting was actually the real deal, and not waste time on collecting
bullshit. But seriously Apple, that was a while back now, and you need to get in shape. Safari is the new Internet Explorer.  

So if we wanted to write this application for all browsers _except_ Safari it would look something like: 
{% highlight python %}
import os
import requests
import uuid
import boto3
from flask import Flask, jsonify, request, Response, stream_with_context
from werkzeug.datastructures import Headers
import json
import re

bucket_name = 's3_media'
service = Flask(__name__)


@service.route('/<key>', methods=['GET'])
def get_specific_media(key):

    storage = boto3.client('s3')
    media_stream = storage.get_object(Bucket=bucket_name, Key=key)
    full_content = media_stream['ContentLength']
    headers = Headers()
    status = 200
    range_header = request.headers.get('Range', None)
    if range_header:
        byte_start, byte_end, length = get_byte_range(range_header)

    headers.add('Content-Type', content_type)
    headers.add('Content-Length', media_stream['ContentLength'])
    response = Response(
        stream_with_context(media_stream['Body'].iter_chunks()),
        mimetype=content_type,
        content_type=content_type,
        headers=headers,
        status=status
        )
    return response


def get_byte_range(range_header):
    g = re.search('(\d+)-(\d*)', range_header).groups()
    byte1, byte2, length = 0, None, None
    if g[0]:
        byte1 = int(g[0])
    if g[1]:
        byte2 = int(g[1])
        length = byte2 + 1 - byte1

    return byte1, byte2, length

{% endhighlight %}
Nothing fancy really. The only thing to discuss here is actually the byte handling. You see, when modern browsers
want to stream video today, they send a request to with a `Range` header, specifying the byte range which they want 
to receive. In Chromes, Firefoxes and (even) Edges case this looks like `Range: 0-` which just means "Give me everything
between first byte and infinity", leaving the server with the decision of closing the connection once there are no more
bytes to send, as well as deciding the byte chunk to send.  
Oh, one more thing to mention! When you query data from S3, it actually returns a byte stream object, making it super
efficient for this kind of application. If you want to get the whole data into your program if need to do a `.read()` on the
`media_stream`. As we can see here, Flask is well prepared for doing these kinds of services at top notch speed. We query
a byte chunk per time from S3 and send it on its way back to the browser.  

Ok, so now, what does Safari do? Well, as mentioned, Safari is a bit more cautious when it comes to accepting content.
When you want to stream video in Safari, it actually makes use of the `Range` header as it is supposed to be used (boring).
What this means is that Safari is always in control of how much content is coming back from the server, and if we want
to continue streaming etc.

{% highlight python %}
import os
import requests
import uuid
import boto3
from flask import Flask, jsonify, request, Response, stream_with_context
from werkzeug.datastructures import Headers
import json
import filetype
import re

bucket_name = 's3_media'
service = Flask(__name__)


@service.route('/<key>', methods=['GET'])
def get_specific_media(key):

    storage = boto3.client('s3')
    media_stream = storage.get_object(Bucket=bucket_name, Key=key)
    full_content = media_stream['ContentLength']
    headers = Headers()
    status = 200
    range_header = request.headers.get('Range', None)
    if range_header:
        byte_start, byte_end, length = get_byte_range(range_header)
        if byte_end:
            status = 206
            media_stream = storage.get_object(Bucket=BUCKET_NAME, Key=key, Range=f'bytes={byte_start}-{byte_end}')
            end = byte_start + length - 1
            headers.add('Content-Range', f'bytes {byte_start}-{end}/{full_content}')
            headers.add('Accept-Ranges', 'bytes')
            headers.add('Content-Transfer-Encoding', 'binary')
            headers.add('Connection', 'Keep-Alive')
            headers.add('Content-Type', content_type)
            if byte_end == 1:
                headers.add('Content-Length', '1')
            else:
                headers.add('Content-Length', media_stream['ContentLength'])

            response = Response(
                stream_with_context(media_stream['Body'].iter_chunks()),
                mimetype=content_type,
                content_type=content_type,
                headers=headers,
                status=status,
                direct_passthrough=True
            )
            return response

    headers.add('Content-Type', content_type)
    headers.add('Content-Length', media_stream['ContentLength'])
    response = Response(
        stream_with_context(media_stream['Body'].iter_chunks()),
        mimetype=content_type,
        content_type=content_type,
        headers=headers,
        status=status
        )
    return response


def get_byte_range(range_header):
    g = re.search('(\d+)-(\d*)', range_header).groups()
    byte1, byte2, length = 0, None, None
    if g[0]:
        byte1 = int(g[0])
    if g[1]:
        byte2 = int(g[1])
        length = byte2 + 1 - byte1

    return byte1, byte2, length

{% endhighlight %}
So what is going on here? Well, basically this is the conversation going on between Safari and the server when we stream
content:
1. Safari asks for byte range 0-1 (`Range: 0-1`)
2. Server responds with the first byte, the status 206 (partial content) and the total content length of the media
(`Content-Range: 0-1/252324`)
3. Safari then calculates how large byte chunks it wants per future request and continues with `Range: 0-1024` for example
4. Server responds with `0-1024/252324`
5. Safari asks for `Range: 1025-2048`
6. etc.

So basically this is a tuturial on how to handle Safari. Because the rest is a piece of cake. Glhf.
