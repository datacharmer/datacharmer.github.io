---
title: The end of dbdeployer
description: After 18 years, MySQL-Sandbox and its successor dbdeployer come to an end
date: 2023-10-29
---


[dbdeployer][dbdeployer] has come to the end of maintenance, and won't be updated anymore.
I am looking for volunteers to take over the development, if there is still interest in using it.


## A bit of history

[dbdeployer][dbdeployer] has been a project that has kept my attention for 18 years. It started in 2005, under the name of [MySQL-Sandbox][mysql_sandbox], written in Perl and updated up to 2018.
In 2018, tired of the many problems related to distributing a set of executables with Perl, I wrote it from scratch in Go. It was a great experience, because I learned a new language, made a lot of mistakes, learned from them, and eventually made a tool that coult be enhanced with relative ease.

Between 2018 and 2022, `dbdeployer` added many features that were not possible with its Perl ancestor, and with such features came more maintenance.
In the same period, [MySQL][mysql] became more complex and added more features of its own account. For some time, dbdeployer kept up, at the cost of many week-ends dedicated to adjusting to the changes that came down the line.

But in the past years my professional involvement with databases in general and MySQL in particular has diminished, and finally ended. For the past 6 years I have kept alive my personal interest in MySQL through dbdeployer. Lately I have realized that my personal interest is not enough to justify my development of a tool that requires intimate knowledge of the database that it wants to assists users with. For this reason, and for a lot of small nuisances that the maintenance of such a tool requires, I have decided to step down as maintainer of dbdeployer. 

## Why I am stepping down

Maintaining dbdeployer is a burden, not only for the coding itself, but especially for the testing and the promise of compatibility that I made when I started the project. The idea is to keep dbdeployer able to work with MySQL versions from 4.1 up to 8.x, and to do so I have to maintain a set of database binaries for every major release made between 2004 and today. In some of my machines I have 50+ GB with mysql binaries that get used during tests at every release.
An then there are the "adjustments", meaning that I need to react to changes made by MySQL that interfere with dbdeployer functioning. While dbdeployer has made a promise of backward compatibility and adopted semantic versioning, MySQL has not, and every new point release can (and does) make incompatible changes that require adjusting dbdeployer code to keep it working. 

## I am looking for a new maintainer

If anyone with some knowledge of Go wants to adopt the project, I will gladly make them access to the repository and give them guidance through the current code.
Alternatively, anyone can fork the project and maintain a new copy from another repository.


[dbdeployer]: https://github.com/datacharmer/dbdeployer
[mysql_sandbox]: https://github.com/datacharmer/mysql-sandbox
[mysql]: https://www.mysql.com
