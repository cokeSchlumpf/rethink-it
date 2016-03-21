In my current project I'm using the SOAPAsyncRequest Node to call a backend WebService. I was struggling a few hours to find out how I could transfer user context data from the request to the response flow. 

A look to the [specification](http://www-01.ibm.com/support/knowledgecenter/SSMKHH_9.0.0/com.ibm.etools.mft.doc/ac56202_.htm) tells us that there is a property `UserContext` we can use to transfer context data. But one important information is missing: Only `BLOB` values are allowed (at least the error message tells you if you set anything else). 

After I found out that restriction the big fight begun: How do I cast my data structure to a `BLOB` value and vice versa? After a lot of googling and playing around with ESQL I found the following solution:

To set the UserContext you could do something like that:

```prettyprint lang-sql
SET EVarRef.UserContext.ReplyToQ = InputRoot.MQMD.ReplyToQ;
SET EVarRef.UserContext.MessageId = InputRoot.MQMD.MsgId;

CREATE LASTCHILD OF EVarRef.UserContext DOMAIN('XMLNSC') NAME 'XMLNSC';
SET EVarRef.UserContext.XMLNSC.UserContext.ReplyToQ = InputRoot.MQMD.ReplyToQ;
SET EVarRef.UserContext.XMLNSC.UserContext.MessageId = InputRoot.MQMD.MsgId;
SET EVarRef.UserContext.BLOB = ASBITSTREAM(EVarRef.UserContext.XMLNSC.UserContext OPTIONS FolderBitStream);

SET OutputLocalEnvironment.Destination.SOAP.Request.UserContext = EVarRef.UserContext.BLOB;
```

To get back the UserContext into your environment within your SOAPAsyncResponse-Flow you can do that:

```prettyprint lang-sql
CREATE LASTCHILD OF EVarRef.UserContext DOMAIN('XMLNSC') NAME 'XMLNSC';
DECLARE ptrBitstream REFERENCE TO InputLocalEnvironment.SOAP.Response.UserContext;

WHILE LASTMOVE(ptrBitstream) DO
   --Parse the BLOB into XML using the PARSE function
   CREATE LASTCHILD OF EVarRef.UserContext.XMLNSC PARSE(CAST(ptrBitstream AS BLOB) OPTIONS FolderBitStream CCSID 1208 FORMAT 'XMLNSC');
   MOVE ptrBitstream NEXTSIBLING;
END WHILE; 
```

I hope these code samples help you if you want to do something similar.


