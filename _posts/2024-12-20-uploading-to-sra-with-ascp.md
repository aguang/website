---
title: More up-to-date notes on using Aspera command line to upload to NCBI
draft: false
categories: ["posts"]
tags: sra, ncbi, ascp
classes: wide
---

I recently had to go through the very frustrating process of uploading fastqs to SRA (for a paper submission! Yay!). My uploads through http kept timing out constantly, and even when using the Aspera Connect client they recommended, if it errored out at all the first time the SRA site itself didn't recognize that the data was successfully uploaded later. So I turned to using the command line tool they recommended instead (also Aspera Connect). But the instructions were not really correct.

I won't upload a screenshot of the instructions because the SRA instructions stated not to include specific information, but the first step from it is to download and install the Aspera Connect software, which I already had from trying the http way.

The next step they write, is use the `ascp` command to upload the files via Aspera.

That was good and all, but I could not figure out for the life of me where this `ascp` command was, as opening up Aspera just gave me the GUI/browser extension. It wasn't anyhwere I could find in the IBM documentation on Aspera Connect for MacOS either. And it definitely wasn't in the SRA submission portal instructions.

Turns out it's here:
```
/Applications/IBM\ Aspera\ Connect.app/Contents/Resources/ascp
```

Super intuitive. Anyway, after finding it things went quickly since I just copied the command off the submission page.