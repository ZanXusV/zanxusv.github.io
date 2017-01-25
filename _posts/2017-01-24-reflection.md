---
layout:     post
title:      "Reflection while encapsulating JWT"
subtitle:   "Experience with  JsonWebToken"
date:       2017-01-25 10:50:00+0800
author:     "ZanXus"
header-img: "img/blog/header/post-bg-07.jpg"
thumbnail: /img/blog/thumbs/thumb07.png
tags: [reflection,jwt]
category: [java]
comments: true
share: true
---

## Tip:Basic knowledge of JWT is described [here](https://jwt.io/introduction/).

## Recently i got a task to change the current token information storage into JsonWebToken instead of redis.The JsonWebToken must at least include user id and token ttl,considering for further extension,it should be scalable which means that there maybe some other unknown fields to be included,and eventually i need to package it as a jar file to upload to company's private maven repository so that it could be imported easilly as maven dependency.

## In such circumstance,my first thought is to offer a base class naming `CustomClaim`,include 2 private fields `id` and `ttl`,and it can be extended if someone wants to add extra fields.However,to guarantee that `id` and `ttl` included in jwt, the `CustomClaim` can't have default constructor that without  arguments.So the `CustomClaim` class finally as folllows:

```java
package com.ucf.jwt;

public  class CustomClaim<T>{

    private T id;

    private Integer ttl;

    public CustomClaim(T id,Integer ttl) {
        this.id = id;
        this.ttl = ttl;
    }

    public T getId() {
        return id;
    }

    public void setId(T id) {
        this.id = id;
    }

    public Integer getTtl() {
        return ttl;
    }

    public void setTtl(Integer ttl) {
        this.ttl = ttl;
    }


}
```

## `id` declared as generic `T` because it can be `long` or `int` type,`ttl` denote the token expire time.

## Next step is to create sign.Basiclly we just need to set `iat`(issuedAt) and `exp`(expireAt),then we add our custom claims to `JWTCreator.Builder`.

```java
JWTCreator.Builder builder = JWT.create();
```

## But there is a limitation that JWT payload only accept instance of `Integer`||`Double`||`Boolean`||`Date`||`String`,otherwise it will throw an RuntimeException.Below is the JWT source code.

```java
/**
 * Add a custom Claim value.
 *
 * @param name  the Claim's name
 * @param value the Claim's value. Must be an instance of Integer, Double, Boolean, Date or String class.
 * @return this same Builder instance.
 * @throws IllegalArgumentException if the name is null or the value class is not allowed.
 */
 public Builder withClaim(String name, Object value) throws IllegalArgumentException {
   final boolean validValue = value instanceof Integer || value instanceof Double ||
         value instanceof Boolean || value instanceof Date || value instanceof String;
   if (name == null) {
       throw new IllegalArgumentException("The Custom Claim's name can't be null.");
   }
   if (!validValue) {
      throw new IllegalArgumentException("The Custom Claim's value class must be an instance of Integer, Double, Boolean, Date or String.");
   }

   addClaim(name, value);
   return this;
  }
```
## So it should be careful when define fields in `CustomClaim`'s derived class.Despite of this limitation, the `id` and `ttl` and other unknown business fields are also better not be null,so i do this check myself before jwt.Below is the warpped sign method:

```java
private static List<Field> getAllFields(CustomClaim claim) {
        List<Field> fields = new LinkedList(Arrays.asList(claim.getClass().getDeclaredFields()));
        Class<?> superClass = claim.getClass().getSuperclass();
        List<Field> inheritedFields = null;
        if (superClass.getName().equals(CustomClaim.class.getName())) {
            inheritedFields = Arrays.asList(superClass.getDeclaredFields());
        }
        if (inheritedFields != null) {
            fields.addAll(inheritedFields);
        }
        return fields;
    }



private static List<Field> validateFields(CustomClaim claim) {
        List<Field> fields = getAllFields(claim);
        fields.stream().forEach(
                field -> {
                    field.setAccessible(true);
                    Object value = null;
                    try {
                        value = field.get(claim);
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                    if (value == null) {
                        throw new IllegalArgumentException(String.format("Field '%s' can not be null",field.getName()));
                    }
                    final boolean validValue = value instanceof Integer || value instanceof Double || value instanceof Boolean || value instanceof Date || value instanceof String;
				    if (!validValue) {
					throw new IllegalArgumentException("The Custom Claim's value class must be an instance of Integer, Double, Boolean, Date or String.");
                }
        );
    }
    

private static JWTCreator.Builder encapsulateCreator(List<Field> fields, CustomClaim claim) {
        JWTCreator.Builder builder = JWT.create();
        fields.forEach(
          field -> {
            String name = field.getName();
            Object value = null;
            try {
                value = field.get(claim);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
          }
          builder.withClaim(name, value);
        });
        return builder;
    }



public static String sign(CustomClaim customClaim, String secret) {
        List<Field> fields=validateNullFields(customClaim);
        final Date iat = new Date();
        final Date exp = new Date(iat.getTime() + Long.valueOf(customClaim.getTtl() * 1000));
        String token = null;
        try {
            token = encapsulateCreator(fields,customClaim)
                    .withIssuedAt(iat)
                    .withExpiresAt(exp)
                    .sign(Algorithm.****(secret));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return token;
    }

```

## In the `getAllFields` method `Arrays.asList()` returns a fixed list so i new a LinkedList to add its inherited fields later.In java `reflect` package,`getDeclaredFields()` can only get all declared fields not including its super class's fields,so i get the true argument class's super class to check if it's `CustomClaim` class,if true get all fields of this super class and add it to variable `fields`. 

## Above all is the code to create sign,Now we come to the steps to verify token.

