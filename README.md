# Server Side Request Forgery (SSRF) Checklist


- [ ] Ensure that IP deny lists are comprehensive.
- [ ] Enforce only defined protocols are processed (E.g. only HTTP/HTTPS)
- [ ] Ensure that DNS name resolutions don’t bypass any checks
- [ ] Ensure that HTTP redirects don’t bypass any checks
- [ ] Ensure that the IPv6 addresses don’t bypass any checks
- [ ] Ensure that uncommon representations of IPs don’t bypass any checks Ensure user controlled data is the expected type
- [ ] Ensure abusing URL parsing doesn’t bypass checks
- [ ] Ensure CR-LF & Unicode characters get encoded correctly
- [ ] Limit user input that is used to create Server side requests to the minimum required
