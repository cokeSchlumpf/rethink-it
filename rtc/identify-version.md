# Identify version of an RTC instance programmatically

When running customized plugins or extensions for several versions of [IBM Rational Team Concert](http://www-03.ibm.com/software/products/de/rtc) it might be necessary to identify the version running on the server to select the right client API.

---

In one of my current projects we have an extension for RTC running based on RTC's Client Java API. Since the Java Client API must fit to the RTC server version we've actually running multiple instances of the extension. Thus at one point of our flow we need to decide which instance of the extension need to be contacted.

To get the version information of a running RTC the following request can be used:

```bash
curl \
  -H "Accept: text/plain" \
  -u ${USERNAME}:${PASSWORD} \
  https://${HOSTNAME}/${CCM_PATH}/service/com.ibm.team.repository.service.internal.IProductRegistryRestService/allProductInfo
```

The response will be something like:

```json
{
   "soapenv:Body":{
      "response":{
         "returnValue":{
            "values":[
               {
                  "version":"6.0.2",
                  "buildId":"RJF-SERVER-I20160322-2253",
                  "name":"Jazz Foundation - Core Libraries",
                  "id":"com.ibm.team.jazz.foundation.server",
                  "_eQualifiedClassName":"http:///repository_DTO/dto.ecore:ProductInfoDTO"
               },
               {
                  "version":"6.0.2",
                  "buildId":"RTC-SERVER-I20160323-2215",
                  "name":"Change and Configuration Management - Core Libraries",
                  "id":"com.ibm.team.rtc.web.product",
                  "_eQualifiedClassName":"http:///repository_DTO/dto.ecore:ProductInfoDTO"
               }
            ],
            "type":"COMPLEX",
            "_eQualifiedClassName":"http:///com/ibm/team/core/services.ecore:ComplexArrayDataArg"
         },
         "method":"getAllProductInfo",
         "interface":"com.ibm.team.repository.service.internal.IProductRegistryRestService",
         "_eQualifiedClassName":"http:///com/ibm/team/core/services.ecore:Response"
      },
      "_eQualifiedClassName":"http://schemas.xmlsoap.org/soap/envelope/:Body"
   },
   "_eQualifiedClassName":"http://schemas.xmlsoap.org/soap/envelope/:Envelope"
}
```

Parsing this JSON is simple and the request can be routed to the right extension.
