---
layout: post
title:  "BIRT Reporting"
date:   2019-01-02 14:34:26 +0300
categories: BIRT spring swagger
---

### Spring with Swagger Integration

The real problem you'll encounter will be integrating with spring with swagger. I expect you're using swagger for your microservice, because it'll make you easily test your microservice REST API. Unfortunately,  there are two compatibility issues with the current (4.8.0) version of the BIRT with spring/swagger. The two packages `com.google.common`and `org.slf4j` are put into the jar package of the BIRT, instead of being a dependency. The versions in the BIRT jar are not compatible with spring's current version which uses `net.logstash.logback`.  And it also not compatible with `springfox-spring-web`(2.9.2) version which uses `google.common` package.

The easiest and a working way of solving this is to open the BIRT jar file with some archive editing software (ex. winrar) and deleting those directories from the jar file. That way, BIRT will use the ones which come with spring and swagger.

This workaround needs another step which makes your service uses the modified jar file instead of the one that comes with gradle/maven. Your dependencies section in `build.gradle`file (or which gradle file you use) should look similar to this:

```java
dependencies {
   compile springBoot
   compile springCloud
   compile swagger
   compile logs
   compile fileTree(dir: "birt_runtime", includes: ['*.jar'])   
}
```

Please notice that I'm using all the files in that folder (which includes all the [BIRT runtime jar files](https://download.eclipse.org/birt/downloads/)), because as we're using the jar file, the gradle system will not be resolving dependencies for it. So, we're putting all the necessary files (which will also be needed in the runtime while creating the report )

If you are also using `docker` for deployment, you should also not forget to add necessary steps to copy files to the docker image. A similar step is should look like:

`COPY ./birt_runtime/*.jar ./birt_runtime/`

And ( :) ) do not forget to somehow map that directory to your `EngineConfig` instance. To achieve that you can easily create an environment variable (as `BIRT_HOME`) and pass it to the `EngineConfig`instance:

```java
EngineConfig config = new EngineConfig();
config.setBIRTHome(System.getenv("BIRT_HOME"));
```

Or you can use a static directory (like `"/birt_runtime"`).

### Getting Data From a REST API Data Source

BIRT doesn't support REST API Data Sources out of box. But you can get data from such endpoint using scripting. You use an `InputStreamReader` and a `BufferedReader` which gets data from the URL. After you get all data and accumulate into a `String`:

```javascript
var inStream = new URL("http://pathtomyservice/restpath").openStream();

var inStreamReader = new InputStreamReader(inStream);
var bufferedReader = new BufferedReader(inStreamReader);
var line;
var result = "";

while ((line = bufferedReader.readLine()) != null)
{
	result += line;
}
inStream.close();
```

After that, you use `JSON.parse` method to parse the JSON string and have the JSON object. You can put the resulting object into an object and use it report-wide:

`jsonObject = JSON.parse(result);`

For all these work, you'll also need to import packages in the beginning:

```javascript
importPackage(Packages.java.io);
importPackage(Packages.java.net);
importPackage(Packages.org.apache.commons.io);
```

### Files Related to Report

The report file may need some resources nearby it (like pictures, etc.). You have three options:

1. Embed resources into the report
2. Have the images as files in somehow nearby the report
3. Have the images loaded from a data source

If you are using docker, you generally compile your entire application/service into one single jar and copy the jar to the image. For me, I couldn't make the 1. option run with a single jar file. I chose to use the 2. option which is easy, indeed:

* I've set an environment variable like `MY_REPORTS_DIR` 

* I've put my reports as folders in that folder.

* I used that env. variable in my code for getting the report design file:

  ```java
  Resource salesReportResource = new FileSystemResource(System.getenv("MY_REPORTS_DIR") + "/salesreport/sales_report.rptdesign");
  IReportRunnable design = engine.openReportDesign( salesReportResource.getInputStream() );
  ```

### Fonts

Be careful about the font you use. Be sure that the font you use in the report is installed on the nodes. For ex. if you are developing on windows but the microservice you're creating (as usual) will run on a linux machine. The so common Arial font is not available on linux, so choose Helvetica instead. You can install [msttcorefonts](apt:msttcorefonts) if you really want to have Microsoft fonts. Or you can utilize some .css files and use `font-family` [property](https://www.w3schools.com/csSref/pr_font_font-family.asp) like web developers do, which will make the report choose among available fonts you've specified.