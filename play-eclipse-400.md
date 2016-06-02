# Resolving Eclipse import problems from Typesafe Activator projects

When you're setting up a new Play project with Typesafe Activator your may face the problems I did when opening the project in Eclipse.

--

![Eclipse doesn't know the view objects.](http://wellnr.de/file/rethinkit/play_classpath_error.png)

I had to made a few changes to the workspace and the project configuration before I was able to start my development:

* **Eclipse Settings:**
	* `Preferences - General - Workspace`; Enable `Refresh native hooks or polling` and disable `Build automatically`. The automatic build should be disabled since this will be done by SBT during your development.

* **Project Settings:**  
We need to change the build path, because the compiled views are not contained on it after creating the Eclipse project with sbt-eclipse/ Activator.

	* `Properties - Java Build Path - Source`; Remove the source folder `target/scala-2.11/src_managed/main`.
    * `Properties - Java Build Path - Libraries`; Add a class folder `target/scala-2.11/classes_managed`.

After I've made these changes I was able to work with that project as usual.
