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
- [ ] Limit user input that is used to create Server side requests to the minimum required

:warning: If I have missed a vector, feel free to open an issue or PR.

### Ensure that IP deny lists are comprehensive

There's probably not a legitimate use case for a user to have a server request something in the private IP space, the loopback address, or the AWS metadata endpoint. Ensure that you have all the following IPs in your disallow list.

|   |   |
|---|---|
| 127.0.0.1/8 | Loopback |
| 10.0.0.0/8 | Private IP Address Block |
| 172.16.0.0/12 | Private IP Address Block |
| 192.168.0.0/16 | Private IP Address Block |
| 169.254.169.254/32 | AWS Metadata Endpoint |

### Enforce only defined protocols are processed (E.g. only HTTP/HTTPS)

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
