---
title: ZFS实现简单介绍
description: ZFS的部分实现介绍(翻译转载)
slug: note-zfs
date: 2023-08-25 18:06:45+0800
categories:
    - learn
tags:
    - zfs
    - 文件系统
    - RAID
    - COW
toc: true
---

翻译自 [ZFS 101—Understanding ZFS storage and performance](https://arstechnica.com/information-technology/2020/05/zfs-101-understanding-zfs-storage-and-performance/)

ZFS 是一个linux广泛使用的文件系统，最早在2001年由Sun公司在自己的操作系统中使用，后来被移植到linux。ZFS的主要特性是多盘管理、COW、RAID。可以认为ZFS是一个单机的采用EC条带的多盘文件系统.

<!--more-->

* ashfit、recordsize
  ashift是ZFS的块对齐地址要求，与存储设备的读写单元对应时能有效减少写放大。当前SSD盘的擦写单元普遍较大，推荐设置为4KB或8KB
  recordsize是ZFS的COW数据读写单位，即写入数据的最小单位，一般范围为4KB~1MB，默认128KB。不过这个单位可以动态修改，只对新写入的数据生效。

* COW(写时拷贝)和RAID
  ZFS的所有文件数据均使用RAID条带(即EC条带)进行存储。为了保证写入不会破坏旧数据，ZFS的所有修改写均采用COW技术。不过ZFS的数据盘组合是固定的，即将一组硬盘创建为vdev，固定冗余比例，作为一个逻辑盘来使用。一般ZFS会创建多个vdev，然后可以从这些vdev提供的存储空间中，创建逻辑文件系统(dataset)或块设备(zvol)。

  {{< image "zpool.png" "zpool示意图" "max-height:550px;" >}}

* ZFS Intent Log
  ZFS将写请求分为同步写和异步写。对于异步写，ZFS是直接写条带；而对于同步写，ZFS为了降低写延迟并且聚合写请求，会将写请求缓存内存中(TXG聚合)，并将写日志持久化(ZIL)。ZFS还允许ZIL写入单独的SLOG的vdev中(一般是SSD)，以提高同步写的性能。

  {{< image "ZIL.png" "ZIL日志" "max-height:400px;" >}}

* 压缩
  压缩对于COW的条带写来说，是天然适配的特性。不同于in-place修改会因为内容不同导致压缩后的数据长短不一，由于是COW，每次写数据都是在新的位置写入，不用考虑数据长度变动的问题。ZFS推荐使用LZ4压缩算法，提供最快的压缩速度。
