---
layout: post
published: true
title: Auto incrementing build number in Xcode 4
date: 2013-03-11 22:54
comments: true
categories: [iOS, Xcode]
alias: /2013/03/auto-incrementing-build-number-in-xcode-4/index.html
author: chrisr
---

Updated: Please see [Auto incrementing build number in Xcode revisited]({% post_url 2015-07-29-auto-incrementing-build-number-in-xcode-revisited %})

I've recently been doing a bit of iOS development and have started using <a href="http://hockeyapp.net/" title="HockeyApp" target="_blank">HockeyApp</a> for distributing beta versions and collecting crash reports. Hockey requires that your builds have a unique version number, they have a great knowledge base article showing <a href="http://support.hockeyapp.net/kb/how-tos-faq/how-to-do-versioning-for-beta-versions-on-ios-or-mac" title="How to do versioning for beta versions on iOS or Mac" target="_blank">How to do versioning for beta versions on iOS or Mac</a>. The only problem I found with their solution was that the file containing the build number needs to exist beforehand. So, here's an update to the script to create the file if it doesn't already exist:

{% highlight sh %}
echo "Updating build number..."
if [ "$CONFIGURATION" == "Release" ]; then
    set -o noclobber
    echo "Build file: ${SRCROOT}/buildnumber.xcconfig"
    echo "BUILD_NUMBER = 0" > "${SRCROOT}/buildnumber.xcconfig"
    set +o noclobber
    /usr/bin/perl -pe 's/(BUILD_NUMBER = )(\d+)/$1.($2+1)/eg' -i buildnumber.xcconfig
fi
{% endhighlight %}

Also, don't forget to add the buildnumber.xcconfig file to your .gitignore.
