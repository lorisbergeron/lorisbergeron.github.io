---
layout: post
title:  "Gain in quality by automating the analysis of your web applications"
subtitle: ""
author: "Loris Bergeron"
tags: devops docker google lighthouse webapp
comments: true
---

We all know that producing and developing a web application is a complex thing. Ensuring its good performance outside a development environment, is another story, but just as complex. Fortunately, nowadays, many solutions allow you to perform performance tests in order to obtain information on the proper functioning of your web application.

For some time I have been using an open-source and complete (maybe too much at times) tool that allows to perform a whole set of tests on web applications. You can find the service online at the following adress [WebPageTest](https://www.webpagetest.org/) and the source code of the project here on [GitHub](https://github.com/WPO-Foundation/webpagetest/).

Today we will focus on a more or less comparable tool, much more flexible and fun than the one previously announced, and whose use has already been proven in production architectures. The tool on which I want to land is from an open-source project from Google and is called *[Lighthouse](https://github.com/GoogleChrome/lighthouse)*.

*Lighthouse* will allow us to test several aspects of our application, including the following: 
- Performance 
- Progressive Web App 
- Best Practices 
- Accessibility 
- SEO 

The result of this analysis is a very fast production (about 20 to 30 seconds on average) of a complete report for your web application with scores ranging from 0 to 100. The goal being to obtain 100 for all scores. When points are returned as perfectible, the documentation tells you in detail what to correct, how and why.

A point I particularly appreciate as well, you can also simulate the access to the application from a mobile or a desktop. We can also simulate different network speeds as part of the stress test and ensure that the application responds in the same way whether we use a 50Mpbs ethernet cable or a degraded 3G network.

### Using Lighthouse in Chrome DevTools

Since the latest versions of Google Chrome, *Lighthouse* is directly integrated into the Google browser. This is the easiest and fastest way to use the tool.

Here is the procedure to follow:

- Install the latest version of [Google Chrome](https://www.google.com/intl/fr_fr/chrome/)
- Open Google Chrome
- Go to the web application you want to test (in our case, this blog)
- Open the Developer Tools window (on macOS ⌥⌘J and on Windows Shift+CTRL+J)
- Select the *Lighthouse* tab

![alt text](https://github.com/lorisbergeron/lorisbergeron.github.io-assets/blob/main/2021-05-18-gain-in-quality-by-automating-the-analysis-of-your-web-applications-1.png?raw=true)

Once on this view you can activate or deactivate the different tests in the *Categories* part (personally I advise to leave everything checked). You also have the possibility to choose the type of device and also to activate plugins from the community.

Last step is to click on the *Generate Report* button and in a few seconds you will get a report of the global health of your web application. You can of course export this report in JSON or HTML format or print it with a very nice rendering.

![alt text](https://github.com/lorisbergeron/lorisbergeron.github.io-assets/blob/main/2021-05-18-gain-in-quality-by-automating-the-analysis-of-your-web-applications-2.png?raw=true)

### Using the Node CLI

The *Node CLI* provides the most flexibility in how *Lighthouse* runs can be configured and reported. Users who want more advanced usage, or want to run *Lighthouse* in an automated fashion should use the *Node CLI*.

FYI: *Lighthouse requires Node 12 LTS (12.x) or later.*

**Installation:**

```shell
$ npm install -g lighthouse
```

**Run it:** 

```shell
$ lighthouse https://lorisbergeron.github.io/
```

By default, *Lighthouse* writes the report to an HTML file. You can control the output format by passing flags. All the options at your disposal are directly accessible [here](https://github.com/GoogleChrome/lighthouse#cli-options).

### Using a Docker container (Unofficial)

The third and last way to run *Lighthouse* is to use a Docker container. An image is provided online by [justinribeiro](https://hub.docker.com/u/justinribeiro) on Docker Hub and and the sources are available on [GitHub](https://github.com/justinribeiro/dockerfiles/tree/master/lighthouse).

The steps are then greatly facilitated and can also be automated as in the case of *Node CLI*.

#### Step 1: Run the container

```shell
$ docker pull justinribeiro/lighthouse
$ wget https://raw.githubusercontent.com/jfrazelle/dotfiles/master/etc/docker/seccomp/chrome.json -O /home/lorisbergeron/chrome.json
$ docker run -itv /home/lorisbergeron/report/:/home/chrome/reports --security-opt seccomp=/home/lorisbergeron/chrome.json justinribeiro/lighthouse
```

As you can see we use a `chrome.json` file. Using the ever-awesome [Jessie Frazelle](https://twitter.com/jessfraz) SECCOMP profile for Chrome, we don't have to use the hammer that is SYS_ADMIN and this is a much better/safer solution.

#### Step 2: Run Lighthouse with --chrome-flags
Once you're in the container, using the `--chrome-flags` option available in *Lighthouse*, we can automatically start Chrome in headless mode within the container light so:

```shell
$ lighthouse --chrome-flags="--no-sandbox --headless --disable-gpu" https://lorisbergeron.github.io
```

#### Known and encountered errors

If you get a message similar to the one below:

```shell
Runtime error encountered: { Error: EACCES: permission denied, open '/home/chrome/reports/lorisbergeron.github.io.report.json'
errno: -13,
code: 'EACCES',
syscall: 'open',
path: '/home/chrome/reports/lorisbergeron.github.io.report.json' }
```

Remember to give the folder where you want to store the generated reports (in our case `/home/lorisbergeron/report`) the access rights `chmod o+w`.

**Command to execute:**

```shell
$ chmod o+w /home/lorisbergeron/report
```