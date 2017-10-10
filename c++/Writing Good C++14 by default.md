Cpp Core Guidelines:

https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md

Based off of: https://www.youtube.com/watch?v=hEx5DNLWGgA

Look into GSL (Guideline support library) and see how it can be used!

### Type and memory safety

- Type: Avoid unions, use *variant*
- Bounds: Avoid pointer arithmetic, use array_view
- Lifetime: Dont leak (forget to delete,) Dont corrupt (dont double delete), dont dangle (dont return local reference).

##### Target guarantee

C++ compiled with  safe subset of GSL is never root cause of type and memory errors except where explicitly annotated.

Used for auditing code for whenever there *is* a type safe error because of the annotations. Made for making code safely by construction or because of the way that you wrote it. It sometimes occurs a runtime penalty.



## Safety profiles

These are made to bring a set of deterministic and portable rules (made for other compilers and architectures) and are guaranteed to achieve a specific type or memory or both safety.

Basically allows you to only opt in to what unsafe operations that you want to do and will still prevent you from doing others. Allowing you to avoid for example only allow raw pointers for alloc calls.

This is the process of taking out unsafe parts of the language while adding suitable alternatives for the features taken out. For example with GSL there is no reinterpret_cast or static_cast downwards.



# Type safety overview

1. Don't use reinterpret_cast
2. Don't use static_cast downcasts (cast from parent to derived). Use dynamic_cast (look it up too)
3. Don't use const_cast to cast away const at all!
4. don't use C-style casts that that would perform a reinterpret_cast, static_cast downwards, or const_cast.
5. Don't use as local variable before it has been initialized.
6. Always initialize a member variable.
7. Avoid accessing members of raw unions. Perfer variant instead.
8. Avoid reading from varargs or passing vararg arguments. perfer variadic template parameters instead (study this more!).

###  Safety profiles pt2. (Bounds safety)

- Array_view<T, Extents>: A view of contiguous T objects, replaces (*,len)

- string_view<CharT, extent>: convenience alias for a 1-d array_view

  - These are the only types with runtime cost

  With these bound safety guidelines don't use pointer arithmetic, only index into arrays using constant expressions, don't use array-to-pointer decay and finally don't use std::funtions and types that are not bounds-checked.

##### Example

Before:

```c++
void f(__In_reads_(num)Thing* things, unsigned count) {
    unsigned totalSize = 0;
  for (unsigned i = 0; i <= count; ++i)
    totalSize += things[i].GetSize();
  
  memcpy(dest, things, count); // incorrect param count
}

void caller() {
    Things things[10]; // Uninitialized memory
  
  	f(things, 5);
  	(f(things, 30)); // Wrong size and not caught at compile time
}
```

After:

```c++
void f(array_view<const Thing> things) {
    unsigned totalSize = 0;
  	for(auto& thing : things)
      totalSize += thing.GetSize();
  
  	copy(things, dest);
}

void caller() {
    Thing things[10]; // unitit data
  	f(things); // length ok but unitit compile error
  	f({things, 5}); // Good and convenient
  	f({things, 30}); // Compile time error bitches
}
```



#### Using types in/with old code

New types work with the old/existing code so we can choose when to adopt them incrementally.

Example:

```c++
void f(array_view<int> av);

// All valid calls
std::vector<int>& vec; f(vec);
int* p; size_t len;    f({p, len});
std::array<int>& arr;  f(arr);
```

Different string representations also can be merged into string_view and wstring_view:

```c++
void f(wstring_view s);

std::wstring* s;        f(s);
wchar_t* s, size_t len; f({s, len});
QString s;				f(s);
CStringA s;				f(s);
UnicodeString s;		f(s);
// More exist...
```

All bounds:

Array_view<> and string_view<> along with ranges allow no accesses beyond the bounds of an allocation. No pointer arithmetic.

# Lifetime safety

Easy to state, but its delete every heap object once, and only once and don't dereference to a deleted object. 

Examples of the problem:

- Handle only C because its simpler
- Incur run-time overheads (GC)
- Rely on whole-program analysis
- Require extensive annotation (Rust, difficult)
- Invent a new language

### Remember pointers are not evil

Smart points have usage (for encapsulating ownership) but raw T* and T& are very efficient compared to smart pointers.



### Lifetime safety overview

