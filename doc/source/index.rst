
SCRATCH
-------

evolving docs for new build stuff



workflow
--------

PREFIX will be /opt/ros/fuerte/  (thereunder, (include|lib|share)), but practice
with temp directories whereever.

cmake will find the packagename-config.cmake stuff via
CMAKE_PREFIX_PATH, which must contain the installdir above (see docs
for find_package)

for now, install python executable scripts to PREFIX/bin and library
code to PREFIX/lib/python and add PREFIX/lib/python to pythonpath
manually.

* ros_cmake is a package.  it has only cmake and python as
  dependencies.  it is the base of everything... common cmake
  infrastructure goes in there.  Maybe eventually it contains all the
  rosbuild2 cmake infrastructure.

  it is capable of e.g. finding all the lang packages listed in
  ROS_GENLANGS

  std_msgs calls generate_msg(${PROJECT_NAME} MESSAGES String.msg LANGS ${ROS_GENLANGS})
  so that packages can selectively enable/disable languages

* install genmsg, gencpp, genpy:
  * from debs -or-
  * from a checkout
  * ros_cmake should provide most of what you need
  * or they can be in your buildspace

* rosinstall file should set up the builds of all this
* rosdeps yaml files should be available for tracking dependencies (e.g. python-setuptools, cmake)

* use find_package(ros_gencpp ...) from my project which contains ros messages (quux_msgs)
  * this should work (for the user who only wants c++)
* use macros pulled in by this find_package to create message-building targets
* there are the lang-specific ones, and then then lang-generic ones that take a LANGS argument

lang-generic case:
------------------
find_package(genmsg)
generate_msgs(... LANGS ${ROS_GENLANGS}) <= comes from genmsg, uses LANGS argument to find available lanaguages

* generate code for these messages
* install this generated code in a normal cmakeish way (make install)
* in a different package (quux_user) use find_package(quux_msgs ...) to get the include paths for these messages
*  [ e.g. include_directories(${quux_msgs_INCLUDE_DIRS}) ]
* compile quux_user_chatter that creates and serializes a message from quux_msgs



debian pkgs
-----------

the 'standard' build will be parameterized on a list of languages ROS_GENLANGS=cpp;py;lua (for instance)
packages that contain messages will be dependent on genmsg and genX for x in ROS_GENLANGS

standard stack .debs will contain generated messages for langs in ROS_GENLANGS

optional packagess for other langs would be ros-DISTRO-STACK-LANG e.g. ros-fuerte-common-msgs-haskell.deb

scenarios
---------

one buildspace, with packages that generate cpp only, py only, both, and e.g. ${ROS_GENLANGS} + lua 
(can write a dummy lua generator for testing purposes)


open questions
==============

