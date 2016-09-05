# IBM API Connect - GatewayScript Code example: Transform application/json to multipart/form-data

IBM API Connect [GatewayScripts](http://www.ibm.com/support/knowledgecenter/SSMNED_5.0.0/com.ibm.apic.toolkit.doc/rapim_gwscript_codesnip.html) are a great possibility to transform a message up to everyones needs within a API Assembly. Unfortunately the GatewayScript API is not documented very well yet. So just to get some ideas how it works: The following example shows how to transform an `application/json` message body to a `multipart/form-data` message.

---
![Assembly Screenshot](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-09-06_gatewayscript.png)

```javascript
var boundary = "--VFCPlauenIstSuper";
var crlf = String.fromCharCode(13) + String.fromCharCode(10);

var json = apim.getvariable('message.body');
var keys = Object.keys(json);
var output = '';

for (var i = 0; i < keys.length; i++) {
  output += "--" + boundary + crlf;
  output += 'Content-Disposition: form-data; name="' + keys[i] + '"' + crlf + crlf;
  output += json[keys[i]] + crlf;
}

console.alert("this should end up in DataPower system log at alert level");

output += "--" + boundary + "--";

apim.setvariable('message.body', output);
apim.output("multipart/form-data; boundary=" + boundary);
```

Some notes on that code snippet:

* `var crlf = String.fromCharCode(13) + String.fromCharCode(10)` is a replacement for `var crlf = "\r\n"` which won't work. During my tests I saw often problems when using escape characters in GatewayScript.

* `apim.setvariable('message.body', output)` sets the message body as string. When using a string as body the Invoke policy didn't work for me - I replaced it with a Proxy policy.

* `apim.output("multipart/form-data; boundary=" + boundary)` sets the message's content type.
