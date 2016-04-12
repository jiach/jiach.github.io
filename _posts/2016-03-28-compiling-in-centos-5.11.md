---
layout: post
title: Compiling in Centos 5.11
date: '2016-03-29 15:10:1 -0400'
categories: gcc
comments: true
---

<style>
table{
    border-collapse: collapse;
    border-spacing: 0;
    border:2px solid #a6cee3;
}

th{
    border:2px solid #1f78b4;
}

td{
    border:1px solid #1f78b4;
}
</style>

TASC was written with C/C++ under Ubuntu W, with the kernel 4.2.0-34. However, a user reported that TASC could not compile on his system. Turns out his cluster is still running RHEL 5.6, with kernel 2.6.18. I set out to compile TASC under CentOS 5.11 and hopefully this will help whoever tries to support legacy system for their software.

First issue I bumped into, was that the default Gnu Scientific Library is of version 1.13. This version was released in September 2009, and it's not the oldest out there (I'm sure there are more annoyingly paranoid system administrators still running gsl-1.0, for "stability" reasons). However, TASC needs the ``gsl_integration_cquad()`` function, which first appeared in version 1.15. So I downloaded the latest gsl-2.1, compiled and installed from [source](http://gnu.mirror.constant.com/gsl/gsl-2.1.tar.gz). It's very straightforward, involving only three commands, ``./configure``, ``make``, ``make install``. YMMV.

Second issue is the version of g++. TASC requires some C++11 features that are not supported by the default gcc-4.1 provided by CentOS. We need at least gcc-4.4 for it to compile. Thank goodness we can install GCC 4.4 from their official depot. Just use

```
yum install gcc44-c++ gcc44
```

Don't forget to change your configuration for CMAKE to use ``g++44`` instead of the provided ``g++``, which is of version 4.1.

The third issue is that we need to install a newer version of boost libraries (by newer I mean 1.41), this can be done through the so called Extra Packages for Enterprise Linux (EPEL) project. The repo has both boost-1.41 and boost-1.48, and we can install either.
1. download the rpm of the appropriate architecture from this [link](https://fedoraproject.org/wiki/EPEL).
2. install the rpm by double clicking and opening the downloaded rpm file.
3. install boost-1.48 by this command ``sudo yum install boost148 boost148-static boost148-devel``
