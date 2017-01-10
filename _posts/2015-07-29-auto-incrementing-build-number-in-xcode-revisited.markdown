---
title: Auto incrementing build number in Xcode revisited
date: 2015-07-29 14:59:00 Z
categories:
- iOS
- Xcode
layout: post
comments: true
author: chrisr
---

For sometime now, I've been using the script in the post [The Best of All Possible Xcode Automated Build Numbering Techniques](http://blog.jaredsinclair.com/post/97193356620/the-best-of-all-possible-xcode-automated-build) to automatically update the build number in Xcode based on the number of git commits.
Unfortunately this script doesn't update the version number in the dSYM bundle, which causes Hockey to complain when uploading a new build.
A quick search found the answer in a comment on this post [A sensible way to increment bundle version (CFBundleVersion) in Xcode](http://tgoode.com/2014/06/05/sensible-way-increment-bundle-version-cfbundleversion-xcode/)

Here's the complete script:

{% highlight sh %}
git=`sh /etc/profile; which git`
appBuild=`$git rev-list --all | wc -l`
if [ $CONFIGURATION = "Debug" ]; then
branchName=`$git rev-parse --abbrev-ref HEAD`
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBuild-$branchName" "${TARGET_BUILD_DIR}/${INFOPLIST_PATH}"
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBuild-$branchName" "${BUILT_PRODUCTS_DIR}/${WRAPPER_NAME}.dSYM/Contents/Info.plist"
else
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBuild" "${TARGET_BUILD_DIR}/${INFOPLIST_PATH}"
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBuild" "${BUILT_PRODUCTS_DIR}/${WRAPPER_NAME}.dSYM/Contents/Info.plist"
fi
echo "Updated ${TARGET_BUILD_DIR}/${INFOPLIST_PATH}"
{% endhighlight %}
