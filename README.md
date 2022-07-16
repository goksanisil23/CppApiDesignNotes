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

- 