rosbuild2 currently makes .deb-per-package... not .deb-per-stack.  :(   

-debug and non-debug packages.

Dependency types
----------------

Times when you have to think about dependencies:

* Making debian source packages ``.dsc``

  * These are actually pretty light, (basically just ``debuild -S``
    but if we are going to automate some of this, there will be
    extras).  

* Making debian binary packages ``.deb``

  * Everything you build against must be listedn in ``Build-Depends``
    in control file and (therefore) also installable via dpkg.

* Building

  TWO STEPS:  configure (cmake) time, and make time.

  * Everything you build against/with (important distinction:
    "against" implies that it is installed, "with" implies that it is
    in the same buildspace") must have cmake find_package()
    infrastructure.

  * Q: what if I depend on e.g. std_msgs, which is installed on the
    system, and I am building with "pascal" in ROS_LANGS.  

* Installing

* Running

  * from build dir
  * from installed dir

* Testing 

  * Locally on developer's box
  * On CI farm (jenkins)
  * after installation

Dependency graph
----------------

Legend
------



.. graphviz::

     digraph legend {
           rankdir=LR
           node [shape=box, style=filled, color="#aaaaff"] "environment variable";
           node [shape=box, style=filled, color="#aaffaa"] "system ROS packages";
           node [shape=box, style=filled, color="#ffaaaa"] "developer ROS packages";
           node [shape=box, style=filled, color="#ffffaa"] "debian packages";
     }

Buildtime
---------

.. graphviz:: 

   digraph ros_build_mechanisms {

        rankdir=BT

        // environment variables:  build parameters
	node [shape = box, style=filled, color="#aaaaff"]; ROS_LANGS

        // packages        
	node [shape = box, style=filled, color="#aaffaa"]; rosbuild genmsg gencpp genpy;

        node [shape = box, style=filled, color="#ffffaa"]; cmake python
        node [shape = box, style=filled, color="#ffaaaa"]; roscpp std_msgs geometry_msgs

        rosbuild -> cmake;
        rosbuild -> python;

        genmsg-> ROS_LANGS;
        genmsg -> rosbuild;
        ROS_LANGS -> genpy [style=dotted];
        ROS_LANGS -> gencpp [style=dotted];

        gencpp -> rosbuild;
        genpy -> rosbuild;
        std_msgs -> genmsg;     
        std_msgs -> rosbuild;     

        geometry_msgs -> genmsg;     
        geometry_msgs -> rosbuild;     

        roscpp -> rosbuild;

        quux_msgs -> genmsg;
        quux_msgs -> std_msgs;
        quux_nodes -> quux_msgs;
        quux_nodes -> rosbuild;
        quux_nodes -> roscpp;
        
   }

On the farm
-----------

.. graphviz::

   digraph ros_end_to_end {
   
   job_generation [ label="job_generation \n(https://code.ros.org/svn/ros/stacks/ros_release/trunk/job_generation/)"];
   rosdistro_file [ label=".rosdistro \n(https://code.ros.org/svn/release/trunk/distros/electric.rosdistro)"];

   each_stack [ label="each stack" ];
   dependee_stack0 [ label="dependee stack0" ];
   dependee_stack1 [ label="dependee stack1" ];
   "ros-electric deb repo" -> each_stack;
   each_stack -> dependee_stack0;
   each_stack -> dependee_stack1;
   "ros-electric deb repo" -> rosdistro_file;
   "ros-electric deb repo" -> hudson_jobs;
   hudson_jobs -> job_generation;
   job_generation -> rosdistro_file;
   hudson_jobs -> hudson_helper;
   
   }

TODO
----

* take rospkg, walk manifest.xml files, generate dot dependency graph
  of ros-electric-* at the stack and package levels.  Install
  ros-electric- from debians and operate on them.  rospkg should be
  able to do this.


questions
---------

why does release upload a tarball which is then made into .dsc

yaml associated with this contains info used in generation of control files



three repos:  shadow, shadow-fixed, and public

shadow builds everytime there is a release

when things look stableish, shadow-fixed is triggered to attempt a rebuild of everything

shadow-fixed

a release:

https://code.ros.org/svn/release/download/stacks/object_recognition/object_recognition-0.1.5/

examine "libeigen3-dev=3.0.1-1+ros4~lucid"


* run my release script
 * .distro file gets checked in: you have commit access: https://code.ros.org/svn/release/trunk/distros/electric.rosdistro
 * creates local repo branches
 * uploads tarball and yaml file (after this point, everything proceed from this tarball)
 * creates test-on-commit-to-dev-branch jobs (by cronjob looking at .distro files)
 * triggers hudson job to build debians (debbuilder).  first source debians then automatically binaries.




farm
----

spin up a jenkins and a deb repo with a script (reproducable)
one parameter:  what repo
spawn n slaves with that same parameter

monitoring/autorepair of storm machines (300 line script)

packages.ros.org is a redirect to the 'live' repo: has no state.  can
test against behind the scenes repo and roll out with a
link-switch/other-simple-operation

proper stripping of debug symbols in .debs

job-per-deb.  current system is job-per-architecture, parameterized:  

not electric-lucid-amd64 but also *-*-*-stack.  use jenkins to chain
through them i.e. if *-*-*-ros_com succeeds then the next guy is
triggered.

it doesn't now...   easy to add:  

there is  job generation package  this uses hudson/jenkins API to configure jobs

http://www.ros.org/doc/unstable/api/job_generation/html/files.html




spin up new jenkins
go over to job generation, populate the jenkins

* NOTE:  openni, gazebo leaking:   https://code.ros.org/svn/ros/stacks/ros_release/trunk/job_generation/scripts/generate_openni.py
