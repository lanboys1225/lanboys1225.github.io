---
title: 内嵌iframe实现html页面轮播
date: 2019-08-06 19:52:39
category: html
tags: [html, iframe, banner, swiper]
---
需求是这样的，我们的产品要在某个展厅中展示，主办方要求我们提供部分系统界面在电视上作为宣传页，时间紧迫，已经来不及开发了，我就在想，能不能轮播我们系统已经存在的页面（哈哈，我是有多么懒），省心省力，想法虽好，但是作为一个前端白痴 ( 没怎么写过前端代码，轮播图都没实现过 )，对我来说还是有难度的啊~~ 但是没办法，自己提出来的想法，含着泪也得实现。

经过一天的查资料，终于实现了，实现后发现原来这么简单，下面把代码贴出来，鼓励自己继续学习前端。

> 代码是jsp写的，用在html的话改改就可以了
>
> 轮播控件用的是 [swiper](https://www.swiper.com.cn/) , 然后用iframe实现网页内嵌

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@page import="java.util.Calendar" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="shiro" uri="http://shiro.apache.org/tags" %>
<c:set var="ctx" value="${pageContext.request.contextPath}"/>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Swiper demo</title>
    <!-- Link Swiper's CSS -->
    <link rel="stylesheet" href="${ctx}/static/swiper/css/swiper.min.css">
    <!-- Demo styles -->
    <style>
        html, body {
            position: relative;
            height: 100%;
        }
        body {
            background: #eee;
            font-family: Helvetica Neue, Helvetica, Arial, sans-serif;
            font-size: 14px;
            color: #000;
            margin: 0;
            padding: 0;
        }
        .swiper-container {
            width: 100%;
            height: 100%;
        }
        .swiper-slide {
            text-align: center;
            font-size: 18px;
            background: #fff;

            /* Center slide text vertically */
            display: -webkit-box;
            display: -ms-flexbox;
            display: -webkit-flex;
            display: flex;
            -webkit-box-pack: center;
            -ms-flex-pack: center;
            -webkit-justify-content: center;
            justify-content: center;
            -webkit-box-align: center;
            -ms-flex-align: center;
            -webkit-align-items: center;
            align-items: center;
        }
    </style>
</head>
<body>
<!-- Swiper -->
<div class="swiper-container">
    <div class="swiper-wrapper">
    	<!-- url1,url2,url3,url4 替换成网页地址，注意跨域问题-->
        <div class="swiper-slide">
            <iframe height="100%" width="100%" frameborder="0" scrolling="no"
                    src="url1"></iframe>
        </div>
        <div class="swiper-slide">
            <iframe height="100%" width="100%" frameborder="0" scrolling="no"
                    src="url2"></iframe>
        </div>
        <div class="swiper-slide">
            <iframe height="100%" width="100%" frameborder="0" scrolling="no"
                    src="url3"></iframe>
        </div>
        <div class="swiper-slide">
            <iframe height="100%" width="100%" frameborder="0" scrolling="no"
                    src="url4"></iframe>
        </div>
    </div>
    <!-- 如果需要导航按钮 -->
    <div class="swiper-button-prev"></div>
    <div class="swiper-button-next"></div>
</div>

<!-- Swiper JS -->
<script src="${ctx}/static/swiper/js/swiper.min.js"></script>
<script type="text/javascript" src="${ctx}/static/jquery/1.9.1/jquery.min.js"></script>

<!-- Initialize Swiper -->
<script
    var swiper = new Swiper('.swiper-container', {
        loop: true,
        autoplay: {
            delay: 10000//10秒切换一次
            // stopOnLastSlide: false,
            // disableOnInteraction: true
        },
        //开启循环
        speed: 2000,
        // 如果需要前进后退按钮
        navigation: {
            nextEl: '.swiper-button-next',
            prevEl: '.swiper-button-prev'
        }
    });
</script>
</body>
</html>

```