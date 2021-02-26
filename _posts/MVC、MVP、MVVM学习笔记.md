---
title: MVC|MVP|MVVM学习笔记
date: 2020-06-29 13:55:38
tags: 
---


# 一、MVC

## View

布局文件全部都属于View，但是不是只要有布局文件就可以在手机上显示，要有Avtivity的配合，所以Activity中也必不可少的有View的部分。

## Controller

Controller是控制器，都在Activity中，常见的功能有:

1. 给按钮添加点击事件；
2. 给View设置文字、图片等。

## Model

Model是管理数据模型，例如：

1. 网络请求获取到的数据，通过回调传给Controller，也就是Activity，然后Activity再将数据设置给View。

现在Android推出了AndroidViewModel，可以对模型中的数据调用observe方法进行监测，不论是基本数据类型还是自定义bean都可以监测；

AndroidViewModel相较于自己写的Model:

1. 省去了对回调的定义
2. 更容易的去单独管理数据，自己写的Model可能还要调用网络请求的封装方法或者其它方法来获取更新数据

# 二、MVP

## Presenter

Presenter的作用主要是弱化Model，Presenter中具体定义方法调用Model中方法，如果是AndroidViewModel，那么Presenter的作用就
是改变AndroidViewModel数据的变化。

Presenter要能够设置给对用页面的View。

使用时定义BasePresenter接口，内部定义Presenter的一些通用方法。

## BaseView

BaseView为一个接口，内部一定要有设置给对其设置Presenter对象的方法。

最后实现BaseView的一定是Activity，在Activity中可以实例化Presenter，并调用设置Presenter对象的方法将其传递给View。

## Contract

将一个页面对应的Presenter和BaseView都以内部类的形式组合在Contract中

1. 定义BasePresenter的子类，其中写对应页面对其Model的数据的获取及改变
2. 定义子接口继承BaseView，其中丰富对对应页面的View设置文字和图片(数据)的方法，Activity实现子接口，
什么时候调用还是在Controller的点击事件或者对AndroidViewModel的数据监测observe方法中。

好处: 方便管理和查看对Model的获取及改变，方便查看对View的设置都有哪些。

弊端: 如果页面很复杂，Model的获取及改变很多，View的设置也很多，Contract作为两者的容器会非常庞大臃肿。