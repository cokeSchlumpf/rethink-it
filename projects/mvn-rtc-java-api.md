# Setting up RTC Java Client API for Maven projects

On [github](https://github.com/cokeSchlumpf/mvn-rtc-java-api) I just pushed a small script which helps to set up Maven projects with the RTC Java Client API.

---

**How does it work?** [Tiago Moura's](https://www.ibm.com/developerworks/community/blogs/cbe857dd-5392-4111-b0ea-6827c54f2e66/entry/setting_up_rtc_java_plain_api_dev_enviroment_with_maven_and_eclipse?lang=en) blog post on IBM developerWorks explains two options to integrate RTC Client Java API's dependencies with Maven:

* Install the libs into the local Maven repository `~/.m2/repository` or
* install them in a company managed repository

... where the 2nd option would be the perfect situation. But what to do in case that no company managed repository exists and the RTC client application should be developed by multiple developers or should be build on a continuous integration environment? In that case an other option would be the best choice:

* Set up a repository within the project as described by [Charlie Wu](http://charlie.cu.cc/2012/06/how-add-external-libraries-maven/).

`setup-repo.sh` automates the set up of a project repository including all the necessary dependencies of the RTC Client Java API. The script iterates through all library files included in the RTC Java Client API [download](http://ca-toronto-dl02.jazz.net/mirror/downloads/rational-team-concert/6.0.2/6.0.2/RTC-Client-plainJavaLib-6.0.2.zip) and collects all necessary information (groupIds, artifactIds, versions) to put them into a Maven repository. Finally it creates a POM artifact which includes all dependencies so that the project itself just needs to add one dependency:

```xml
<dependency>
   <groupId>com.ibm.rtc</groupId>
   <artifactId>rtc-java-api</artifactId>
   <version>${VERSION}</version>
   <type>pom</type>
</dependency>
```