- Indirection concept:
  - Owner (can't dangle): owner<>, containers, smart pointers, ...
  - Pointer (could dangle); *, &, iterators, array_view/string_view, ranges ...
  - not_null<T>: Wraps any Indirection and enforces non-null
- owner<>:  ABI-compatible, building block for smart pointers and containers



Rules:

1. Prefer to allocate heap objects using make_unique/make_shared or containers.
2. Otherwise, use owner<> for source/layout compatibility with old code. Each non-null owner<> must be deleted exactly once or moved.
3. Never dereference a null or invalid Pointer.
4. Never allow an invalid Pointer to escape a function.

## Approach

Local rules, static enforce where no runtime overhead and whole-program guarantees if we build the whole program. Identify owners and track pointers to enforce leak-freedom for owners. Track points to for pointers.

Only a few annotations where we infer owner and pointer types. (Contains an owner -> is owner, else contains a poiner -> pointer).

## Principles

1. A pointer tracks its pointee(s) and must not outlive them.
2. Track the outermost object. For example if you have a member from a class that you use, you have to track the class (the outermost object pertaining to what you want). If you have a reference to an array element, you have to track the array too. Finally if you have a heap object you track its owner
3. Pointer parameters are valid for the function call & independent by default. You enforce in the caller for a function to prevent passing a pointer to the callee could invalidate.
4. A pointer returned from a function is derived from its inputs by default. If you have a function that returns a raw pointer for an object that doesn't come from something you passed in, it better be some type of smart pointer or owner<t>. Enforced in callee.

### Example: Pointer to a local

```c++
int* p1 = nullptr, *p2 = nullptr, *p3 = nullptr; // They all point to null
{
    int i = 1;
  	struct mystruct { char c; int i; char c2;} s = {'a', 2, 'b'};
  	array<int> a = {0, 1, 2, 3, 4 ,5 6, 7, 8, 9};
  
  	p1 = &i;
  	p2 = &s.i;
  	p3 = &a[3];
  
  	*p1 = *p2 = *p3 = 42; // All are valid pointers
}
```

This is terrible, because as soon as the code block is exited, all of the pointers p1-p3 are deleted. You can use static analysis tools to help us avoid invalidations like this.

### Example: address-of, and pointer to pointer

Taking the address of any object including an owner or pointer

```c++
int i = 1;    	// Non-pointer
int& ri = 1;	// ri points to i
int* pi = &ri;	// pi points to i

int** ppi = &pi; // ppi points to pointer pi

auto s = make_shared<int>(2);
auto* ps = &s; // ps points to Owner s
```

### Example: Dereferencing

```c++
int i = 0;
int* pi = &i; // pi points to i
int** ppi = &pi; // ppi points to pi

int* pi2 = *ppi; // IN: ppi points to pi, pi points to i, *ppi eventually must point to i

int j =  0;
pi = &j; // pi points to j - **ppi points to j. IN: ppi points to pi, pi points to j therefore *ppi points to j

pi2 = *ppi;
```

### Example: pointer from owner

Getting a pointer from an owner:

```c++
auto s = make_shared<int>(1);
int* p = s.get(); // p points to s` = an object
				  // owned by s (current value)
*p = 42;		  // OK, p is valid

s = make_shared<int>(2); // A: modify s -> invalidate p

*p = 43; // Error, p was invalidated by assignment to s at line A
```

### Example: unique_ptr bug

This code compiles but rA contains garbage. Can someone explain to me why this code invalid

```c++
unique_ptr<A> myFun(){
    unique_ptr<A> pa(new A());
  	return pa; // call this returned object temp_up ....
}

const A& rA = *myFun(); // *temp_up points to temp_up` == "owned by temp_up"
						// So this means that this is garbage data now that its deleted...
use(rA);

// FIX: Here is the fix, store in local auto variable before putting into const reference
auto p = myFun();
const A& rA = *p;
use(rA);

// FIX2: Temporary lifetime extension
const auto& rA = myFun();
use(*rA);
```

### Example: shared_ptr<vector<int>>

```c++
auto sv = make_shared<vector<int>>(100);
shared_ptr<vector<int>>* sv2 = &sv; // sv2 points to sv
vector<int>* vec = &*sv; // vec points to sv`
int* ptr = &(*sv)[0]; // ptr points to sv``

*ptr = 1; // OK

vec->push_back(1); // Modifying sv` invalidates sv``
*ptr = 2; // Error: ptr was invalidated by "push_back" on line A

ptr = &(*sv)[0];

(*sv2).reset(); // B: modifying sv invalidates sv`
vec->push_back(1); // ERROR: vec was invalidated by reset on line B
*ptr = 3; // ERROR: vec was invalidated by reset on line B
```

- Branches add the possiblity of "or" : p can point to x or y
- Loops are like branches: if exit set != entry set. process loop body once more
- Points to null removed in a branch that tests against null pointer constant
  - p = cond ? x : nullptr; *p = 42; // Error: p could be null...
- try/catch: treat a catch block as if it could have been entred from every point in the try block where an exception could have been raised.
  - This allows to record all potential invalidations in the try block, and remove any revalidations in the try block (potentially none were executed)
  - Its conservative.

# Function calling

```
T* p = ...;
f(p);
```

Where did that pointer come from?

- In callee, assume pointer params are valid for the call, and independent.
  - void f(int* p) { /// } In f, assume ps is valid for its lifetime
- In caller, enforce no arguments that we know the callee can invalidate.

```c++
void f(int*);
void g(shared_ptr<int>&, int*);
shared_ptr<int> gsp = make_shared<int>();
int main() {
    f(gsp.get()); // ERROR: arg points to gsp` and gsp is modifiable by f
  	auto sp = gsp;
  	f(sp.get()); // OK arg points to sp`, and sp is not modifiable by f
  	g(sp, sp.get()); // ERROR, arg2 points to sp`, and sp is modifiable by f...
  	g(gsp, sp.get()); // ok, arg2 points to sp`, and sp is not modifiable by f
}
```

What is happening here is that we shouldn't pass pointers that could be invalidated as raw pointers if we don't own them locally (I.e scope). This also goes for scope of functions, see arg2 points to sp`, and sp is modifiable by f... Look at ownership relationships.

At the top of any call tree where you will pass down modifiable pointers, take a local copy of the shared_ptr then pass that copy down the call tree!

_This is the #1 correctness issue for smart pointers_ 

Remember: dont forget about reentrancy.

Now in static analysis tools it can figure out invalid iterator ranges

# Pointers from functions

- In principle, you have to "state" the lifetime of a returned Pointer.

  - Caller assumes that lifetime.
  - Callee enforces that lifetime when separately compiling callee body.

- Defaults are to minimize the frequency that you have to state it explicitly, so that the most of the time you state it the convenient way:

  - Vast majority of returned pointers are derived from Owner and Pointer inputs
  - If there are no inputs (sg singletons) we assume you're returned a pointer to something sattic. this handles singleton instance functions
  - Only if it's "something else": Clear error when compiling...

  ### Calling functions return/out lifetimes

  A returned pointer is assumed to come from Owner/Pointer inputs.

  - Vast majority of cases: Derived from Owner and Pointer arguements.
    - int* f(int* p, int* q); // ret points to *p or *q
    - char* g(string& s);  // ret points to s` (s-owned)
  - Params that are owner rvalue weak magnets: owner const& parameters
    - Ignored by default, because owner const& can bind to temporary owners.
      - char* find_match(string& s, const string& sub); // ret points to s`
    - only if there are no other candidates, consiuder owner weak rvalue magnets.
      - const char* point_into(const string& sub); // ret points to sub`
  - Params that are owner rvalue string magnets: owner&& parameters
    - Always ignored, because owner&& strongly attracts temporary owners.
      - int* find_match(unique_ptr<X>&&);

### Example: find_match

```c++
char* // defaults points to s`
find_match(string& s, const string& sub);
string str = "xyzzy", z = "zzz";
p = find_match(str, z); // p points to str`
p = find_match(str, "literal"); // p poitns to str`
p = find_match(str, z+"temp"); // p points to str`
p = find_match(str, "UDL"s); // p points to s`

char* find_match(string& s, const string& sub) {
    if (,..) return &s[i];
  	if (...) return &sub[j]; // ERROR, {s`} is not a subset of {sub`}
  	char* ret = nullptr; // ret points to null
  	if (...) ret = &s[i];
  	else ret = &sub[i];
  
  	return ret; // ERROR, {s`} is not a subset (in this case) {s` OR sub`}
}
```

### Example: std::min and max

An example of the points above in action is std::min and max which return from their const T& members.

```c++
template<class T>
  const T& min(const T& a, const T& b) { return b<a ? b : a; }

int x = 10, y = 2;
int& ref = min(x, y); // OK prints 2
cout << ref;

int& bad = min(x, y+1); // Trap! this is data-dependent bug and wouldn't fail in Max case...
						// bad will point to a deleted temporary object...
```

