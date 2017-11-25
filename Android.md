
Android Internals Series 


Services
===========
- IntentService is subject to all the background execution limits imposed with Android 8.0 (API level 26). In most cases, you are better off using JobIntentService, 
  which uses jobs instead of services when running on Android 8.0 or higher.

- Context#startService : Each call to startService() results in significant work done by the system to manage service lifecycle surrounding the processing of the intent,
  which can take multiple milliseconds of CPU time. Due to this cost, startService() should not be used for frequent intent delivery to a service, and only for scheduling 
  significant work. Use bound services for high frequency calls. 

- Service#StartForeground: Note that calling this method does not put the service in the started state itself, even though the name sounds like it.  
  You must always call startService(Intent) first to tell the system it should keep the service running, and then use this method to tell it to keep it running harder. 
     
- onStartCommand : Note that the system calls this on your service's main thread.  A service's main thread is the same thread where UI operations place for Activities running in the
  same process.  You should always avoid stalling the main thread's event loop.  When doing long-running operations,
  network calls, or heavy disk I/O, you should kick off a new thread. 
     
     
Important Concepts:
- Distinction between bound and unbound services
- Distinction betweem background and foreground services

TODO: take a look at services sample 


Usefull Code Commits 
=============================

* [22/08/2009] One of the problems I have been noticing is background services sitting around running and using resources. 

Some times this is due to the app developer doing this when they shouldn't, but there are also a number of issues with the current Service 
interaction model that make it very difficult (or impossible) to avoid getting services stuck in the started state. 

a new Service.onStartCommand() that allows the service to better control how the system should manage it. 

The key part here is a new result code returned by the function, telling the system what it should do with the service afterwards:  

   - START_STICKY is basically the same as the previous behavior, where we usually leave the service running. The only difference is that it if it gets restarted because its process is killed, 
     onStartCommand() will be called on the new service with a null Intent instead of not being called at all.
  
    - START_NOT_STICKY says that, upon returning to the system, if its process is killed with no remaining start commands to deliver, then the service will be stopped instead of restarted. 
      This makes a lot more sense for services that are intended to only run while executing commands sent to them.  
  
    - START_REDELIVER_INTENT is like START_NOT_STICKY, except if the service's process is killed before it calls stopSelf() for a given intent, that intent will be re-delivered to it until it completes 
      (unless after 4 or more tries it still can't complete, at which point we give up). 

* [06/08/2013] Start restricting service calls with implicit intents.  
  
  The bindService() and startService() calls have always had undefined behavior when used with an implicit Intent and there are multiple matching services. 
  Because of this, it is not safe for applications to use such Intents when interacting with services, yet the platform would merrily go about doing... something.  
  
  In KLP I want to cause this case to be invalid, resulting in an exception thrown back to the app. Unfortunately there are lots of (scary) things relying on this behavior, 
  so we can't immediately turn it into an exception, even one qualified by the caller's target SDK version.  In this change, we start loggin a WTF when such a call happens, 
  and clean up some stuff in Bluetooth that was doing this behavior 

* [16/09/2013] We now have the activity manager kill long-running processes during idle maintanence.
  This involved adding some more information to the activity manager about the current memory state, so that it could know if it really should bother killing anything. 
  
  While doing this, I also improved how we determine when memory is getting low by better ignoring cases where processes are going away for other reasons (such as now idle maintenance). 
  We now won't raise our memory state if either a process is going away because we wanted it gone for another reason or the total number of processes is not decreasing.
  The idle maintanence killing also uses new per-process information about whether the process has ever gone into the cached state since the last idle maintenance, 
  and the initial pss and current pss size over its run time.

* [26/03/2017] API refactor: context.startForegroundService() 
   Rather than require an a-priori Notification be supplied in order to start a service directly into the foreground state, we adopt a two-stage compound operation for undertaking ongoing 
   service work even from a background execution state. Context#startForegroundService() is not subject to background restrictions, with the requirement that the service formally enter the 
   foreground state via startForeground() within 5 seconds. If the service does not do so, it is stopped by the OS and the app is blamed with a service ANR.  
   
   We also introduce a new flavor of PendingIntent that starts a service into this two-stage "promises to call startForeground()" sequence, so that deferred and second-party launches can take advantage of it. 


