---
title: Turn off local preview mirroring
titleSuffix: An Azure Communication Services article
description: This article describes how to turn off local preview mirroring.
author: yassirbisteni
manager: gaobob

ms.author: yassirb
ms.date: 6/26/2025
ms.topic: quickstart
ms.service: azure-communication-services
ms.subservice: calling
ms.custom: mode-other, devx-track-js

zone_pivot_groups: acs-plat-web-windows-android-ios
---

### Code to turn off local preview mirroring

````swift
var renderer = try VideoStreamRenderer(localVideoStream: localVideoStream)
var view = try renderer?.createView(withOptions: CreateViewOptions(scalingMode: scalingMode))
var view?.layer.transform = CATransform3DMakeAffineTransform(CGAffineTransformMakeScale(-1.0, 1.0))
````