---
layout: post
title: Repurposing an ESP8266 as a Stock Ticker
comments: true
---

A while back a good buddy of mine gave me a pretty sweet little arduino. It’s
purpose was to reach out to the osquery github periodically and get the number
of open pull requests and issues we had open, which was pretty relevant to me
for some time.

![Original ESP8266](/images/blog/mitchell_arduino_cropped.jpg)

Lately though I’ve been wondering what else I could do with this little device,
and being relatively new to the world of Arduino, I decided to set out with a
specific goal in mind. I’ve been pretty interested in various stock prices
lately, so I figured I’d see if I could get this lil’ guy acting like a stock
ticker as the board came with an Arduino feather OLED. Overall this isn’t too
challenging, there’s plenty of tutorials out there for leveraging an ESP8266 to
get a remote resource and then display this to the OLED, however I had a lot of
trouble tracking down good examples of interfacing with an API over HTTPS. The
examples definitely exist, but I figured I’d make my code public as it took me
some time to aggregate all the pieces needed. In particular there’s an entire
SSL library, BearSSL, that makes interacting with HTTPS endpoints pretty simple,
all you need is the Sha1 fingerprint of your remote resource and the HTTPClient
class handles the rest.

```cpp
  Serial.println("Fetching resource from " + uri);
  std::unique_ptr<BearSSL::WiFiClientSecure>secure_client(new BearSSL::WiFiClientSecure);
  
  secure_client->setFingerprint(kAlphaVantageFingerprint);
  secure_client->setTimeout(3000);

  HTTPClient https;
  auto ret = https.begin(*secure_client, uri);

  if (!ret) {
    Serial.println("Failed to connect to URI with " + String(ret));
    return 1;
  }
  
  auto status = https.GET();
  if (status != HTTP_CODE_OK) {
    Serial.println("HTTPS GET failed with code " + String(status));
    return status;
  }

  resp = https.getString();
  return status;
```

While it wasn’t too difficult getting the Arduino to fetch the resources, I had
a bit of trouble finding a good financial API. Turns out they’re all super
interested in making money or something. I eventually stumbled across Alpha
Vantage, which has been pretty decent, unfortunately their API limits aren’t
great at 5 calls/minute, 500 requests/day. This wouldn’t be a huge issue if I
could batch up all of the ticker symbols I wanted to fetch summaries for at
once, but I haven’t been able to figure out how that part works, so for now my
stock data is pretty stale, and I eventually exhaust my API counts and just
display nulls :\

While I was moving to the Great Pacific Northwest, the structure of the 3d
printing on the original board broke, so I had to improvise a bit. Here’s
where I ended up:

![New Stock Ticker](/images/blog/stock_ticker_1.jpg "Arduinio Stock Ticker")

![That solder job tho.](/images/blog/stock_ticker_2.jpg "Ticker solder job")

Overall it was a pretty great entry project to Arduino grabbing content from a
remote endpoint over a TLS connection.
[Here’s a link to the code base](https://github.com/muffins/arduino/blob/master/stock_ticker/stock_ticker.ino)
in case anyone else is trying to hit HTTPS endpoints with an ESP 8266 on
Arduino. Really this project was just setting the stage for my next post,
where I built an arduinio to take the temperature and post the data to a remote
SIEM, more to come soon, Happy hacking!
