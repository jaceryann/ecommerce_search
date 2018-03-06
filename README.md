## Introduction

The goal for this project is to take a specified list of search terms, use each one in a basic search on [Amazon](https://www.amazon.com/), and pull key data (product name, price, rating) from the resulting page. This GitHub Page will walk through each of the steps required in detail, addressing ways to capture the key data when web design differences may apply.

### GET html

The first step is to GET the web page HTML by using the API of a PoolManager (using **urllib3**). As recommended by the **urllib3** [documentation](https://urllib3.readthedocs.io/en/latest/user-guide.html#ssl), SSL certificate verification should be used when making HTTPS requests. Mozilla's root certificate bundle is available via the **certifi** package.

```python
import certifi
import urllib3

http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', 
                           ca_certs=certifi.where())
```
