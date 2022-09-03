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

*Note that in the case above, one would need to write a deep copy constructor as we have a pointer (no matter raw or smart) as a member variable. Otherwise the copied AutoTimer would point to the exact same mImpl. If copying is not meant to be supported, then the pointer must be a unique_ptr.*

- **Singletons:** 
Useful for modeling resources that are inherently singular in nature (e.g. class to access the hardware, managers that provide single point of access to multiple resources)
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

- **Factory Methods**
Factory classes can simplify the usage of derived classes and hide implementation details of variants from the users.
```c++
// -------------- renderer.h -------------- //
class IRenderer
{
public:
	virtual void Render() = 0;
	virtual ~IRenderer() {} // always provide a virtual destructor
};

// --------------  opengl_renderer.hpp -------------- //
#include "renderer.h"
class OpenGLRenderer : public IRenderer
{
public:
	~OpenGLRenderer() {}
	void Render() { std::cout << "OpenGL Render" << std::endl; }
};

// --------------  vulkan_renderer.hpp -------------- //
#include "renderer.h"
class VulkanRenderer : public IRenderer
{
public:
	~VulkanRenderer() {}
	void Render() { std::cout << "Vulkan Render" << std::endl; }
};

// --------------  renderer_factory.hpp -------------- //
#include "opengl_renderer.hpp"
#include "vulkan_renderer.hpp"

class RendererFactory {
public:
	IRenderer* CreateRenderer(const std::string& type) {
		if(type == "opengl") {
			return std::make_unique<OpenGLRenderer>();
		}
		if(type == "vulkan") {
			return std::make_unique<VulkanRenderer>();
		}
		return NULL;
	}
};
// -------------- main -------------- //
#include "renderer_factory.hpp"

std::unique_ptr<RendererFactory> renderer_factory = std::make_unique<RendererFactory>();

std::unique_ptr<IRenderer> ogl_renderer = renderer_factory->CreateRenderer("opengl");
ogl_renderer->Render();
std::unique_ptr<IRenderer> vulkan_renderer = renderer_factory->CreateRenderer("vulkan");
vulkan_renderer->Render();

```
Note that the Factory methods can be extended such that the user can also add a Renderer variant (say in main.cpp). This would require the factory to have a register and create functionality in order to keep track of the overall state. This fits better into the "Open/Closed Principle" which states that a class should be open for extension but closed for modification.
```c++
// -------------- main -------------- //
class DirectXRenderer() : public IRenderer
{
public:
	~DirectXRenderer() {}
	void Render() {std::cout << "DirectX Render" << std::endl;}

	std::unique_ptr<IRenderer> Create() {
		return std::make_unique<DirectXRenderer>();
	}
};

RendererFactory::RegisterRenderer("directx", DirectXRenderer::Create);
std::unique_ptr<IRenderer> directx_ren = RendererFactory::CreateRenderer("directx");
```

- **Functional Requirements & Use Cases**
Use cases describe requirements to the API from the perspective of the user
They doesn't mean all sorts of requirements gathering. 
It should only contain behavorial requirements on how user should interact with the API.
They should not propose a design, although some design elements can be derived from them.

What an official use-case doc might look like:

<img src="https://raw.githubusercontent.com/goksanisil23/CppApiDesignNotes/main/resources/use_cases_ATM" width=100% height=50%>

**Liskov Substitution Principle**
It lets you decide whether you should choose **inheritance** during class design.
*Rule* : If S is a subclass of T, it should be possible to exchange T objects with S objects without behavior change. 
An example braking this rule:
```c++
class Ellipse {
public:
	virtual void SetMajorRadius(float major);
	virtual void SetMinorRadius(float minor);
	float GetMinorRadius() const;
	float GetMajorRadius() const;
private: 
	float rMajor;
	float rMinor;
};

class Circle: public Ellipse {
public:
	void SetRadius(float r) {
		SetMajorRadius(r);
		SetMinorRadius(r);
	}
	void SetMajorRadius(float r) {
		Ellipse::SetMajorRadius(r);
		Ellipse::SetMinorRadius(r);
	}
	void SetMinorRadius(float r) {
		Ellipse::SetMajorRadius(r);
		Ellipse::SetMinorRadius(r);
	}	
	float GetRadius() const {
		return GetMajorRadius();
	}
}

void TestEllipse(Ellipse& e) {
	e.SetMajorRadius(10.0);
	e.SetMinorRadius(20.0);
	assert(e.GetMajorRadius()==10.0 && e.GetMinorRadius()==20.0);
}

Ellipse e;
Circle c;
TestEllipse(e);
TestEllipse(c); // FAILS!
```

It is generally agreed that composition should be prefered over inheritance, due to:
- Inheritance producing tighter coupling (access to protected+public members of base) whereas composition allowing looser coupling (access to only public members of base)
- Holding a pointer to base object allows forward declaration and can reduce compile time.

```c++

class Circle {
public:
	void SetRadius(float r) {
		mEllipse.SetMajorRadius(r);
		mEllipse.SetMinorRadius(r);
	}
	float GetRadius() const {
		return mEllipse.GetMajorRadius();
	}
private: 
	Ellipse mEllipse;
}

```