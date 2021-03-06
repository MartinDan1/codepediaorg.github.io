---
layout: post
title: Error handling design in a JAX-RS REST API
description: "In this post I will present a few points I had to do to migrate a Java REST API from JBoss EAP 6/JEE 6/JAX RS 1.0 to Jboss EAP7/JEE 7/JAX RS 2.0"
author: ama
permalink: /ama/how-to-migrate-a-java-ee-rest-api-from-jboss-eap6-to-jboss-eap7
published: false
categories: [article]
tags: [jboss, wildfly, rest, resteasy, javaee, keycloak, cors, error-handling]
---

In this post I will briefly describe the steps and issues encountered when migrating a Java EE REST API from JBoss EAP 6 to JBoss EAP 7 - this implies migrating from a Java EE 6/JAX RS 1.0 implementation to a Java EE 7/JAX RS.2.0 implementation. The trigger was the announcement from Red Hat regarding the general availability of their JBoss Enterprise Application Platform 7 (JBoss EAP) [^1]. JBoss EAP 7 is based on Wildfly 10[^2], so the code snippets showed along the post should work on Wildfly 10 too.

[^1]: <https://www.redhat.com/en/about/press-releases/red-hat-delivers-jboss-eap-7-foundation-hybrid-cloud-applications>
[^2]: <https://access.redhat.com/documentation/en/red-hat-jboss-enterprise-application-platform/7.0/paged/introduction-to-jboss-eap/chapter-2-overview-of-jboss-eap>

<!--more-->

* TOC
{:toc}

**Generic Exception Mapper**

```java
package ch.bkw.oneportal.commons.exceptionhandling;

import ch.bkw.oneportal.commons.env.ConfigFactory;
import org.codehaus.jackson.map.ObjectMapper;
import org.slf4j.Logger;

import javax.inject.Inject;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;
import java.io.IOException;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.UUID;

/**
 * Generic exception mapper
 *
 * Prepares error messages for front end clients
 *
 */
@Provider
public class GenericExceptionMapper implements ExceptionMapper<Exception> {

    @Inject
    private Logger logger;

    public Response toResponse(Exception e) {
        String uuid = UUID.randomUUID().toString();
        if(e instanceof AppException){
            return getAppExceptionResponse((AppException) e, uuid);
        } else if (e instanceof WebApplicationException) {
            return getWebApplicationResponse((WebApplicationException) e, uuid);
        } else {
            return getUnknownExceptionResponse(e, uuid);
        }
    }

    /**
     * Builds Response based on an unknown exception
     *
     * @param e
     * @param uuid - unique error identifier
     * @return
     */
    private Response getUnknownExceptionResponse(Exception e, String uuid) {

        logger.error("Unknown server error with uuid " + uuid, e);

        String developerMessage = getDeveloperMessage(e, uuid);

        ErrorMessage unknownErrorMessage =  new ErrorMessage.Builder()
                .httpStatus(Response.Status.INTERNAL_SERVER_ERROR.getStatusCode())
                .uuid(uuid)
                .developerMessage(developerMessage)
                .message("Unknown server exceptionhandling, see log files for more details - look for the uuid " + uuid)
                .build();

        return Response.status(unknownErrorMessage.getHttpStatus())
                .entity(unknownErrorMessage)
                .type(MediaType.APPLICATION_JSON_TYPE)
                .build();
    }


    private String getDeveloperMessage(Exception e, String uuid) {
        String environment = ConfigFactory.getEnvironment();
        String developerMessage = "Search in log files for " + uuid;

        boolean isDevEnvironment = "dev".equals(environment) || "local".equals(environment);

        if(isDevEnvironment){
            developerMessage = getStackTraceAsString(e);
        }
        return developerMessage;
    }

    /**
     *  Builds Response based on WebApplicationException
     * @param webApplicationException
     * @param uuid
     *
     * @return
     */
    private Response getWebApplicationResponse(WebApplicationException webApplicationException, String uuid) {

        logger.error("WebApplicationException ", webApplicationException);

        ErrorMessage jaxRSerrorMessage =  new ErrorMessage.Builder()
                .httpStatus(webApplicationException.getResponse().getStatus())
                .uuid(uuid)
                .developerMessage("Search in log files for " + uuid)
                .message(webApplicationException.getMessage())
                .build();

        return Response.status(jaxRSerrorMessage.getHttpStatus())
                .entity(jaxRSerrorMessage)
                .type(MediaType.APPLICATION_JSON_TYPE)
                .build();
    }

    /**
     * Returns a Response built from an AppException and also "pretty" logs it in JSON format
     *
     * @param appException
     * @param uuid - unique error identifier
     * @return
     */
    private Response getAppExceptionResponse(AppException appException, String uuid){

        appException.getErrorMessage().setUuid(uuid);

        ObjectMapper mapper = new ObjectMapper();
        try {
            logger.error("Application Exception: \n "
                    + mapper.writerWithDefaultPrettyPrinter().writeValueAsString(appException.getErrorMessage())
                    + "\n");
        } catch (IOException e1) {
            logger.error("Error when pretty printing the error message", e1);
        }

        return Response.status(appException.getErrorMessage().getHttpStatus())
                    .entity(appException.getErrorMessage())
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .build();
    }

    private String getStackTraceAsString(Exception e){
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        e.printStackTrace(pw);
        return sw.toString(); // stack trace as a string
    }

}
```

First thing to note, the @Provider annotation https://docs.oracle.com/javaee/7/api/javax/ws/rs/ext/ExceptionMapper.html
" so that is automatically "

