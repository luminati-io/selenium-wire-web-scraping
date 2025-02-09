# Web Scraping With Selenium Wire in Python

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/) 

This guide explains how to use Selenium Wire for web scraping and covers topics like request interception and dynamic proxy rotation.

- [What Is Selenium Wire?](#what-is-selenium-wire)
- [Why Use Selenium Wire?](#why-use-selenium-wire)
- [Key Features of Selenium Wire](#key-features-of-selenium-wire)
    - [Access Requests and Responses](#access-requests-and-responses)
    - [Intercept Requests and Responses](#intercept-requests-and-responses)
    - [WebSocket Monitoring](#websocket-monitoring)
    - [Manage Proxies](#manage-proxies)
- [Proxy Rotation in Selenium Wire](#proxy-rotation-in-selenium-wire)
    - [Requirements](#requirements)
    - [Step 1: Randomize Proxies](#step-1-randomize-proxies)
    - [Step 2: Set the Proxy](#step-2-set-the-proxy)
    - [Step 3: Visit the Target Page](#step-3-visit-the-target-page)
    - [Step 4: Put It All Together](#step-4-put-it-all-together)
- [Rotating proxies with Bright Data Proxies](#a-better-approach-to-proxy-rotation-bright-data-proxies)
- [Selenium vs Selenium Wire for Web Scraping](#selenium-vs-selenium-wire-for-web-scraping)
- [Conclusion](#conclusion)

## What Is Selenium Wire?

[Selenium Wire](https://github.com/wkeeling/selenium-wire) is an extension for Selenium’s Python bindings that provides control over browser requests. It allows intercepting and modifying both requests and responses in real time directly from your Python code while using Selenium.

> **Note**:\
> The library is no longer maintained, however, several scraping technologies and scripts still use it.

## Why Use Selenium Wire?

Browsers have certain limitations that can make web scraping challenging. For example, they do not enable you to set authorized proxy URLs or [rotate proxies](/solutions/rotating-proxies) on the fly. Selenium Wire helps you overcome those limitations by interacting with sites as regular human users would.

Here are some of the reasones why you should use Selenium Wire for web scraping:

- **Gain Direct Access to Network Traffic**: Analyze, monitor, and modify AJAX requests and responses to extract valuable data efficiently.
- **Evade Anti-Bot Detection**: [`ChromeDriver`](https://developer.chrome.com/docs/chromedriver/downloads?hl=en) reveals identifiable details that anti-bot systems use for detection. Technologies like `undetected-chromedriver` leverage Selenium Wire to mask these details and bypass detection mechanisms.
- **Enhance Browser Flexibility**: Traditional browsers rely on fixed startup configurations that require a restart to modify. Selenium Wire enables real-time updates to request headers and proxy settings within an active session, making it an optimal solution for dynamic web scraping.

## Key Features of Selenium Wire

### Access Requests and Responses

Selenium Wire allows you to monitor and capture HTTP/HTTPS traffic from the browser, providing access to the following key attributes:

| **Attribute** | **Description** |
| --- | --- |
| `driver.requests` | It reports the list of captured requests in chronological order |
| `driver.last_request` | It reports the most recently captured request  <br>(This is more efficient than using `driver.requests[-1]`) |
| `driver.wait_for_request(pat, timeout=10)` | This method will wait—the time is defined by the `timeout` parameter—until it sees a request matching a pattern, defined by the `pat` parameter—which can be a substring or a [regular expression](/blog/web-data/web-scraping-with-regex). |
| `driver.har` | A JSON formatted [HAR](https://docs.brightdata.com/api-reference/proxy-manager/get_har_logs) archive of HTTP transactions that have taken place. |
| `driver.iter_requests()` | It returns an iterator over captured requests. |

A Selenium Wire `Request` object has the following attributes:

| **Attribute** | **Description** |
| --- | --- |
| `body` | The body’s request is presented as `bytes`. If the request has no body the value of `body` will be empty (for example: `b''`). |
| `cert` | It reports information about the server SSL certificate in a dictionary format (it’s empty for non-HTTPS requests). |
| `date` | It shows the datetime at which the request was made. |
| `headers` | It reports a dictionary-like object of the request’s headers (note that in Selenium Wire headers are case-insensitive and duplicates are permitted). |
| `host` | It reports the request host ( for example, `https://brightdata.com/`). |
| `method` | It specifies the HHTP method (`GET`, `POST`, etc…) |
| `params` | It reports a dictionary of the request’s parameters (note that if a parameter with the same name appears more than once in the request, its value in the dictionary will be a list). |
| `path` | It reports the request path. |
| `querystring` | It reports the query string. |
| `response` | It reports the response object associated with the request (note that the value will be `None` if the request has no response). |
| `url` | It reports the request URL complete with `host`, `path`, and `querystring`. |
| `ws_messages` | In the case a request is a WebSocket (in which case, the URL is generally like `wss://`) the `ws_messages` will contain any websocket messages sent and received. |

Instead, a `Response` object exposes these attributes:

| **Attribute** | **Description** |
| --- | --- |
| `body` | The body’s response is presented as `bytes`. If the response has no body the value of `body` will be empty (for example: `b''`). |
| `date` | It shows the datetime at which the response was received. |
| `headers` | It reports a dictionary-like object of the response’s headers (note that in Selenium Wire headers are case-insensitive and duplicates are permitted). |
| `reason` | It reports the reason phrase of the response, like `OK`, `Not Found`, etc… |
| `status_code` | It reports the status of the response, like `200`, `404`, etc… |

Let's write a Python script to test this feature:

```python
from seleniumwire import webdriver

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

try:
    # Open the target website
    driver.get("https://brightdata.com/")

    # Access and print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Method: {request.method}")
        print(f"Headers: {request.headers}")
        print(f"Response Status Code: {request.response.status_code if request.response else 'No Response'}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

This code opens the target website and capture requests by using `driver.requests`. Then, it loops through a for loop to intercept some request attributes like `url`, `method`, and `headers`.

Here is the expected result:

![Some of the logged requests](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-98-1024x597.png)

### Intercept Requests and Responses


Selenium Wire enables interception and modification of requests and responses using interceptors—functions that are triggered as network traffic flows through the browser.

There are two separate interceptors:

* `driver.request_interceptor`: intercepts requests and accepts a single argument.
* `driver.response_interceptor`: intercepts the response and accepts two arguments, one for the originating request and one for the response.

Here is an example that shows how to use a request interceptor:

```python
from seleniumwire import webdriver

# Define the request interceptor function
def interceptor(request):
    # Add a custom header to all requests
    request.headers["X-Test-Header"] = "MyCustomHeaderValue"

    # Block requests to a specific domain
    if "example.com" in request.url:
        print(f"Blocking request to: {request.url}")
        request.abort()  # Abort the request

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

# Assign the interceptor function to the driver
driver.request_interceptor = interceptor

try:
    # Open a website that makes multiple requests
    driver.get("https://brightdata.com/")

    # Print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Headers: {request.headers}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

This is what this snippet does:

* **Interceptor function**: Creates an interceptor function to be called for every outgoing request. This adds a custom header to all outgoing requests with `request.headers[]`. Also, it blocks browser requests for `example.com` domain.
* **Captures requests**: After the page is loaded, all captured requests are printed, including the modified headers.

> **Note:**\
>  Blocking requests is beneficial when pages load extra resources like ads, analytics scripts, or third-party widgets that are not essential to your task. This approach enhances scraping efficiency by increasing speed and minimizing bandwidth consumption.

The expected result should be something like this:

![Note the X-Test-Header](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-99-1024x538.png)

### WebSocket Monitoring

Many modern web pages use [`WebSockets`](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) for real-time communication with servers. `WebSockets` establish a persistent connection between the server and the browser so that the data can be exchanged continuously without the overhead of traditional HTTP requests.

Critical data frequently passes through these channels, making direct access essential for efficient data retrieval. By intercepting `WebSocket` communication, you can capture raw server-sent data in real time, bypassing the need for browser processing or page rendering.

Here are the attributes of a Selenium Wire `WebSocket` object:

| **Attribute** | **Description** |
| --- | --- |
| `content` | It reports the message’s content which can be either a `str` or in the `bytes` format. |
| `date` | It shows the datetime of the message. |
| `headers` | It reports a dictionary-like object of the response’s headers (note that in Selenium Wire headers are case-insensitive and duplicates are permitted). |
| `from_client` | This is a boolean that returns `True` when the message was sent by the client and `False` by the server. |

### Manage Proxies

Proxy servers function as intermediaries between your device and target websites, concealing your IP address. They facilitate bypassing IP-based restrictions, mitigate blocking due to rate limits, and enable access to geo-restricted content for seamless web scraping.

Let's configure a proxy in Selenium Wire:

```python
# Set up Selenium Wire options
options = {
    "proxy": {
        "http": "<YOUR_HTTP_PROXY_URL>",
        "https": "<YOUR_HTTPS_PROXY_URL>"
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

This setup differs from configuring a proxy in vanilla Selenium, where you need to rely on Chrome’s `--proxy-server` flag. This means that proxy configuration is static in vanilla Selenium. After a proxy has been set, it remains in effect for the entire browser session and cannot be modified without restarting the browser. This restriction can be limiting, particularly when dynamic proxy rotation is required.

In contrast, Selenium Wire provides the flexibility to change proxies dynamically within the same browser instance. That is possible thanks to the `proxy` attribute:

```python
# Dynamically change the proxy
driver.proxy = {
    "http": "<NEW_HTTP_PROXY_URL>",
    "https": "<NEW_HTTPS_PROXY_URL>"
}
```

Plus, Chrome’s `--proxy-server` flag does not support proxies with authentication credentials in the URL:

```
protocol://username:password@host:port
```

Instead, Selenium Wire fully supports authenticated proxies, making it the better choice for web scraping.

## Proxy Rotation in Selenium Wire

Let's set up a Selenium Wire project for proxy rotation. This will help you make your exit IP change at every request.

### Requirements

You need the following prerequisites to follow this part of the guide:

* Python 3.7 or higher
* [Supported web browser](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/driver_location/)

Start with creating a virtual environment directory:

```bash
python -m venv venv
```

To activate it, on Windows, run:

```bash
venv\Scripts\activate
```

On macOS/Linux, execute:

```bash
source venv/bin/activate
```

Now  install Selenium Wire (Selenium will be automatically installed as its dependency):

```bash
pip install selenium-wire
```

### Step 1: Randomize Proxies

First, you need a list of valid proxy URLs. You can use our list of [free proxies](https://brightdata.com/solutions/free-proxies). Add them to a list and use [`random.choice()`](https://docs.python.org/3/library/random.html#random.choice) to pick a random element from it:

```python
def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # ...
    ]
    
    # Randomize the list
    return random.choice(proxies)
```

Once called, this function returns a random proxy URL from the list.

To make it work, do not forget to import `random`:

```
import random
```

### Step 2: Set the Proxy

Call the `get_random_proxy()` function to get a proxy URL:

```python
proxy = get_random_proxy()
```

Initialize the browser instance and set the selected proxy:

```python
# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)
```

The above snippet requires the following imports:

```python
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
```

For dynamically changing the proxy during the browser session, use this code instead:

```python
driver.proxy = {
    "http": proxy,
    "https": proxy
}
```

### Step 3: Visit the Target Page

Visit the target website, extract the output, and close the browser:

```python
try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

To make it work, import `By` from Selenium:

```python
from selenium.webdriver.common.by import By
```

In this example, the destination page is the [`/ip`](https://httpbin.io/ip) endpoint from the HTTPBin project: this page returns the IP address of the caller. If everything goes as expected, the script should print a different IP from the list of proxies on each run.

### Step 4: Put It All Together

This is the entire Selenium Wire proxy rotation logic that should be in your `selenium_wire.py` file:

```python
import random
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By

def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # Add more proxies here...
    ]
    
    # Randomly pick a proxy
    return random.choice(proxies)
 
# Pick a random proxy URL 
proxy = get_random_proxy()

# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)

try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

To run the file, launch:

```bash
python3 selenium_wire.py
```

At each run, the output should be:

```json
{
  "origin": "PROXY_1:XXXX"
}
```

Or:

```json
{
  "origin": "PROXY_2:YYYY"
}
```

And so on…

Run the script multiple times, and you will see a different IP address each time.

## A Better Approach to Proxy Rotation: Bright Data Proxies

Manual proxy rotation in Selenium Wire involves a lot of boilerplate code and requires maintaining a list of valid proxy URLs. Instead, you can use Bright Data’s rotating proxies that automatically handle IP address changes. Here is how you can use them.

If you already have an account, log in to Bright Data. Otherwise, create an account for free. You will gain access to the following user dashboard:

![The Bright Data dashboard](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-100-1024x498.png)

Click the “View proxy products” button:

![View proxy products](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-101.png)

You will be redirected to the “Proxies & Scraping Infrastructure” page below:

![Configuring your residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-102-1024x483.png)

Scroll down, find the “[Residential Proxies](/blog/proxy-101/ultimate-guide-to-proxy-types)” card, and click on the “Get started” button:

![Residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-103.png)

You will reach the residential proxy configuration dashboard. Follow the guided wizard and set up the proxy service based on your needs.

![Configuring your residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-104.png)

Go to the “Access parameters” tab and retrieve your proxy’s host, port, username, and password as follows:

![access parameter](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-105.png)

Note that the “Host” field already includes the port.

That is all you need to build the proxy URL and set it in Selenium Wire. Collect all the information and build a URL with the following syntax:

```
<username>:<password>@<host>
```

For example, in this case it would be:

```
brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX
```

Toggle “Active proxy,” follow the last instructions, and you are good to go!

![Active proxy toggle](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-106-1024x164.png)

Here is the Selenium Wire proxy snippet for Bright Data integration:

```python
# Bright Data proxy URL
proxy = "brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX"

# Set up Selenium Wire options
options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

## Selenium vs Selenium Wire for Web Scraping

To summarize, here is the Selenium vs Selenium Wire comparison:

|     | **Selenium** | **Selenium Wire** |
| --- | --- | --- |
| **Purpose** | Automates web browsers to perform UI testing and web interactions | Extends Selenium to provide additional capabilities for inspecting and modifying HTTP/HTTPS requests and responses |
| **HTTP/HTTPS request handling** | Does not provide direct access to HTTP/HTTPS requests or responses | Allows inspection, modification, and capturing of HTTP/HTTPS requests and responses |
| **Proxy support** | Has limited proxy support (requires manual configuration) | Advanced proxy management, with support for dynamic setting |
| **Performance** | Lightweight and fast | Slightly slower due to the capturing and processing of the network traffic |
| **Use cases** | Primarily used for functional testing of web applications, handy for basic web scraping cases | Useful for testing APIs, debugging network traffic, and web scraping |

## Conclusion

While Selenium Wire can be used for web scraping efficiently, it isn't maintained software and is not a one-size-fits-all solution.

Instead, consider using vanilla Selenium with a dedicated scraping browser like the [Scraping Browser from Bright Data](https://brightdata.com/products/scraping-browser). It's a scalable cloud browser that works with Playwright, Puppeteer, Selenium, and others. It seamlessly rotates exit IPs for each request while managing browser fingerprinting, retries, CAPTCHA solving, and more. Try it to eliminate blocking issues and optimize your scraping workflow.
