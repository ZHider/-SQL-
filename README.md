---
description: Advanced SQL injection to operating system full control
---

# 通过高级 SQL 注入完全控制操作系统

Bernardo Damele Assumpção Guimarães

bernardo.damele@gmail.com

April 10, 2009



本白皮书讨论了由于与数据库通信的网络应用程序中的 SQL 注入漏洞而导致的服务器安全漏洞。

自一位著名黑客提出 SQL 注入一词以来，十多年过去了，它仍然被认为是主要的应用程序威胁之一。关于这个漏洞已经有很多论述，但还没有揭示其所有方面和影响。

本文旨在整理一些现有知识，介绍新技术，并演示如何在被忽视和理论上无法利用的情况下，通过 SQL 注入漏洞完全控制数据库管理系统的底层操作系统、文件系统和内部网络。
