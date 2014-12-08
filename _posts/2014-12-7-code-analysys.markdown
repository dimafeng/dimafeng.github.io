---
layout: post
title:  "Code analysis"
categories: java, code analysis, code style
---
This small post is going to tell you how do we check and control code quality in my company. I was a initiator of this approach and I want to show some cases which I implemented in scope of this problem.

Few words about our codebase - we have almost 10 000 files in main project with more than 1 000 000 LOC. Project started in early 2000 and has changed a lot of frameworks, approaches, paradigms. The age of some parts of code is about 10 years. All code is stored in *git*.

First time when we wanted to get some metrics of our code brought idea to check the entire project by [Sonar][1]. It showed a lot of errors and warnings and it was totally useless, we didn't get where to start. We just ignored this and continued to write code. But recently we got back to this problem and it was decided to go on another way. There was a point - we don't want to analyze the entire project. We need a report after *git push* which will show critical issues and will indicate incompatibility with internal code standard. Also we don't want to waste the developers' time - obviously, it should be *continous integration* task. We use [Jenkins][2] as a CI server and it was pretty easy to configure it up to our requirements. 

## Implementation

I wrote small java application which does following things:

1. Gets value from env variable **GIT_PREVIOUS_COMMIT** which is set by [git plugin][3]. Retrieves all commint which were pushed by developer. Retrieves all lines which were affected in pushed commits. Everything in this step uses standard git console utils like (git log, etc) 
2. Analyzes affected files by [checkstyle][4] and [PMD][5]
3. Build project and analyse classes by [findbugs][6]
4. Intersects lines with errors/warnings with affected lines from p.1 and generate result report
5. Send report to developer into [HipChat][7] through HipChat api

## Conclusion
As you see this approach doesn't require much time to implementation but will bring good results: compliance with code standards, some issues is found fast, relieving the burden on the codereview step. 
If you want me to explain with more details and code snippets how it works please request it in comments.

[1]: http://www.sonarsource.com/
[2]: http://jenkins-ci.org/
[3]: https://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin
[4]: http://checkstyle.sourceforge.net/
[5]: http://pmd.sourceforge.net/
[6]: http://findbugs.sourceforge.net/
[7]: https://www.hipchat.com