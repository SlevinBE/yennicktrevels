---
layout: post
title: "Automated Gradle project deployment to Sonatype OSS Repository"
date: 2013-10-11 21:53
comments: false
categories: 
---

This post will walk you through the steps to deploy your gradle project to the Sonatype OSS staging repository.

<!--more-->

### PREREQUISITES

* Minimum Gradle 1.0 milestone 5
* Gpg4zin, you can find the installer on this website: http://www.gpg4win.org/

### CONFIGURING YOUR BUILD.GRADLE FILE
First you'll need to configure your build script (build.gradle file). A full working build script of my GradleFx project can be found here:  
https://github.com/GradleFx/GradleFx/blob/develop/build.gradle

These are the steps:

Apply the maven and signing plugins. The signing plugin will sign your jar and pom files, which is required for deployment to the Sonatype OSS repository.

	apply plugin: 'maven'
	apply plugin: 'signing'
{:lang="groovy"}

Specify the group and version of your project:

	group = 'org.gradlefx'
	version = '0.3.1'
{:lang="groovy"}

Define tasks for source and javadoc jar generation. These jars are also required for Sonatype:

	task javadocJar(type: Jar, dependsOn: javadoc) {
		classifier = 'javadoc'
		from 'build/docs/javadoc'
	}

	task sourcesJar(type: Jar) {
		from sourceSets.main.allSource
		classifier = 'sources'
	}
{:lang="groovy"}

Specify your artifacts:

	artifacts {
		archives jar

		archives javadocJar
		archives sourcesJar
	}
{:lang="groovy"}

Configure the signing task by specifying which artifacts to sign, in this case all the artifacts in the archives configuration. This will not yet include the pom file, which will be signed later.

	signing {
		sign configurations.archives
	}
{:lang="groovy"}

Then we need to configure the generated pom file, sign the pom file and configure the Sonatype OSS staging repository.  
The **'beforeDeployment'** line will sign the pom file right before the artifacts are deployed to the Sonatype OSS repository.  
The **'repository'** part configures the Sonatype OSS staging repository. Notice the sonatypeUsername and sonatypePassword variables, these are property variables to which I'll come back to later in this post.  
The **'pom.project'** section configures the generated pom files, you need to specify all this information because it's required for the Sonatype repository (and Maven Central).

	uploadArchives {
		repositories {
			mavenDeployer {
				beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

				repository(url: "https://oss.sonatype.org/service/local/staging/deploy/maven2/") {
				  authentication(userName: sonatypeUsername, password: sonatypePassword)
				}

				pom.project {
				   name 'GradleFx'
				   packaging 'jar'
				   description 'GradleFx is a Gradle plugin for building Flex and Actionscript applications'
				   url 'http://gradlefx.github.com/'

				   scm {
					   url 'scm:git@github.com:GradleFx/GradleFx.git'
					   connection 'scm:git@github.com:GradleFx/GradleFx.git'
					   developerConnection 'scm:git@github.com:GradleFx/GradleFx.git'
				   }

				   licenses {
					   license {
						   name 'The Apache Software License, Version 2.0'
						   url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
						   distribution 'repo'
					   }
				   }

				   developers {
					   developer {
						   id 'yennicktrevels'
						   name 'Yennick Trevels'
					   }
					   developer {
						   id 'stevendick'
						   name 'Steven Dick'
					   }
				   }
			   }
			}
		}
	}
{:lang="groovy"}  

That's it for the configuration of your build script.

### INITIAL SETUP
There are a couple of thing you need to do before you can run the build automatically, like:
Register your project at Sonatype
Create a public key pair to sign your jar/pom files
Create a gradle properties file

#### Register your project at Sonatype

First you'll need to register your project at sonatype. Follow the "2. Sign Up" and "3. Create a JIRA ticket" sections on this page: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide

#### Create a public key pair

A public key pair is needed to sign your jar/pom files. To generate this we need to dive into command line. Follow the "Generate a key pair" and "Distributing your public key" sections on this page: https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven
This will generate a key that will be stored on your computer and on sks-keyservers.net. Remember the password because there is no way to reset or restore it when you forget it.

#### Create a gradle properties file

Now we'll have to create a gradle.properties file which will contain the public key and sonatype login information.  
Because this is sensitive information it's best to store this file in your ".gradle" directory (e.g. D:\Users\MyUsername\.gradle) and not under your project.  
This properties file is used by your gradle build script. The signing.* properties are used and defined by the signing task, the sonatypeUsername and sonatypePassword properties are the variables we defined earlier in our build script.  

So first create the "gradle.properties" file under D:\Users\MyUsername\.gradle  
Then add the following properties to the file and change their values:


	signing.keyId=A5DOA652
	signing.password=YourPublicKeyPassword
	signing.secretKeyRingFile=D:/Users/MyUsername/AppData/Roaming/gnupg/secring.gpg

	sonatypeUsername=YourSonatypeJiraUsername
	sonatypePassword=YourSonatypeJiraPassword
{:lang="plain"}

the signing.keyId property is the public key id, you can list your keys with the "gpg --list-keys" command. With this command you'll get an output like this:

	pub 2048R/A5DOA652 2011-10-16
	uid MyUid
	sub 2048R/F66623G0 2011-10-16
{:lang="plain"}

That's it for the initial setup.

### DEPLOYMENT TO THE SONATYPE OSS REPOSITORY
Once you've done all the previous steps in this post you can deploy your project with just one single command: "gradle uploadArchives"
