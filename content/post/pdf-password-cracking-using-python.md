---
title: "PDF password cracking using Python"
date: 2016-06-28
categories:
- password cracking
- python scripts
tags:
- scripting
- python
- pdf

thumbnailImagePosition: left
thumbnailImage: /img/pdf-password-cracking-using-python/1.png
---

A simple Python script that can be used to brute force the password of a password protected PDF file.

<!--more-->

## Introduction

There are several occasions when I don't remember passwords to the PDF documents that are sent by banking services (banking statements) and telephone operators (mobile bills). Most of these documents, as you are aware, are password protected by complicated looking yet easy to guess passwords. For example, one of the password formats could be: your 8 digit customer ID ending with 45 concatenated with your date of birth in DDMMYYYY format. If your customer ID for this particular service was 12345645 and your date of birth was 18th August 1990, your password would be 1234564518081990. You get the drift right?

In any case, this is a complex looking long number but because you know how it is constructed, we can create a simple brute force script that uses the information available to us and attempts to decrypt a supplied PDF file.

## The script

The following non-elegant python script uses the [pyPDF](https://pypi.python.org/pypi/pyPdf/1.13) python library to attempt decryption on a PDF file with a password that is created on the fly using a logic that we already know. This script can be easily modified for different password generation algorithms. The following code in particular can be used to brute force passwords of the PDF mobile bills of one of the largest mobile operators in India. The format is the combination of the first 4 characters of your email in lowercase and a random 4 digit number.

Note that this script was tested on a PDF encrypted with 128 bit RC4 (most credit card statements) with 100% success. A wider implementation would be to loop something like [qpdf](https://sourceforge.net/projects/qpdf/?source=typ_redirect) with the passwords being generated using it's `--pasword='<password>'` and `--decrypt` option.

Anyways, here’s the simplified version with pyPDF:

```python
Import sys
from pyPdf import PdfFileReader

helpmsg = "Simple PDF brute force script\n"
helpmsg += "Cracks pwds of the format <first 4 chars of email>0000-9999."
helpmsg += "Example: snow0653\n\n"
helpmsg += "Usage: pdfbrute.py <encrypted_pdf_file> <email_address>"
if len(sys.argv) < 2:
        print helpmsg
        sys.exit()
        
pdffile = PdfFileReader(file(sys.argv[1], "rb"))
if pdffile.isEncrypted == False:
        print "[!] The file is not protected with any password. Exiting."
        exit

print "[+] Attempting to Brute force. This could take some time..."

z = ""
for i in range(0,9999):
        z = str (i)
        while (len(z) < 4):
                z = "0" + z
        
        a = str(sys.argv[2][:4] + str(z))
               
        if pdffile.decrypt(a) > 0:
                print "[+] Password is: " + a
                print "[...] Exiting.."
                sys.exit()
```

Feel free to modify this code and share/redistribute!

Happy Hacking!

---