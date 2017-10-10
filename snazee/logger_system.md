# Purpose

The logger system is made to be a *thread-safe* implementation of logging runtime data to a log file. We should be able to set different **detail levels** in order to log as much or as little data as needed, where logging less data is equivalent to a release version of the engine and more data is a debug build.

We should be able to create **log handles** in order to define a specific *entity* that will log data to the *log output steam* and use the *stream operators* to log primitive data or other entities with a *template specialization*. 

### Detail Levels

Here is a listing of all of the detail levels and how they should be used. We should be able to set different minimum levels for different amounts of info to be logged.

##### Info

​	This level should be used to log any information deemed non-crucial when debugging the program, basically any info that documents easily seen actions in the program like object creation/destruction, changing from one state to another, setting attributes of different entities etc...

##### Debug

​	This should be used for tracing info that would be crucial when debugging different actions like states internally managed within a method, anything that could subtlety change the program flow or entity attributes or states.

##### Warn

​	This should be used when approaching limits of resources on the executing machine, or when a known non-tested procedure is used, warning the developer of usage of code that could fail at the log location or warning the developer of excessive resource usage (time, memory, storage...)

##### Error

​	This is to be used when a fail condition for a procedure has been encountered and the procedure cannot finish. This is not to be used when the execution of the engine must be halted but when, for example, a texture cannot be loaded because it can't be found or a targeted entity has been destroyed already etc... Recoverable errors for the engine should be logged here.

##### Critical

​	This is used for when an unrecoverable error has been encountered and the engine runtime must exit. This would include initialization errors for a critical subsystem (Render system can't initialize, Audio System can't initialize, GPU/CPU not supported, dereferencing invalid/null objects/ptrs)

### Log Handles

A log handle represents an internally managed logger object that was created for logging a specific procedure/object and should be able to be used easily stringing together multiple stream operators for all different types akin to *std::cout*. The usage of the log handles should be able to be used like so:

```c++
#include "Utility/Logger.h"

class Fuck {
    LoggerHandle<Fuck> m_logger;
  
  	void SomeMethod() {
        m_logger.info << "Here is an info stream";
      	m_logger.warn << "Output to warn stream with " << this << " some object too";
      	m_logger.debug << "It should also output what function used the logger too like this:";
        // [Fuck] [SomeMethod] It should also output what function used the logger too like this:
      	m_logger.critical << "And it should add each log onto seperate lines!";
    }
};
```

