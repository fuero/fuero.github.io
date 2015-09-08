---
layout: post
title: Shibboleth SP, Tomcat and Charset oddities
permalink: shibboleth-tomcat-charset-oddities
comments: true

---

## The problem

When passing an attribute to Tomcat via HTTP or AJP, I noticed that Tomcat treats it as ISO-8859-1.
No matter what I did fiddling with Apache's config settings, it didn't matter.
Shibboleth SP's [docs][sp-attribute-access] mandate treating the data as UTF-8.

Digging through tomcat's source code, I found this in [ByteChunk.java][tc-bytechunk]:

{% highlight java %}
/** Default encoding used to convert to strings. It should be UTF8,
    as most standards seem to converge, but the servlet API requires
    8859_1, and this object is used mostly for servlets.
*/
public static final Charset DEFAULT_CHARSET = B2CConverter.ISO_8859_1;
{% endhighlight %}

Judging from the code where HTTP headers/AJP attributes are fed into the `ServletRequest` object, no
attempt is made to pass any encoding information about them to `ByteChunk` - thus defaulting to ISO.

## Reproducing the problem

The [Tomcat FAQ][tc-faq] has an entry on this topic, but as this example shows, it doesn't matter for the
interpretation of HTTP headers or AJP attributes.

- Download tomcat 8.x, unzip and change to its directory.
- mkdir -p webapps/foo/WEB-INF
- create web.xml with this content:

{% highlight xml %}
<web-app>
  <filter>
    <filter-name>encoding-filter</filter-name>
    <filter-class>org.apache.catalina.filters.SetCharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>encoding-filter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
</web-app>
{% endhighlight %}

- create index.jsp:

{% highlight jsp %}
<%--
  Adapted 2015 by fuero
  original Copyright (c) 2002 by Phil Hanna
  All rights reserved.
  
  You may study, use, modify, and distribute this
  software for any purpose provided that this
  copyright notice appears in all copies.
  
  This software is provided without warranty
  either expressed or implied.
--%>
<%@ page contentType="text/html; charset=UTF-8" %>
<%@ page import="java.util.*" %>
<html>
   <head>
      <title>Echo</title>
   </head>
   <body>
      <h1>HTTP Request Headers Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      <%
         Enumeration eNames = request.getHeaderNames();
         while (eNames.hasMoreElements()) {
            String name = (String) eNames.nextElement();
            String value = normalize(request.getHeader(name));
      %>
         <tr><td><%= name %></td><td><%= value %></td></tr>
      <%
         }
      %>
      </table>
      <h1>HTTP Request Parameters Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      <%
         eNames = request.getParameterNames();
         while (eNames.hasMoreElements()) {
            String name = (String) eNames.nextElement();
            String value = normalize(request.getParameter(name));
      %>
         <tr><td><%= name %></td><td><%= value %></td></tr>
      <%
         }
      %>
      <h1>HTTP Request Attributes Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      <%
         eNames = request.getAttributeNames();
         while (eNames.hasMoreElements()) {
            String name = (String) eNames.nextElement();
            String value = normalize((String) request.getAttribute(name));
      %>
         <tr><td><%= name %></td><td><%= value %></td></tr>
      <%
         }
      %>
      </table>
   </body>
</html>
<%!
   private String normalize(String value)
   {
      StringBuffer sb = new StringBuffer();
      for (int i = 0; i < value.length(); i++) {
         char c = value.charAt(i);
         sb.append(c);
         if (c == ';')
            sb.append("<br>");
      }
      return sb.toString();
   }
%>
{% endhighlight %}

- add URIEncoding="UTF-8" to server.xml (HTTP Connector)
- run `curl` on a UTF-8 enabled shell:

> curl --header "Foo: äöü" http://localhost:8080/foo/

{% highlight html %}
<html>
   <head>
      <title>Echo</title>
   </head>
   <body>
      <h1>HTTP Request Headers Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      
         <tr><td>user-agent</td><td>curl/7.38.0</td></tr>
      
         <tr><td>host</td><td>localhost:8080</td></tr>
      
         <tr><td>accept</td><td>*/*</td></tr>
      
         <tr><td>foo</td><td>Ã¤Ã¶Ã¼</td></tr>
      
      </table>
      <h1>HTTP Request Parameters Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      
      <h1>HTTP Request Attributes Received</h1>
      <table border="1" cellpadding="4" cellspacing="0">
      
      </table>
   </body>
</html>
{% endhighlight %}

## Solution

I'm not too familiar with the servlet API, so I'm unsure what to make of the
comment in ByteChunk.
Without being able to modify the third party application I wanted to "shibbolize", I had
to get sneaky.
I've used a custom ServletFilter to inject a dynamically proxied (using [ByteBuddy][byte-buddy])
[Request][tc-request] object, intercepting `getAttribute` and `getHeader` to fix the charset issue and
intercepting `getAttributeNames` to fix another annoying issue of Shibboleth
attribute names being hidden, but being returned when calling `getAttribute`.


[sp-attribute-access]:   https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPAttributeAccess                            "Shibboleth SP - Attribute Access Docs"
[tc-bytechunk]:          http://svn.apache.org/repos/asf/tomcat/tc7.0.x/trunk/java/org/apache/tomcat/util/buf/ByteChunk.java     "Tomcat 7.x ByteChunk.java"
[tc-faq]:                http://wiki.apache.org/tomcat/FAQ/CharacterEncoding#Q8                                                  "Tomcat FAQ on charset issues"
[tc-request]:            http://svn.apache.org/repos/asf/tomcat/tc7.0.x/trunk/java/org/apache/coyote/Request.java                "Tomcat 7.x Request.java"
[byte-buddy]:            http://bytebuddy.net                                                                                    "Byte Buddy library"
