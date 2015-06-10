---
layout: post
title: "Writing CRUD Applications in Go"
---

I have been writing CRUD applications in Go for a few months now, and I was asked
to show how I was doing this. The underlying assumption being that it's unnecessarily
difficult to write CRUD applications without some form of framework and ORM to
support you.

Needless to say, I disagree with this assumption; as long as you have a bit of discipline
to make up for the missing structure provided by your average framework.

As a point of reference, I have also written CRUD applications using Django,
Flask, and a Java inspired custom ORM and templating framework.

# The Support Structure

To even begin writing a web application in Go, we are going to want a bit of support for it.
While Go's webserver is likely capable of handling everything on its own, I still
prefer to keep it behind a well tested and proven web server. In this case,
we will go with the Nginx web server, and use it to terminate SSL, serve static files,
and proxy back to the Go server.

The other half of the equation is the DB. I will be using MySQL.

### An Aside: Why MySQL

Unfortunately, any time I mention that I use MySQL as a DB, I get plenty of virtual
dirty looks. So why MySQL? The answer is twofold. It's a time tested and proven
DB. Two, it's what I know. I have spent the last 5 years doing both administration for,
and automation against, MySQL. I've also spent some time with PostgreSQL at a job
between it, but I was never terribly impressed by a DB which tries so hard to compete
with Oracle on features. But, this is just my opinion - for what we're doing here,
neither one is really going to do better than the other.

Ultimately, I believe you should go with what you know, and I know MySQL.

### Support Structure Cont.

I will configure MySQL to use up to 50% of my machine's memory for its buffer pool,
and make sure to configure the appropriate backups for it.

The final component is something which I don't install on the server - Ansible.
If I can avoid it, I never actually modify any files on the web server, nor do I ever
run apt-get or yum by hand. With proper DB backups, this makes it really simple to
rebuild my server anywhere I want, within minutes. It's also provider agnostic,
so I can use the same tools on Linode as I use on AWS.

# Go Boilerplate

No matter how much you try to avoid it, you're going to end up with boilerplate if
you're writing a web application. Unfortunately for us, Go is no different in this
regard.

## Third Party Libraries

Our need for third party libraries can be as broad or narrow as you want
it to be, in our case we will be using three external packages.

### go-mysql-driver

https://github.com/go-sql-driver/mysql/

We use this as our MySQL driver, simply because it works. We will use it as a driver
to the database/sql package, so when I finally see the light and switch DBs, it will
be fairly straightforward to re-use the existing code.

### gcfg

https://code.google.com/p/gcfg/

I always prefer to use ini- style config files, for a couple of reasons:

- Everyone knows the ini config file
- It supports comments
- Ansible, Puppet and Chef all provide tools for manipulating ini files directly

When ini files are no longer enough, I move on to using YAML config files. They offer
similar capabilities as ini files, but with the ability to specify more complicated
nesting and even language specific types.

### DATA-DOG

https://github.com/DATA-DOG/go-sqlmock

Finally, a third party tool for mocking out the DB for our testing. The data dog library
acts as a driver, letting you substitute it for your normal driver with no changes to your
code required.

The one current shortfall of the library is the lack of support for parallel tests. I
haven't had this bite me yet, but when it does I'll likely end up submitting a pull request,
I like this library so much.

## Main

I'm skipping the import section, simply because I am assuming that if you have made it
this far, you understand how to fill out the import section to make this all work.





