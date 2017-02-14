+++
date = "2017-02-12T09:25:39-06:00"
title = "hugo test"
draft = false

+++

This is just a test.
<!--more-->

~~~java
package com.morningstar.alerts.spring.sample.configuration;

import com.morningstar.alerts.spring.sample.interceptor.TestInterceptor;
import com.morningstar.alerts.spring.sample.interceptor.TestInterceptor2;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
@EnableWebMvc
public class WebConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // register interceptors here http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-config-interceptors
        registry.addInterceptor(new TestInterceptor());
        registry.addInterceptor(new TestInterceptor2());
    }
}
~~~
