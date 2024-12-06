---
title: Monitor Service With a Version Number or Instance Name using a Service Template
date: 2024-12-06 14:10
categories: [N-Central]
tags: [n-central]
---
Sometimes all you want is to simply apply a template and then all services you added in that template are monitored.
But what if you need to monitor a service on many servers which have an instance specific name, or even a version number added to the service name?

Simply put, just replace the Service Name where the version would be, with a star and remove everything behind it.
For example, if you want to monitor Apache Tomcat, the service name could be __Apache Tomcat 9.0 Tomcat9__ or it could be __Apache Tomcat 8.5 Tomcat8__
This example already has 2 different versions which would mean 2 different Services you need to add to you Service Template.
By just adding a single Windows Service in your Service Template and giving the Service Name as __Apache Tomcat*__ you would achieve the same as adding both services seperatly.