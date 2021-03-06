---
layout: post
title: "Edge Case: Nested and Mixed Lists"
categories:
  - Edge Case
tags:
  - content
  - css
  - edge case
  - lists
  - azure
last_modified_at: 2017-03-09T14:25:52-05:00
---

Nested and mixed lists are an interesting beast. It's a co**rner case to make sure that lists within lists do not break the ordered list numbering order and list styles go deep enough.**


> [!WARNING]
> Hello

![picture 1](../images/9aa96b7211bd2b50ef0216de71fb8ef9ee9521bfe75b2e9cc428e309186f2548.png)  



> [!IMPORTANT]
> this is dangerous
> 

## Ordered -- Unordered -- Ordered

1. ordered item
2. ordered item
  * **unordered**
  * **unordered**
    1. ordered item
    2. ordered item
3. ordered item
4. ordered item

## Ordered -- Unordered -- Unordered

1. ordered item
2. ordered item
  * **unordered**
  * **unordered**
    * unordered item
    * unordered item
3. ordered item
4. ordered item

## Unordered -- Ordered -- Unordered

* unordered item
* unordered item
  1. ordered
  2. ordered
    * unordered item
    * unordered item
* unordered item
* unordered item

## Unordered -- Unordered -- Ordered

* unordered item
* unordered item
  * unordered
  * unordered
    1. **ordered item**
    2. **ordered item**
* unordered item
* unordered item