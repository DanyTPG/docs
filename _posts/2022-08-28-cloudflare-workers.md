---
title: cloudflare workers scripts
date: 2022-08-28 12:00:00 +0330
categories: [serverless, cloud functions]
tags: [cloudflare-workers]     ## TAG names should always be lowercase
---

Aggregated some scripts that I use on cloudflare workers.

## Reverse Proxy


```javascript
addEventListener(
  "fetch", event => {
    let url = new URL(event.request.url);
    url.host = "example.com";
    let request = new Request(url, event.request);
    event.respondWith(
      fetch(request)
    )
  }
)
```

## General Reverse Proxy with this format "/host/path" 

```javascript
export default {
  async fetch(req) {
    try {
      const url = new URL(req.url);
      const splitted = url.pathname.replace(/^\/*/, '').split('/');
      const address = splitted[0];
      url.pathname = splitted.slice(1).join('/');
      url.hostname = address;
      url.protocol = 'https';
      return fetch(new Request(url, req));
    } catch (e) {
      return new Response(e);
    }
  }
}
```
## Load Balance 301 Redirecting

```javascript
const statusCode = 301;
const urls = ["https://example1.com",
              "https://example2.com",
              "https://example3.com",
              "https://example4.com"
              ];

const random = Math.floor(Math.random() * urls.length);
const base = urls[random]
console.log(base);
async function handleRequest(request) {
  const url = new URL(request.url)
  const { pathname, search } = url

  const destinationURL = base + pathname + search

  return Response.redirect(destinationURL, statusCode)
}

addEventListener("fetch", async event => {
  event.respondWith(handleRequest(event.request))
})
```

## Get your IP Address

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {

  let content = request.headers.get('cf-connecting-ip') + '\n'

  return new Response(content, {
    headers: {
      "content-type": "application/json;charset=UTF-8",
    },})
}
```

## IP details by Cloudflare


```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  let content = "{" + '\n'

  content += '    ' + '"IP": "' + request.headers.get('cf-connecting-ip') + '",\n'
  content += '    ' + '"Organization": "' + request.cf.asOrganization + '",\n'
  content += '    ' + '"ASN": "' + request.cf.asn + '",\n'
  content += '    ' + '"Country": "' + request.cf.country + '",\n'
  content += '    ' + '"City": "' + request.cf.city + '",\n'
  content += '    ' + '"Continent": "' + request.cf.continent + '",\n'
  content += '    ' + '"Colo": "' + request.cf.colo + '",\n'
  content += '    ' + '"Latitude": "' + request.cf.latitude + '",\n'
  content += '    ' + '"Longitude": "' + request.cf.longitude + '",\n'
  content += '    ' + '"PostalCode": "' + request.cf.postalCode + '",\n'
  content += '    ' + '"MetroCode": "' + request.cf.metroCode + '",\n'
  content += '    ' + '"Region": "' + request.cf.region + '",\n'
  content += '    ' + '"RegionCode": "' + request.cf.regionCode + '",\n'
  content += '    ' + '"Timezone": "' + request.cf.timezone + '",\n'
  content += '    ' + '"Protocol": "' + request.cf.httpProtocol + '",\n'
  content += '    ' + '"tlsCipher": "' + request.cf.tlsCipher + '",\n'
  content += '    ' + '"tlsVersion": "' + request.cf.tlsVersion + '",\n'
  content += '    ' + '"Method": "' + request.method + '",\n'
  content += '    ' + '"Host": "' + request.headers.get('host') + '",\n'
  content += '    ' + '"UA": "' + request.headers.get('user-agent') + '",\n'
  content += '    ' + '"Language": "' + request.headers.get('accept-language') + '"\n'

  content += "}\n"
  return new Response(content, {
    headers: {
      "content-type": "application/json;charset=UTF-8",
    },})
}
```

## IP details by ipinfo.io

```javascript
async function handleRequest(request) {
  const clientIP = request.headers.get("cf-connecting-ip")
  console.log(clientIP)
  const url = new URL(request.url)
  let response = await fetch("https://ipinfo.io/" + clientIP + "?token=00000000000000") //enter your token instead of this
  let text = await response.text()
  text = text + "\n"
  console.log("THIS IS THE FUCKING LOG!! \n" + text)
  var init = { "status" : 200 , "statusText" : "OK" };
  var response2 = new Response(text,init);
  return response2
}