```java
private static <T extends CustomClaim> T getBusinessObject(Class<? extends CustomClaim> cls, JWT jwt) throws ClassNotFoundException, IllegalAccessException, InstantiationException, InvocationTargetException {
        Constructor<?>[] constructors = cls.getDeclaredConstructors();
        Constructor<?> ctor = null;
        for (Constructor<?> constructor:constructors){
            if (constructor.isAccessible()){
                ctor = constructor;
                break;
            }
        }
        if (ctor==null){
            throw new RuntimeException("No accessed constructor available");
        }
        Integer parameterCount = ctor.getParameterCount();
        Parameter[] parameters = ctor.getParameters();
        Object[] objects = new Object[parameterCount];
        IntStream.range(0, parameterCount)
                .forEach(i -> {
                    Parameter parameter = parameters[i];
                    if (parameter.getType().isPrimitive()) {
                        if (parameter.getType().getTypeName().equals(int.class.getName())) {
                            objects[i] = 0;
                        } else if (parameter.getType().getTypeName().equals(double.class.getName())) {
                            objects[i] = 0d;
                        } else if (parameter.getType().getTypeName().equals(boolean.class.getName())) {
                            objects[i] = true;
                        }
                    }
                });
        T t = (T) ctor.newInstance(objects);
        List<Field> fields = getAllFields(t);
        final T finalT = t;
        fields
                .forEach(field -> {
                    field.setAccessible(true);
                    try {
                        String type = field.getType().getTypeName();
                        String claimValue = jwt.getClaim(field.getName()).asString();
                        if (type.equals(Integer.class.getName()) || type.equals(int.class.getName())) {
                            field.set(finalT, Integer.parseInt(claimValue));
                        } else if (type.equals(Double.class.getName()) || type.equals(double.class.getName())) {
                            field.set(finalT, Double.parseDouble(claimValue));
                        } else if (type.equals(Boolean.class.getName()) || type.equals(boolean.class.getName())) {
                            field.set(finalT, Boolean.parseBoolean(claimValue));
                        } else if (type.equals(String.class.getName()) || type.equals(Object.class.getName())) {
                            field.set(finalT, claimValue);
                        } else {
                            throw new Exception("Unsupported filed type:" + type);
                        }
                    } catch (IllegalAccessException e) {
                        logger.error(e.getMessage());
                        e.printStackTrace();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                });
        return finalT;
    }



public static <T extends CustomClaim> T verify(Class<? extends CustomClaim> cls, String token, String secret) throws com.ucf.jwt.JWTVerificationException {
        JWTVerifier verifier = null;
        try {
            verifier = JWT.require(Algorithm.****(secret))
                    .build();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        JWT jwt = null;
        try {
            jwt = (JWT) verifier.verify(token);
        } catch (JWTVerificationException e) {
            throw new com.ucf.jwt.JWTVerificationException(e.getMessage());
        }
        T t = null;
        try {
            t = getBusinessObject(cls, jwt);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return t;
    }
```

##  `verify()` method accept a `Class` that extends `CustomClaim` and token&secret,in the JWT package `JWTVerificationException` is a `Runtime` Exception,so i rewrite a copy of it as a `Checked` Exception and throw it to force developer to catch it and do some business logic.

## At the first moment,i can't make sure which derived class of `CustomClaim` will be used as argument and what constructors it declared since the base class `CustomClaim` does't has a default constructor with empty arguments  so the `cls.newInstance()` is not work.Hence the problem is how to instantiate the class.

## Finally i have to get all its constructors and traverse the constructor array,find out which constructor is accessible,if a constructor is private,the `setAccessible()` method is not work according to JDK `java.lang.reflect` package:

```java
/* Check that you aren't exposing java.lang.Class.<init> or sensitive
       fields in java.lang.Class. */
    private static void setAccessible0(AccessibleObject obj, boolean flag)
        throws SecurityException
    {
        if (obj instanceof Constructor && flag == true) {
            Constructor<?> c = (Constructor<?>)obj;
            if (c.getDeclaringClass() == Class.class) {
                throw new SecurityException("Cannot make a java.lang.Class" +
                                            " constructor accessible");
            }
        }
        obj.override = flag;
    }

```

## So if there is no public constructor,i will throw a `RuntimeException`.If got an available constructor,then use `getParameterCount()` get the constructor's parameter count,and new an object array whose length equals parameter count,but all elements in this array is `null` as initialized,so if the constructor's parameters have primitive data type,the objects can't be used as arguments of `newInstance(args)` method,so we foreach parameters then check if it is primitive,if so then we give it a initial value.And the rest is to call the constructor's `newInstance(objects)` method and then set the instance's fileds from JWT object,finally return the object that extends `CustomClaim`.

## Now the code is completed,the last one thing to do is to package project to a jar file,since i use `gradle` as build tool,define a task that named `fatJar`:

```gradle
task fatJar(type: Jar) {
    manifest {
        attributes 'Implementation-Title': 'Gradle Jar File',
                'Implementation-Version': version
    }
    exclude 'META-INF/*.RSA', 'META-INF/*.SF','META-INF/*.DSA'
    baseName = "jwt"
    from { configurations.compile.collect { it.isDirectory() ? it : zipTree(it) } }
    with jar
}
```

## Which should be mentioned is that the **exclude 'META-INF/*.RSA', 'META-INF/*.SF','META-INF/*.DSA'**{: style="color: red"} is necessary otherwise when import this jar dependency into maven project,an error `java.lang.SecurityException: Invalid signature file digest for Manifest main attributes`.

## Then just use `gradle clean fatJar` and the jar file will generated under ${project}/build/libs directory.