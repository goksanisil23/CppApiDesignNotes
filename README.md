# C++ API Design Notes
- Data members of a class should *almost* always declared private, not public or protected. Writing setters and getters might become inconvenient and verbose however through inlining and modern compiler optimizations, the performance drawbacks are minimized. This will minimize unintended usage of the class members in the future and allows many features such as validation of set variables, notifications, syncronization in multi-threading.

- Convenience API's that combine multiple core API calls should come in a separate module, on top of the core API, in order to keep the core API pure. By this way, some users can use the extended modules whereas the users that want more control can use the core API. (e.g. OpenGL vs GLUT)

- When writing an API function that takes multiple arguments of the same type (bool,bool), the user might confuse/forget which one was which. Instead, create and pass enums which also makes readability easier 
```c++
std::string FindString(const std::string &text, SearchDirection direction, 
	CaseSensitivity case_sensitivity);

result = FindString(text, FORWARD, CASE_INSENSITIVE);
// instead of 
result = FindString(text, true, false); 
```

- **Pointer to Implementation (PIMPL)** paradigm:
Avoids exposing private details in your header files. It relies on defining a member in your class which is a pointer to a forward declared type, which contains all the actual details of the class:
```c++
// autotimer.h
class AutoTimer {
public:
	explicit AutoTimer(const std::string& name);
	~AutoTimer();

private:
	class Impl;
	Impl* mImpl;
}
```

Note that above, almost no detail or private members of the actual class is exposed to the user in the header file, hence it looks clean.

Corresponding cpp file looks like:

```c++
// autotimer.cpp
class AutoTimer::Impl {
public:
	double GetElapsed() const {
		return  std::chrono::now() - mStartTime;
	}

	std::string mName;
	auto mStartTime{std::chrono::now()};
}

AutoTimer::AutoTimer(const std::string& name) 
	: mImpl(new AutoTimer::Impl()) 
{
	mImpl->mName = name;
	mImpl->mStartTime = std::chrono::now();
}

AutoTimer::~AutoTimer() {
	delete mImpl;
	mImpl = nullptr;
}
```
Since mImpl is a private member of AutoTimer, nobody else can access the members of Impl.

*Note that in the case above, one would need to write a deep copy constructor as we have a pointer (no matter raw or smart) as a member variable. Otherwise the copied object would point to the exact same mImpl. If copying is not meant to be supported, then the pointer must be a unique_ptr.*

- **Singletons:** Useful for modeling resources that are inherently singular in nature (e.g. class to access the hardware, managers that provide single point of access to multiple resources)
It relies on creating a class with a static method that returns the same instance of the class everytime it's called. 
Since we don't want new instances or copies of singleton objects, default constructor, copy constructor, assignment op. and destructor can be defined as private.

```c++
class Singleton {
public:
	static Singleton& GetInstance(){
		static Singleton instance;
		return instance;
	}
private:
	Singleton();
	~Singletion();
	Singleton(const Singleton& );
	const Singleton& operator=(const Singleton&);
}
```

