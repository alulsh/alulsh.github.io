---
layout:     post
title:      Auditing public Google Drive files
date:       2018-04-16
summary:    I wrote a Google Apps Script to help you audit your publicly shared Google Drive files.
---

## The problem with Google Drive

I love Google Drive. I love collaborating in real time and not having to email versions of files back and forth.

You can share Google Drive files with the entire internet. This is useful for crowd sourcing data or publishing content, but it's not appropriate for sensitive information. Many people aren't aware they can share Google Drive files directly with other Google users or within their G Suite domain. They instead use public URLs to share files.

On the surface, security through a random URL seems reasonable. Who's going to brute force or guess the URL? Unfortunately, this is security through obscurity. Someone could forward an email with the URL or accidentally copy and paste it into a public Slack channel. Worse, it could be cached in Google search results.

![Link sharing in Google Drive](/images/blog/drive-audit/share-with-others.png)

{: style="text-align:center"}
_Link sharing defaults to anyone with the link_

## Auditing public Google Drive files

Public Google Drive files may leak sensitive information. For some organizations banning public Google Drive files is not an option. The G Suite admin panel displays the total number of public Drive files in your organization but it doesn't list the public files themselves. How can security teams or G Suite admins audit the public Google Drive files in their G Suite domains? 

One option is a script using the [Google Drive REST API](https://developers.google.com/drive/v3/web/about-sdk) that runs under a service account with [domain-wide delegation of authority](https://developers.google.com/drive/v3/web/about-auth#perform_g_suite_domain-wide_delegation_of_authority). Although this provides a complete audit, someone has to manually review the results. While reviewing the auditor may encounter sensitive information they should not have access to.

Instead, we could distribute the work and empower users to do their own self audits. Not only does this respect document sensitivity and user privacy, it empowers users to learn more about Google Drive permissions.

## Google Apps Script

I needed a way for any user to run a self audit without having to install, configure, or run a local script. The Google Drive UI doesn't allow you to search for publicly shared files. 

![Google Drive search](/images/blog/drive-audit/drive-search-ui.png)

{: style="text-align:center"}
_You can't search for public files in the Drive UI_

You can access Google Drive files using the [DriveApp class in Google Apps Script](https://developers.google.com/apps-script/reference/drive/drive-app) though. With a few clicks users can run their own self audits on demand.

![](/images/blog/drive-audit/google-apps-script.png)

{: style="text-align:center"}
_The audit script open in Google Apps Script_

You can check out the source code for this script on [GitHub](https://github.com/alulsh/drive-public-files). You can run it yourself right now by [opening the script](https://script.google.com/d/1BfmF_Iw728kZdTugkzDrH4FmTo0S_i78Fgt61QF55P9uuym8rPIrKIlU/edit) and clicking on the triangle run button. Check out the [README on GitHub](https://github.com/alulsh/drive-public-files#running-the-script) for detailed information on running it.

While writing this script I made several mistakes.

### Not using clasp

Google Apps Scripts are convenient. The code runs on Google's servers. You are already authenticated and don't have to register a development application and set up [OAuth 2.0](https://developers.google.com/drive/v3/web/about-auth#OAuth2Authorizing).

Unfortunately, by default you have to use the built in Google Apps Script IDE (Integrated Development Environment). Although the IDE has helpful autocompletion, it's hard to indent or comment out code and it lacks support for source control.

When developing Google Apps Scripts, use [clasp](https://github.com/google/clasp) instead. You'll still have to run `clasp open` periodically to test your scripts in the IDE, but you can use your favorite text editor and source control programs. I used Sublime Text and git.

![Using clasp in action](/images/blog/drive-audit/clasp-in-action.png)

{: style="text-align:center"}
_Clasp in action_

### Iterating through all Drive files

The initial version of my script iterated through all files in a user's Google Drive and filtered on public files:

{% highlight javascript %}
function auditFiles() {
  var files = DriveApp.getFiles();

  while (files.hasNext()) {
    var file = files.next();
    var access = file.getSharingAccess();

    if (access == 'ANYONE_WITH_LINK' || access == 'ANYONE') {
      // do things to each public file
    }
  }  
}
{% endhighlight %}

This works if you don't have a lot of files in Google Drive. If you have thousands of files you will hit the maximum execution time limit of [6 minutes](https://developers.google.com/apps-script/guides/services/quotas#current_limitations).

### Splitting the work

To work around the maximum time limit, I first tried dividing the audit in to multiple runs each under 6 minutes. 

The [`FileIterator` class provides a continuation token](https://developers.google.com/apps-script/reference/drive/file-iterator#getcontinuationtoken). You can store this token in the [properties service](https://developers.google.com/apps-script/reference/properties/) to paginate through all your files. I borrowed much of the code for this from this [StackOverflow answer](https://stackoverflow.com/a/23243756).

{% highlight javascript %}
function iterateFiles() {
  var maxFiles = 100;
  var scriptProperties = PropertiesService.getUserProperties();
  var continuationToken = scriptProperties.getProperty('continuationToken');

  if (continuationToken === null) {
    var iterator = DriveApp.getFiles();
  } else {
    var iterator = DriveApp.continueFileIterator(continuationToken);
  }
  
  for (var i = 0; i < maxFiles && iterator.hasNext(); i++) {
    var file = iterator.next();
    processFile(file);
  }

  if (iterator.hasNext()) {
    scriptProperties.setProperty('continuationToken', iterator.getContinuationToken());
  } else {
    scriptProperties.deleteProperty('continuationToken');
  }
}
{% endhighlight %}

This allowed me to process 100 files at a time and stay under 6 minutes for each run. Storing the continuation token allowed the second run to pick up where the first run left off.

Unfortunately, I still had to manually re-run the script each time. If you have thousands of files this is a lot of clicks. This is a terrible user experience. I needed a way to automate this.

### Using time-driven triggers

Google Apps Script has no event to signal the end of a script execution. You can set up [time-driven triggers](https://developers.google.com/apps-script/guides/triggers/events#time-driven_events) to run on a schedule though.

I created a trigger to run my main `iterateFiles()` function every minute. When the file iterator ran out of files I ran `deleteTriggers()` to delete all the triggers and end the script.

{% highlight javascript %}
function deleteTriggers() {
  var allTriggers = ScriptApp.getProjectTriggers();
  for (var i = 0; i < allTriggers.length; i++) {
    ScriptApp.deleteTrigger(allTriggers[i]);
  }
}

function createTrigger() {
  ScriptApp.newTrigger('iterateFiles')
    .timeBased()
    .everyMinutes(1)
    .create();
}
{% endhighlight %}

This failed spectacularly. Google Apps Script time-driven triggers are approximate (around every minute), not exact (every minute on the minute). I ended up with multiple concurrent script executions and hit the daily DriveApp quota for my script. `deleteTriggers()` never ran and I had to manually delete all triggers and cancel all executions. I was about to give up on version two of my script.

![Canceled script executions](/images/blog/drive-audit/script-executions.png)

{: style="text-align:center"}
_Manually canceled time-driven script executions_

## Searching Google Drive

I looked at the [DriveApp docs](https://developers.google.com/apps-script/reference/drive/drive-app) one more time for ideas and saw the [`searchFiles()`](https://developers.google.com/apps-script/reference/drive/drive-app#searchfilesparams) method. Although you can't search for publicly shared files in the Drive UI, you can search for files by `visibility` in the [Google Drive Rest API](https://developers.google.com/drive/v3/web/search-parameters).

The search query `'visibility = "anyoneWithLink" or visibility = "anyoneCanFind"'` did the trick! This worked quickly on my personal Google account that has ~40GB of files in Drive with thousands of files.

![Email report of audit](/images/blog/drive-audit/email-report.png)

{: style="text-align:center"}
_Public Google Drive files audit results email_

## The full script

Here's the full script, which is [available on GitHub](https://github.com/alulsh/drive-public-files/blob/master/script/getPublicFiles.js). A few things worth noting:

* I only report on files that you own (`if (fileOwner === currentUser)`), not copies of documents shared with you.
* I use several try catch statements as I've seen bad or invalid permissions on files blow up this script in the past.
* I use the built in [Logger class](https://developers.google.com/apps-script/reference/base/logger) as my data store for ease of use. You could use the [Cache Service](https://developers.google.com/apps-script/reference/cache/) or a Drive spreadsheet instead.
* I use `HtmlService.createHtmlOutput` to sanitize user provided data (Drive file names) with [Google Caja](https://developers.google.com/apps-script/reference/html/html-output#).

{% highlight javascript %}
function getPublicFiles() {

  var files = DriveApp.searchFiles('visibility = "anyoneWithLink" or visibility = "anyoneCanFind"');
  var currentUser = Session.getActiveUser().getEmail();
  
  while (files.hasNext()) {
    var file = files.next();

    try {
      var fileOwner = file.getOwner().getEmail();
    } catch(e) {
      Logger.log('Error retrieving file owner email address for file ' + file.getName() + ' with the error: ' + e.message);
    }
    
    if (fileOwner === currentUser) {
      try {
        var access = file.getSharingAccess(); 
      } catch(e) {
        Logger.log('Error retrieving access permissions for file ' + file.getName() + ' with the error: ' + e.message);
      }
        
      try {
        var permission = file.getSharingPermission();
      } catch(e) {
        Logger.log('Error retrieving sharing permissions for file ' + file.getName() + ' with the error: ' + e.message);
      }
      
      var url = file.getUrl();
      var html = HtmlService.createHtmlOutput('Google Drive document <a href="' + url + '"> ' + file.getName() 
        + '</a> is public and ' + access + ' can ' + permission + ' the document<br/>');
      Logger.log(html.getContent()); 
    }
  }
  
  var body = Logger.getLog();
  MailApp.sendEmail({
    to: currentUser,
    subject: 'Your public Google Drive files',
    htmlBody: body
  });
}
{% endhighlight %}

## Conduct your own audits

Curious to know which of your files are public? [You can run my audit script](https://script.google.com/d/1BfmF_Iw728kZdTugkzDrH4FmTo0S_i78Fgt61QF55P9uuym8rPIrKIlU/edit) on your Google account now to find out.

My script is only a starting point. The [code is on GitHub](https://github.com/alulsh/drive-public-files) - please fork it and modify it to meet your needs! [You can also create a copy in your own Google Drive](https://github.com/alulsh/drive-public-files#creating-a-copy).

Here are some ideas to get started:

* Audit files where you are not the file owner.
* Create a Drive spreadsheet instead of sending an email.
* Create an [interactive Google Apps script web app](https://developers.google.com/apps-script/guides/web) for a more familiar user experience.