How to register it see bellow

```
import org.jboss.resteasy.plugins.interceptors.CorsFilter;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;
import java.util.HashSet;
import java.util.Set;

@ApplicationPath("")
public class TestApplication extends Application {

    private Set<Object> singletons = new HashSet<Object>();
    private HashSet<Class<?>> classes = new HashSet<Class<?>>();

    public OnePortalB2bApplication()
    {
        CorsFilter corsFilter = new CorsFilter();
        corsFilter.getAllowedOrigins().add("*");
        corsFilter.setAllowedMethods("OPTIONS, GET, POST, DELETE, PUT, PATCH");
        singletons.add(corsFilter);
        singletons.add(new Ping());

        classes.add(ApiResource.class);
        classes.add(V1Resource.class);
        classes.add(UsersResource.class);
        classes.add(AdminResource.class);
        classes.add(GenericExceptionMapper.class);
    }

    @Override
    public Set<Object> getSingletons() {
        return singletons;
    }

    @Override
    public HashSet<Class<?>> getClasses(){
      return classes;
    }

}
```

** Usage example **

```
public void deleteUser(String userId) throws AppException {

    UserEntity userEntity =  userServiceDB.getUser(userId);

    if(userEntity == null){

        ErrorMessage errorMessage = new ErrorMessage.Builder()
                .httpStatus(Response.Status.NOT_FOUND.getStatusCode())
                .message("User was not found in the database")
                .build();

        throw new AppException("User was not found in the OnePortal DB", errorMessage);
    } else {
        userServiceDB.deleteUser(userId);
    }
}
```

AppException
```
public class AppException  extends Exception {

    private ErrorMessage errorMessage;

    public AppException(String message, ErrorMessage errorMessage){
        super(message);
        this.errorMessage = errorMessage;
    }

    public ErrorMessage getErrorMessage() {
        return errorMessage;
    }

    public void setErrorMessage(ErrorMessage errorMessage) {
        this.errorMessage = errorMessage;
    }
}
```

Either stick to checked or unchecked exceptions, mention article from jenkov

ErrorMessage
```
import org.codehaus.jackson.annotate.JsonIgnoreProperties;
import org.codehaus.jackson.annotate.JsonProperty;


@JsonIgnoreProperties(ignoreUnknown = true)
public class ErrorMessage {

    @JsonProperty("http_status")
    private Integer httpStatus;

    @JsonProperty("error_code")
    private String errorCode;

    @JsonProperty("documentation_link")
    private String documentationLink;

    @JsonProperty("message")
    private String message;

    @JsonProperty("developer_message")
    private String developerMessage;

    @JsonProperty("uuid")
    private String uuid;

    public static class Builder {

        private Integer httpStatus;

        private String errorCode;

        private String documentationLink;

        private String message;

        private String developerMessage;

        private String uuid;

        public Builder(Integer httpStatus, String documentationLink, String errorCode, String message, String developerMessage, String uuid) {
            this.httpStatus = httpStatus;
            this.documentationLink = documentationLink;
            this.errorCode = errorCode;
            this.message = message;
            this.developerMessage = developerMessage;
            this.uuid = uuid;
        }

        public Builder(){};

        public Builder httpStatus(Integer httpStatus){
            this.httpStatus = httpStatus;
            return this;
        }

        public Builder documentationLink(Integer documentationLink){
            this.httpStatus = documentationLink;
            return this;
        }

        public Builder errorCode(String errorCode){
            this.errorCode = errorCode;
            return this;
        }

        public Builder message(String message){
            this.message = message;
            return this;
        }

        public Builder developerMessage(String developerMessage){
            this.developerMessage = developerMessage;
            return this;
        }

        public Builder uuid(String uuid){
            this.uuid = uuid;
            return this;
        }

        public ErrorMessage build(){
            return new ErrorMessage(this);
        }
    }

    private ErrorMessage(Builder builder){
        httpStatus = builder.httpStatus;
        errorCode = builder.errorCode;
        message = builder.message;
        developerMessage = builder.developerMessage;
        documentationLink = builder.documentationLink;
        uuid = builder.uuid;
    }

    public Integer getHttpStatus() {
        return httpStatus;
    }

    public void setHttpStatus(Integer httpStatus) {
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() {
        return errorCode;
    }

    public void setErrorCode(String errorCode) {
        this.errorCode = errorCode;
    }

    public String getDocumentationLink() {
        return documentationLink;
    }

    public void setDocumentationLink(String documentationLink) {
        this.documentationLink = documentationLink;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public String getDeveloperMessage() {
        return developerMessage;
    }

    public void setDeveloperMessage(String developerMessage) {
        this.developerMessage = developerMessage;
    }

    public String getUuid() {
        return uuid;
    }

    public void setUuid(String uuid) {
        this.uuid = uuid;
    }
}

```

## Other good REST(easy) related resources

* [RESTEasy Documentation examples, HTML, PDF, Javadocs](https://resteasy.jboss.org/docs.html) - for all versions...
* [Java EE 7 and JAX-RS 2.0](https://www.oracle.com/technetwork/articles/java/jaxrs20-1929352.html) by Adam Bien
* [What's new in JAX-RS 2.0](https://www.infoq.com/news/2013/06/Whats-New-in-JAX-RS-2.0) at InfoQ
* [Java EE 7 Deployment Descriptors](https://antoniogoncalves.org/2013/06/04/java-ee-7-deployment-descriptors/)

## References
