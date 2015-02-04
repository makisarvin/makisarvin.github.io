---
layout: post
title: "Managing project dependencies using SBT"
date: 2015-02-02 09:17:18 +0000
comments: false
categories: 
---

More and more projects nowdays are using the microservices concept. Instead of having a large multi-module project, we prefer small projects that do one thing and we compose those in order to create bigger applications. Most of these projects share the same structure and the same libraries. For example we could have a number of REST services implemented with akka and spray talking to each other and to a SPA frontend like AngularJS. 

Each project has the same settings and the same dependencies. Do we have to copy them over to each project? Is there a better way to manage those dependencies faster? Can we create a sbt plugin that has all the settings for us and use that?


First lets see how we would go to solve this problem withough using a plugin. Normally we will start with seperating all the dependencies from the build.sbt. So lets create a simple project and start

We will use a spray application as our example, so lets go ahead and clone the spray starter template. 

{% codeblock %}
git clone https://github.com/spray/spray-template.git hello-spray
{% endcodeblock %}

As we can see all the dependencies are in `build.sbt` file.

So first lets move the dependencies to another file are per suggestion of the [sbt documentation](http://www.scala-sbt.org/0.13/tutorial/Organizing-Build.html):


	
{% codeblock lang:scala project/Dependencies.scala %}	
import sbt._

object Dependencies {
  // Versions
  lazy val akkaVersion = "2.3.6"
  lazy val sprayVersion = "1.3.2"
  lazy val speac2Version = "2.3.11"

  // Libraries
  val akkaActor = "com.typesafe.akka" %% "akka-actor" % akkaVersion
  val akkaTestKit = "com.typesafe.akka" %% "akka-testkit" % akkaVersion
  val sprayCan = "io.spray" %% "spray-can" % sprayVersion
  val sprayRouting = "io.spray" %% "spray-routing" % sprayVersion
  val sprayTestKit = "io.spray" %% "spray-can" % sprayVersion
  val specs2core = "org.specs2" %% "specs2-core" % speac2Version

  // Projects
  val backendDeps = Seq(
  	akkaActor, 
    sprayCan, 
    sprayRouting, 
    sprayTestKit % Test, 
    akkaTestKit % Test, 
    specs2core % Test
  )

}
{% endcodeblock %}

Now going back to `build.sbt` we change the `libraryDependencies` to :

{% codeblock lang:scala build.sbt %}
libraryDependencies ++= Dependencies.backendDeps
{% endcodeblock %}

We can do the same with every other setting, Put them in a Common.scala and import them as settings. 
So an easy solution is to copy and paste the Dependecies.scala to each project, or better yet link this file from a central location. is this all?

Well, if we look into the projects, we see that it's not only the library dependencies that need to be shared but various other settings. So lets move those settings into another common file. 

	
{% codeblock lang:scala project/Common.scala %}
import sbt._
import sbt.Keys._
import spray.revolver.RevolverPlugin.Revolver

object CommonSettings {

  val commonSettings = Revolver.settings ++
    List(
      // Core settings
      organization := "com.example",
      version := "0.1",
      scalaVersion := "2.11.2",
      scalacOptions ++= List(
        "-unchecked",
        "-deprecation",
        "-language:_",
        "-encoding", "UTF-8"
      )
    )
}
{% endcodeblock %}

and build.sbt will change into:

{% codeblock lang:scala build.sbt %}
name  := "hello-spray"

Common.commonSettings

libraryDependencies ++= Dependencies.backendDeps
{% endcodeblock %}

Now we have two files that need to shared each time, and as the project grows, more requirements increase the complexity. For example, what is we create new tasks or settings?

To improve upon our solution, we can try to create a plugin to keep all this information. 
The end goeal will be that each project will use the plugin to manage their settings and dependencies. When we update the plugin all the project would get the new version, thus the new settings. 

So lets create a new sbt plugin. 

A sbt plugin has the same structure with a scala project. So lets create a new sbt project. 

Open `build.sbt` and enter the following code:

{% codeblock lang:scala build.sbt %}
version := "1.0"

sbtPlugin := true

organization := "com.example"

name := "myplugin"

resolvers in ThisBuild ++= Seq (
	"ReactiveCouchbase Snapshots" at "https://raw.github.com/ReactiveCouchbase/repository/master/snapshots/",
	"Typesafe Releases" at "http://repo.typesafe.com/typesafe/releases/"
)

addSbtPlugin("com.typesafe.sbt" % "sbt-scalariform" % "1.3.0")

addSbtPlugin("io.spray" % "sbt-revolver" % "0.7.2")

addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.12.0")
{% endcodeblock %}

So our plugin will do three things:

* Manage the settings for all our projects
* Manage the library dependencies for all our projects
* Manage the plugins that our projects are using. 

Because this is a plugin we add the plugins we will expose to project in the `build.sbt` and not in the `plugins.sbt` file.
Another important line is the `sbtPlugin := true` this denotes this project as a plugin project.

Now lets add our code.

{% codeblock lang:scala src/main/scala/Library.scala %}
package myplugin

import sbt._

object Version {

  val projectVersion = "1.0.0"

  val akka = "2.3.8"
  val logback = "1.1.2"
  val scala = "2.11.4"
  val scalaParsers = "1.0.1"
  val spray = "1.3.2"
  val scalaLogging = "3.1.0"
  val scalaTest = "2.2.0"
  val specs2 = "2.3.11"

}

object Library {
  val akkaActor = "com.typesafe.akka" %% "akka-actor" % Version.akka
  val akkaCluster = "com.typesafe.akka" %% "akka-cluster" % Version.akka
  val akkaContrib = "com.typesafe.akka" %% "akka-contrib" % Version.akka
  val akkaPersistence = "com.typesafe.akka" %% "akka-persistence-experimental" % Version.akka
  val akkaSlf4j = "com.typesafe.akka" %% "akka-slf4j" % Version.akka
  val sprayRouting = "io.spray" %% "spray-routing" % Version.spray
  val sprayCan = "io.spray" %% "spray-can" % Version.spray
  val sprayTestKit = "io.spray" %% "spray-testkit" % Version.spray
  val akkaTestkit = "com.typesafe.akka" %% "akka-testkit" % Version.akka
  val logbackClassic = "ch.qos.logback" % "logback-classic" % Version.logback
  val scalaLogging = "com.typesafe.scala-logging" %% "scala-logging" % Version.scalaLogging
  val scalaParsers = "org.scala-lang.modules" %% "scala-parser-combinators" % Version.scalaParsers
  val scalaTest = "org.scalatest" %% "scalatest" % Version.scalaTest
  val specs2 = "org.specs2" %% "specs2-core" % Version.specs2

}
{% endcodeblock %}

We splitted the dependency object we created before to Version and Library and added a few more libraries. 

What we need now is an object that will hold our settings. The projects settings can be split logicaly into the following categories:

* Project settings like organization, plugin settings etc
* libraryDependencies. 
* Resolvers

So lets create an object that has those three categories 

{% codeblock lang:scala src/main/scala/CommonSettings.scala %}
package myplugin

import com.typesafe.sbt.SbtScalariform._
import sbt.Keys._
import sbt._
import spray.revolver.RevolverPlugin.Revolver

import scalariform.formatter.preferences.{PreserveDanglingCloseParenthesis, DoubleIndentClassDeclaration, AlignSingleLineCaseStatements}

object CommonSettings {
  val commonSettings =
    scalariformSettings ++ Revolver.settings ++
      List(
        // Core settings
        organization := "com.hotelier.frontdesk",
        version := Version.projectVersion,
        scalaVersion := Version.scala,
        scalacOptions ++= List(
          "-unchecked",
          "-deprecation",
          "-language:_",
          "-encoding", "UTF-8"
        ),
        // Scalariform settings
        ScalariformKeys.preferences := ScalariformKeys.preferences.value
          .setPreference(AlignSingleLineCaseStatements, true)
          .setPreference(AlignSingleLineCaseStatements.MaxArrowIndent, 100)
          .setPreference(DoubleIndentClassDeclaration, true)
          .setPreference(PreserveDanglingCloseParenthesis, true)
      )


  import Library._

  val commonLibraryDependencies: Seq[ModuleID] = Seq(
    sprayCan,
    sprayRouting,
    akkaActor,
    akkaSlf4j,
    sprayTestKit % "test",
    akkaTestkit % "test",
    specs2 % "test"
  )

  val commonResolvers: Seq[MavenRepository] = Seq(
    "ReactiveCouchbase Snapshots" at "https://raw.github.com/ReactiveCouchbase/repository/master/snapshots/",
    "Typesafe Releases" at "http://repo.typesafe.com/typesafe/releases/"
  )
}
{% endcodeblock %}

So now we have a `CommonSettings` object with three vals that define all our settings. 

Finally we need to create our plugin:

{% codeblock lang:scala src/main/scala/CommonProjectSettings.scala %}
package myplugin

import sbt.Keys._

object CommonProjectSettings extends sbt.AutoPlugin {

  import CommonSettings._

  override def projectSettings =
    commonSettings ++ Seq(libraryDependencies ++= commonLibraryDependencies) ++ Seq(resolvers ++= commonResolvers)
}
{% endcodeblock %}

Thats it. the only think we had to do was add our settings into the projectSettings by overriding the method. 

now we need to build and publish our plugin:

{% codeblock %}
	sbt compile, publish-local
{% endcodeblock %}

going back to our project, open `project/plugins.sbt`, delete everything and add the following line:

{% codeblock lang:scala project/plugins.sbt %}
addSbtPlugin("com.example" % "myplugin" % "1.0")
{% endcodeblock %}

Now go to `build.sbt` and replace the contents with:

{% codeblock lang:scala build.sbt %}
name := "hello-spray"

enablePlugins(CommonProjectSettings)
{% endcodeblock %}

Thats it. if you compile the project everything will work as before. What about if you want to change a setting for a single project? You can override whatever setting you want in the project's `build.sbt`, for example try ot override the version: `version := 2.0` and see that it works. 


