---
layout: post
comments: true
title: Painless Maven project releases with Maven GitFlow Plugin
categories:
- git
- maven
- git-flow
- sdlc
type: post
disqus: true
---

We recently moved all our code repositories from Atlassian's hosted SVN to [BitBucket](bitbucket.org) and took the opportunity to revisit our deployment process. We had been operating using SVN trunk for ongoing development and stabilizing it whenever we wanted to cut releases. This approach worked well in the early days but started to show its' challenges as we added  developers to the team and worked on parallel tracks. So the move to Git presented the perfect time to consider a development process that would be a better fit for a DVCS. Enter Git-Flow.
<!--more-->

## Git-Flow

Git-Flow is a popular [branching model](http://nvie.com/posts/a-successful-git-branching-model/) for Git that is aimed at structuring and streamlining the development process in a DVCS setting. If you've been living under a rock and haven't read the original article by Vincent Driessen, you should drop what you are doing right now and [go read it](http://nvie.com/posts/a-successful-git-branching-model/). Do it now! I'll wait... ;-)

![A successful Git branching model](/images/branching_model.png)

Back? So, just to recap, the main highlights are:

- Master is always clean; master head is always what's currently in production
- Development is done on a development branch or on feature branches that are eventually merged into a development branch
- Release branches are prepared from the development branch; when a release branch is ready, it's tagged and merged back into both the master and the development branch
- Emergency hotfix branches are taken from the master branch and merged back into master and development

All this branching may feel like a lot of overhead but in practice, it really isn't. Plus it pays whenever there are several team members working concurrently on different features. And thankfully, Vincent Driessen released some [Git extensions](https://github.com/nvie/gitflow) to help automate some of the branching tasks.

We had been familiar with this model for a while, so moving to Git was the perfect time to implement it. We don't find it necessary to create feature branches 100% of the time. For instance, for quick maintenance tasks, we tend to develop on the development branch. But for features that may take more than a week to develop, we do branch off as it helps us avoid stepping on each other's toes.

## Git-Flow with Maven and maven-jgitflow-plugin

If you work with Java and Maven, you are probably familiar the [maven-release-plugin](http://maven.apache.org/maven-release/maven-release-plugin/) which performs a [variety of release tasks](http://maven.apache.org/maven-release/maven-release-plugin/examples/prepare-release.html) and most importantly for [multi-module projects](http://books.sonatype.com/mvnex-book/reference/multimodule-sect-simple-parent.html) (of which we have many), takes care of bumping the sub-module versions together with the parent project version.

It is possible to use the `maven-release-plugin` together with a tool like [Git-flow](https://github.com/nvie/gitflow) but for a  smoother, more streamlined experience, the [maven-jgitflow-plugin](http://blogs.atlassian.com/2013/05/maven-git-flow-plugin-for-better-releases/) by Atlassian's [Jonathan Doklovic](http://blogs.atlassian.com/author/doklovic/) is definitely the way to go. This plugin is designed to make the Maven/Git-Flow combo experience as painless as possible. To quote from Jonathan's [own blog post](http://blogs.atlassian.com/2013/05/maven-git-flow-plugin-for-better-releases/) about the plugin, here is how it improves on the stock `maven-release-plugin` for git-flow projects:

> - doesn’t create any .backup or release.properties files (or any other files in your working tree)
> - makes all changes on branches. This means to “roll-back” you can simply delete the branch
> - automatically copies any profiles (-P) and user-properties (-D) passed on the command line to the forked maven process when building
> - doesn’t run site-deploy
> - provides the ability to completely turn off maven deployment
> - provides the ability to completely turn off remote pushes/tagging
> - Only builds your project once in the finish goal. e.g. if you do release-start and release-finish together, your tests only run once
> - Never clones your project to a temp folder
> - auto tracks origin based on maven scm values if origin tracking is not already setup (I’m looking at you Bamboo!)
> - master branch is always kept at the latest release version (not a SNAPSHOT)

Unless you want to keep cleaning up all of the above manually, using this plugin is a no-brainer.

## Configuration

The following is the configuration we have been using internally for our own deployments, annotated to explain the various flags:

```xml
[...]
<plugin>
  <groupId>com.atlassian.maven.plugins</groupId>
  <artifactId>maven-jgitflow-plugin</artifactId>
  <version>1.0-alpha20</version>
  <configuration>
    <!-- Enable this to push to origin using SSH keys -->
    <enableSshAgent>true</enableSshAgent>
    <!-- Keep your maven submodules at the same version as the parent POM -->
    <autoVersionSubmodules>true</autoVersionSubmodules>
    <!--
      Pusing in-development features to origin allows all devs to see what each other
      are working on
    -->
    <pushFeatures>true</pushFeatures>
    <!--
      This allows the CI server (e.g. Jenkins) to automatically push new releases to
      origin; you can then either manually deploy them or, if you are doing Continuous
      Deployments, auto-deploy them to prod
    -->
    <pushReleases>true</pushReleases>
    <!--
      Hot Fixes should be pushed to origin as well so that any dev can pick them up
    -->
    <pushHotfixes>true</pushHotfixes>
    <!--
      Prevents deployments from dev workstations so that they can be done by a CI
      server
    -->
    <noDeploy>true</noDeploy>
    <flowInitContext>
      <!--
        You can customize your development branch name. Why? Because it's faster to type
        'git push origin dvlp'
        than
        'git push origin develop'
        ;-)
      -->
      <developBranchName>dvlp</developBranchName>
      <!-- You can also customize your tag prefixes -->
      <versionTagPrefix>rev-</versionTagPrefix>
    </flowInitContext>
  </configuration>
</plugin>
[...]
```

More configurations options exist. You can read about them on the plugin's [wiki](https://bitbucket.org/atlassian/maven-jgitflow-plugin/wiki/Home).

## Usage

The main plugin commands are:

- `mvn jgitflow:feature-start`: creates a new feature branch; prompts for the branch name which will take the format `feature/<branch name>`; pushes the branch to origin automatically so that we can do two things: 1) collaborate with others and 2) get Jenkins to build it
- `mvn jgitflow:feature-finish`: merges a feature branch back into the dvlp branch and pushes to origin
- `mvn jgitflow:release-start`: creates a release branch (e.g. `release/rev-5.1`) and pushes it to origin
- `mvn jgitflow:release-finish`: builds, tags and merges the release branch back into master and dvlp; updates versions in master and dvlp branches
- `mvn jgitflow:hotfix-start`: works like `mvn jgitflow:release-start` but branches off of master
- `mvn jgitflow:hotfix-finish`: works like `mvn jgitflow:release-finish`

When a release branch is started the plugin prompts for the release version. E.g. if the development branch version was 5.1-SNAPSHOT it will assume that the release is 5.1 (the users can override this). When the release branch is merged back into master, the plugin automates several things:

- builds and runs the tests
- cuts a tag with the version number and a specified prefix: e.g. `tags/rev-5.1`
- updates the master branch version to 5.1 (for all parent project and all its' sub-modules)
- updates the development branch version to the next snapshot: 5.2-SNAPSHOT (users can override this)

## Conclusion

If you are using Maven and Git, I have hopefully convinced you to give the [maven-jgitflow-plugin](https://bitbucket.org/atlassian/maven-jgitflow-plugin/wiki/Home) plugin a try. Fell free to drop a comment or <a href="mailto:george@george-stathis.com">email me</a> with any questions.

## Update

Since the time of this writing, this plugin was merged into [https://bitbucket.org/atlassian/jgit-flow/](https://bitbucket.org/atlassian/jgit-flow/), so please read [https://bitbucket.org/atlassian/jgit-flow/wiki/goals.wiki](https://bitbucket.org/atlassian/jgit-flow/wiki/goals.wiki) for the latest configuration options.