addEventListener("fetch", event => {
  event.respondWith(handleRequest(event.request))
})
```
## IP detail by ipinfo.io with query

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  const params = new URLSearchParams(url.search);
  const ip = params.get('ip');

  if (!ip) {
    // If no IP address is provided, return a form for the user to input an IP address
    const htmlOutput = `
      <style>
        body {
          font-family: sans-serif;
          background-color: #1d4158;
          text-align: center;
          padding: 30px;
          color: white;
        }
        form {
          display: inline-block;
          margin-bottom: 20px;
        }
        label {
          display: block;
          margin-bottom: 5px;
        }
        input[type="text"] {
          width: 250px;
          font-size: 16px;
          padding: 10px;
          border: 1px solid #ccc;
          border-radius: 4px;
          box-sizing: border-box;
        }
        button[type="submit"] {
          font-size: 16px;
          padding: 10px 20px;
          border: 1px solid #ccc;
          border-radius: 4px;
          background-color: #eee;
          cursor: pointer;
        }
        .result-container {
          display: none;
          white-space: nowrap;
          background-color: #112836;
          text-align: left;
          color: white;
          font-family: monospace;
          font-size: 16px;
          padding: 30px;
          margin-top: 100px;
          border: 1px solid #ccc;
          border-radius: 8px;
        }
        strong {
          display: inline-block;
          width: 120px;
          font-weight: bold;
        }
      </style>
      <form id="ipForm">
        <label>Enter an IP address:</label>
        <input id="ipInput" type="text" />
        <button type="submit">Submit</button>
      </form><br>
      <div id="resultContainer" class="result-container"></div>
      <script>
        document.getElementById('ipForm').addEventListener('submit', handleFormSubmit);
        
        function handleFormSubmit(event) {
          event.preventDefault();
          const ip = document.getElementById('ipInput').value;
          fetch('/?ip=' + ip)
            .then(response => response.text())
            .then(result => {
              const resultContainer = document.getElementById('resultContainer');
              resultContainer.innerHTML = result;
              resultContainer.style.display = 'inline-block'; // Show the result container
            })
            .catch(error => console.error(error));
        }
      </script>
    `;

    return new Response(htmlOutput, {
      headers: { 'Content-Type': 'text/html' }
    });
  } else {
    // If an IP address is provided, query the IPinfo.io API for information about the IP
    const response = await fetch(`https://ipinfo.io/${ip}?token=00000000000000`); // replace the token with your own
    const data = await response.json();

    // Format the IP information in an HTML response
    let htmlOutput = `
      <h1>IP Information for ${ip}</h1>
    `;
    htmlOutput += `<p><strong>Hostname:</strong> ${data.hostname}</p>`;
    htmlOutput += `<p><strong>Country:</strong> ${data.country}</p>`;
    htmlOutput += `<p><strong>Region:</strong> ${data.region}</p>`;
    htmlOutput += `<p><strong>City:</strong> ${data.city}</p>`;
    htmlOutput += `<p><strong>Location:</strong> ${data.loc}</p>`;
    htmlOutput += `<p><strong>Organization:</strong> ${data.org}</p>`;
    htmlOutput += `<p><strong>Postal Code:</strong> ${data.postal}</p>`;
    
    

    return new Response(htmlOutput, {
      headers: { 'Content-Type': 'text/html' }
    });
  }
}
```

## DNS over HTTPS handler

```javascript
addEventListener('fetch', function(event) {
    const { request } = event
    const response = handleRequest(request)
    event.respondWith(response)
})

async function handleRequest(request) {
    const doh = 'https://cloudflare-dns.com/dns-query'
    const contype = 'application/dns-message'
    const { method, headers, url } = request
    const { host, searchParams } = new URL(url)
    if (method == 'GET' && searchParams.has('dns')) {
        return await fetch(doh + '?dns=' + searchParams.get('dns'), {
            method: 'GET',
            headers: {
                'Accept': contype,
            }
        });
    } else if (method == 'POST' && headers.get('content-type')=='application/dns-message') {
        return await fetch(doh, {
            method: 'POST',
            headers: {
                'Accept': contype,
                'Content-Type': contype,
            },
            body: await request.arrayBuffer()
        });
    } else {
        return new Response("", {status: 404})
    }
}
```
