# Handbook
## Chapter 1.1. What is a Mod?

A **mod** (also known as an **extension**, **hack**, or **plugin**) is a piece of software that modifies another piece of software. Usually in games, this is done through the use of a **mod loader**; something that, well, loads mods. Some games, like Friday Night Funkin and Skyrim have built-in support for loading mods, as well as built-in interfaces for working with mods, which make modding very easy and simple.

**Geometry Dash**, unfortunately, does not have any built-in mod support. However, using very clever methods such as **DLL injection** and **hooking**, we can create an **external mod loader**. This does come with some disadvantages like the lack of a **stable API** (meaning it's not guaranteed to work between versions), but it also comes with some huge advantages, such as **fully unrestricted access** (which might also be seen as a huge disadvantage, depending on who and when you ask). **A mod can do basically anything**, within the limits of computer hardware.

But how exactly does a mod work? The actual executable file for Geometry Dash that you have installed on your computer is what is known as an **executable binary program**. If you open the game's executable with a text editor, you will see a bunch of random-looking characters. This is, in fact, the game's code; however it is _not_ its **source code**. This is **binary code**, the raw low-level instructions your CPU executes. If we had the game's source code, we could create a fork of it and add built-in mod loader support. However, as Geometry Dash is closed-source, we have to deal with modifying the binary code.

Modifying binary code is a bit difficult, however, as it is not meant to be for humans; binary code is strictly for computers, and as such it is unreadable and unmodifiable for a human. Binary code is also **platform-specific**: a mod written purely in binary would only work on one platform, and could not be ported over to others without fully rewriting the entire mod from the ground up.

As a historical curiosity, it should be noted that mods used to be written fully in binary. Or, more accurately, they were written in **assembly**, which is a (somewhat) human-readable form of binary. However, to make it very clear, **no sane person does this anymore**. Working with binary code directly is nowadays only done in rare cases, which will be explained in [Chapter 1.2](/handbook/vol1/chap1_2.md).

But what's the alternative? We can make modding much easier by knowing two facts:

 - Binary code is produced through **compilation**.
 - Geometry Dash is written in **C++**.

**Compilation is the process of turning source code into binary code**. While we can't modify GD's binary code easily, we can write our own source code in C++ and compile it into compatible binary code. Knowing what language GD was made in is vitally important, as not all binary code is equal. Some programming languages such as [Rust](https://www.rust-lang.org/) produce wildly different binary code from C++, and while it is technically compatible, it is much easier to work with a C++ game using C++. (Some modders, however, are working on making Rust usable for modding!) Writing our mods in a higher-level language like C++ also makes porting much easier; **C++ can be compiled to compatible machine code on all the platforms GD is available on**. This does come with multiple caveats though, however those are left for a later chapter.

What all of this means is if we write our mod in C++ and then find some way to make GD run the compiled binary code of it, then we have unlocked a path to modding! And luckily, we know how to make GD load extra code: **binary injection**.

Almost every platform GD is on has some sort of **dynamic library support**. On Windows, these are known as **.DLL files**; on Mac, they're **.DYLIBs**; on Android, they're **.SO files**. A dynamic library is like a binary executable, but they have one special property: other executables can load them. This is usually done through address tables and the such; these are much too complicated for the purposes of this tutorial, and the specifics are also platform-dependant and modern modloaders use proxy DLLs instead. As such, how binary injection works in detail will not be explained here. All you have to take away is that we have methods to make GD load a dynamic library, and that acts as our entry point to modding the game.

> :information_source: There is one platform that doesn't have dynamic library support: iOS (unless jailbroken). Modding on iOS without jailbreaking requires basically baking all the mods you want into a premodified game, like [iCreate](https://icreate.pro/) has done, but a general mod loader like Geode will likely never see iOS support unless some major changes to the OS happen first.

> :green_book: If you are interested in **learning more about binary injection**, [the Wikipedia article on DLL injection](https://en.wikipedia.org/wiki/DLL_injection) is a pretty good place to start. 

Usually, the first custom dynamic library we make GD load is a mod loader; that is, a mod whose purpose is to load other mods. This is because the methods we use for injection are not easily scalable, but once we have our one library running, we can invent much simpler ways of loading more of them. For example, on Windows, loading a library from another library that you control is as simple as calling the `LoadLibrary` function. This can also be done an arbitary number of times within our initial library, so we can for example automatically find all the .DLL files in some directory and load them. This is how old 2.1 mod loaders such as **Mega Hack v7** used to work: they would search for all of the `.dll` files in a predefined folder like `extensions` and call `LoadLibrary` on them.

At this point, we've gotten our library up and running, and are ready to start writing some code. However, now we enter a new problem: **how do we actually do things?**

## Chapter 1.2: Hooking & Patching

So we've got our own code running inside GD. Now we're faced with a much bigger problem however; **how do we actually do stuff?**

For example, let's say we want to do something as simple as adding a button to the main menu that displays a message when clicked. The first thing we would want to figure out is how to even add stuff to the main menu; that is, figure out when the main menu is entered, and how to add a thing to it.

We can reasonably infer that the way GD enters the main menu is by calling some function that creates it, probably in a way reminiscent of this:

```cpp
void onLoadingFinished() {
    createMainMenu();
}
```

If we could somehow listen for when this function is called, then that's problem #1 solved - we would know that now the main menu is created. Let's leave problem #2 (actually adding things to it) for later, and first focus on how to listen for that function call.

There are two fundamental tools in every GD modder's toolkit: **patching** and **hooking**.

### Patching

In Chapter 1.1 it was stated that nowadays modders rarely work with binary code directly. There are, however, some cases in which working with raw binary code is in fact the optimal solution for some functionality in a mod. In these cases, it is done through a method called **patching**, which means applying, well, patches to binary code. Patches are, however, inherently **platform-dependent** and **unportable**, so their use is highly discouraged if higher-level options are available.

Patches do, however, still play a seminal role in GD modding. For example, one of the most famous mods, **noclip**, can be achieved with [a single binary patch](https://github.com/absoIute/Mega-Hack-v5/blob/master/bin/hacks/player.json#L7) (in 2.1; in 2.2, it's a little harder). There are also some cases in complex mods where a few patches can replace writing hundreds of lines of C++ code. However, **it is very uncommon** for binary patches to be optimal. Binary patches should **never be your first solution to a problem**, but when the time comes, don't be afraid to use them if they're clearly the best solution.

Patches also serve another very important purpose: they are the base on top of which **hooking** is built. Although, you won't use them directly - Geode will handle hooking for you.

### Hooking

In contrast to patching, **hooking** is not only more portable(*-ish*) but also arguably the most important tool in a GD modder's toolkit - understanding it is vitally important for all modding.

Consider the following function:

```cpp
int addTwo(int a, int b) {
    return a + b;
}
```

This function is quite simple; it is just adding two integers together. Let's say that this function is located in some other binary, and **we just want to know whenever it's called**, and do some stuff when that happens. **This is what hooking is for**; it lets you detect when a function is called, and do something with that information.

Hooking the function would look something like this:
```cpp
int addTwo(int a, int b) {
    addTwoHook(a, b);
    return a + b;
}

int addTwoHook(int a, int b) {
    std::cout << "addTwo called!\n";
}
```

A hook **always has the signature [[Note 1]](#notes) as the function being hooked**. This means that we couldn't hook `addTwo` with something that takes two strings. Likewise, the return type has to be the same; if `addTwo` returns an `int`, so does our hook.

This is the basic premise of hooking: when the function you're hooking is called, the first thing you do is hop into your own code, and then hop back into the original once you're finished. The function that contains your own code is called a **detour**, and the function being hooked is called the **original**.

However, the code above is **actually misleading**. It would be more accurate to say that hooking does this:

```cpp
int addTwoOriginal(int a, int b) {
    return a + b;
}

int addTwo(int a, int b) {
    return addTwoDetour(a, b);
}

int addTwoDetour(int a, int b) {
    std::cout << "addTwo called!\n";
    return addTwoOriginal(a, b);
}
```

When you hook a function like `addTwo`, what first happens is that the body of the hooked function is stored somewhere else, and then the function body is replaced with a call to your detour [[Note 2]](#notes). Then in your detour, you execute your own code and then return, either by calling the original or by giving your own return value.

Notice that you are not actually required to call the original, or return its value. We could just as easily make `addTwoDetour` do something completely different, for example like this:

```cpp
// never called!
int addTwoOriginal(int a, int b) {
    return a + b;
}

int addTwo(int a, int b) {
    return addTwoDetour(a, b);
}

int addTwoDetour(int a, int b) {
    std::cout << "addTwo called!\n";
    return a - b; // not calling the original
}
```

In this case, we skip calling the original function completely, and instead return the difference between `a` and `b`. Of course, this would be quite inconvenient to anyone calling `addTwo` who is expecting the numbers to be added together, but there's not much they could do about it; we have completely overwritten the function's definition.

We could also call the original, but pass it different parameters. Let's make `addTwo` act as normal, except always passing 7 as the second parameter and ignoring `b`:

```cpp
// this now always gets 7 as the b argument
int addTwoOriginal(int a, int b) {
    return a + b;
}

int addTwo(int a, int b) {
    return addTwoDetour(a, b);
}

int addTwoDetour(int a, int b) {
    std::cout << "addTwo called!\n";
    return addTwoOriginal(a, 7);
}
```

Alternatively, let's say we want to use the result of `addTwo` ourselves. We can do this by utilizing it just like any other function call:

```cpp
int addTwoOriginal(int a, int b) {
    return a + b;
}

int addTwo(int a, int b) {
    return addTwoDetour(a, b);
}

int addTwoDetour(int a, int b) {
    int res = addTwoOriginal(a, b);
    std::cout << "addTwo returned " << res << "\n";
    return res;
}
```

Here, we first call the original to see its result, then log it into the console and then return the result as normal. We could also just return something else:

```cpp
int addTwoOriginal(int a, int b) {
    return a + b;
}

int addTwo(int a, int b) {
    return addTwoDetour(a, b);
}

int addTwoDetour(int a, int b) {
    int res = addTwoOriginal(a, b);
    std::cout << "addTwo returned " << res << "\n";
    return 0;
}
```

In this case, we can use the result of the original `addTwoOriginal` function as we please, however any callers of `addTwo` will always get 0 as a result.

### My Brain Hurts!

At this point, hooking might still feel difficult to wrap your head around. Don't worry though; hooking will be a seminal part of this whole handbook, and **you will see plenty more practical examples of it going forward**.

However, at this point it should be noted that **the syntax for hooking in Geode looks quite different** from what was shown here. The underlying premise, however, is always the same: you replace the function body with a call to your own, and then do whatever you want. You may call the original function and use its result however you'd like, or you may completely overwrite the function's behaviour by not calling the original at all.

For now, we can leave hooking be, as before we can find any practical applications for it, we must first **find some functions to hook**.

### Notes

> [Note 1] **Signature** means the parameter and return types of a function, i.e. `int addTwo(int, int)`.

> [Note 2] This, too, is not actually how hooking is implemented. Geode uses the purpose-built [TulipHook](https://github.com/geode-sdk/TulipHook) library for hooking, which does a lot of crazy stuff under the hood like automatically translating between calling conventions and other tricks to allow calling the original without needing a trampoline. For the purposes of this tutorial however, **it is easier to think of the whole function body as being replaced** instead of just a part of it.

## Chapter 1.3: Functions & Addresses

In the last chapter, we looked at hooking and how it works. However, the last chapter only touched hooking in theory. The code shown was not what actuals hooks in your code look like. For instance, we do not have access to GD's source code, so we can't exactly just write `return ourDetour()` at the start of the function we want to hook. Instead, we need to figure out some way to **insert hooks into GD's binary code**.

### Manual hooking

The main way to create hooks in Geode is using an abstraction called `$modify` - however, it is a very powerful tool and hard to explain without some preface. It is also just an abstraction; you can create hooks manually in Geode as well through the [`Mod::addHook`](/classes/geode/Mod#addHook) interface - although in practice **you should never be creating manual hooks unless necessary**.

> :information_source: Manual hooks are sometimes necessary, for example to hook some obscure low-level functions that only exists on one platform, like GLFW on Windows. However, for 99% of mods, you should just be using `$modify`, since it makes your code much more portable and easier to work with.

> :information_source: The traditional way in 2.1 of placing hooks used a library called [**MinHook**](https://github.com/TsudaKageyu/minhook). If you've been in the GD modding scene before 2.2, you've almost certainly heard of MinHook before, or at least seen its 32-bit dynamic library `minhook.x32.dll`. However, using MinHook directly is **no longer considered good practice**. While it can work perfectly fine for a single mod, MinHook has a few issues: it's Windows-only. hard to use, and if you don't link to it as a dynamic library, it will cause **hook conflicts** [[Note 1]](#notes).

Let's see a [real-world example](/tutorials/manualhooks) of creating manual hooks in Geode:
```cpp
auto wrapFunction(uintptr_t address, tulip::hook::WrapperMetadata const& metadata) {
	auto wrapped = geode::hook::createWrapper(reinterpret_cast<void*>(address), metadata);
	if (wrapped.isErr()) {{
		throw std::runtime_error(wrapped.unwrapErr());
	}}
	return wrapped.unwrap();
}

void MenuLayer_onNewgrounds(MenuLayer* self, CCObject* sender) {
    log::info("Hook reached!");
	static auto original = wrapFunction(
        geode::base::get() + 0x27b480,
        tulip::hook::WrapperMetadata{
            .m_convention = geode::hook::createConvention(tulip::hook::TulipConvention::Thiscall),
            .m_abstract = tulip::hook::AbstractFunction::from(void(*)(MenuLayer*, CCObject*)),
        }
    );
    reinterpret_cast<void(*)(MenuLayer*, CCObject*)>(original)(self, sender);
    log::info("After original!");
}

$execute {
    Mod::get()->addHook(
        reinterpret_cast<void*>(geode::base::get() + 0x27b480),
        &MenuLayer_onNewgrounds,
        "MenuLayer::onNewgrounds",
        tulip::hook::TulipConvention::Thiscall
    );
}
```

Now, this code sure is quite a jump from the hooking code in the previous chapter. If you haven't done much low-level C++, you might be confused at a lot of the syntax here. There's a bit too much to take in from the code above, so for now we will just be concentrating on a few key details.

The most important part of this code is `geode::base::get() + 0x27b480`. This is the **address** of the function. When C++ is compiled down to machine code, **all variable and function names are erased** and functions are instead given **memory addresses**. A memory address is just **the location in a binary that the function resides in**. For example, `geode::base::get() + 0x27b480` means that the function is located at offset `0x27b480` (or 2602112 in decimal) bytes from the **base address** - GD's base address, given by the function `geode::base::get()`.

> :information_source: The reason we need to add the base address to the function's address is because **the base address of GD is dynamic** - it changes between startups!

What this means is that in order to hook a function in GD, **we need to know its address**. On top of that, **we need to know its signature**, as your detour must always have the same signature as the function you're hooking - otherwise the game will crash!

> :warning: The above code snippet also references **calling conventions** - if you use `$modify`, Geode handles them for you, however if you're doing manual hooks or reverse engineering, these are very important to get right.

### Addresses

So how do we find out these things? Usually, this is done through **reverse engineering**; however, RE is quite a complex skill, and would take far too much time to explain here, so it has its [own dedicated volume instead](https://docs.geode-sdk.org/handbook/vol2/chap2_1). And on top of that, **most common functions have already been found**. This means that instead of REing the function yourself, you can use the GD bindings that come packaged with Geode.

> :information_source: Traditionally, the most common GD header library was [**gd.h**](https://github.com/HJfod/gd.h). However, nowadays **gd.h is completely obsolete**, as it is only for 2.1 and fully unmaintained.

However, it is also important to note that **you still need to know how to reverse engineer** in order to make GD mods. Even if the addresses and signatures are all available, they still don't tell you what the function actually does, how it works, or where its called.

As noted previously, you shouldn't usually be creating hooks manually. Instead, **Geode comes with a special hooking syntax called `$modify`**. How it works will be explained in a later chapter, but first we must talk a bit about GD's game engine: **Cocos2d**.

### Notes

> [Note 1] Hook conflicts are a type of [**race condition**](https://en.m.wikipedia.org/wiki/Race_condition) and it happens when two mods try to hook the same function at the same time. If the mods do this sufficiently close to one another, there is a high chance that **one mod's hook will replace the other's**. The end result of this is that one of the mods functions incorrectly, when it fails to hook the function it expected to. In the best case, this just results in the mod losing functionality, but in the extreme case this **could cause crashes**.

## Chapter 1.4: Cocos2d

### GD's Game Engine

In order to modify any game, it's good to find out at least one thing: what engine the game was made with. Luckily, GD tells us this on its loading page: **Cocos2d-x**. GD modders have also figured out the specific version of Cocos2d that GD uses: **v2.2.3**. Although, there's a catch.

You see, if we look around the **libcocos2d** file GD comes with and compare it with the [official v2.2.3 branch](https://github.com/cocos2d/cocos2d-x/tree/cocos2d-x-2.2.3/cocos2dx), we will find that there are a bunch of functions and classes present in GD that aren't in the original, like `CCLabelBMFont::limitLabelSize`. As it turns out, **the Cocos2d that GD uses has been modified by RobTop**. These modifications range from small things like a few helper functions added to classes to some entire backends having been rewritten. Due to this, GD mods can't just use the publicly available Cocos2d headers; they need to use modified headers that account for RobTop's changes (which are, naturally, included in Geode).

> :warning: Some old modding tutorials on YouTube might tell you to use libraries such as CappuccinoSDK or Cocos-Headers - these are the traditional 2.1 libraries used, however they are nowadays obsolete.

### But What is Cocos2d?

Cocos2d, or specifically Cocos2d-x 2.2.3, is a **game engine featuring a sprite and node-based UI**. It is what powers GD's user interface, and many of its core data structures. For most GD mods, the two most important things Cocos2d provides us with are the **node system** and **garbage collector**.

The seminal building block of all Cocos2d UI and by extension GD UI is the `CCNode` class. A node is an UI object that can have multiple children, a single parent, and transformations such as position, rotation, scale, etc., with all transformations being also applied to a node's children.

These nodes form a hierarchy known as a **node tree**, on top of which is a `CCScene`. The scene is the only visible node without a parent [[Note 1]](#notes); at a time, there may only be one scene present. The class that manages scenes is `CCDirector`. At all times in GD, there is one director running, which you can get using the `CCDirector::sharedDirector` method, or the Geode-specific `CCDirector::get` shorthand. The director contains information such as the current window size, what scene is running, and getters for many of the other static **singleton managers** in GD. To change to a different scene, you would use the `CCDirector::replaceScene` method. To get the current scene, use `CCDirector::getRunningScene`.

Nearly all nodes you see in GD aren't instances of the base `CCNode` class, but of its derivatives. Some of the most commonly used derivatives are `CCSprite`, `CCMenu`, `CCLabelBMFont`, and `CCLayer`, and some of the most used GD-specific ones are `CCMenuItemSpriteExtra`, `FLAlertLayer`, `GameObject`, and `ButtonSprite`.

As Cocos2d is a node-based framework, nearly all nodes are **aggregates** of other nodes. In other words, **most nodes simply consist of other nodes**. For example, a popup might consist of a node for the background, another node for the text, and maybe some extra nodes for buttons and decoration and such. You most likely **won't have to do any rendering yourself**; if you want to show some text in your node, just create a label and add it as a child to it.

For example, here is **the structure of a comment** in GD:

As you can see, the `CommentCell` class consists wholly of other nodes. It does not do any of its own rendering. The position of the nodes (relative to the parent `CommentCell`) is marked in parenthesis; one important thing to note about Cocos2d is that unlike some other game frameworks, **higher Y-coordinate means higher on screen**.

> :warning: Please note that the above image has been simplified and some of the positions do not reflect the actual arrangement of `CommentCell`.

### Creating Nodes

Every node usually has a `create` function for creating an instance of the class, and then a bunch of methods like `setPosition` and `addChild` for setting the node's properties and adding children to it. For example, to create a simple "Hi mom!" text, you would use code like this:

```cpp
// First parameter is the text, second is the font
auto label = CCLabelBMFont::create("Hi mom!", "bigFont.fnt");
// Set the position of the label to be at the coordinates 100, 50 (in units, not pixels)
label->setPosition(100, 50);
// Assuming 'this' is some node aswell
this->addChild(label);
```

By default, all node `create` functions may return `nullptr` in case something goes wrong. However, nodes like `CCLayer` and `CCMenu` are likely to never fail, and in the rare case they do, something has probably already gone catastrophically wrong and the game is about to crash anyway, so handling the null case with these nodes is not usually necessary. However, with some classes like `CCSprite` **handling `create` returning null may be vital**, as `CCSprite::create` will fail if the sprite doesn't exist, which is entirely possible.

### Sprites

Cocos2d is a **sprite-based** framework, meaning that instead of rendering things at runtime using vector graphics, nearly everything shown on screen are textures loaded from disk. If you've ever messed around with GD's files, you've probably noticed this; most of GD's textures are contained in **sprite sheets**, such as `GJ_GameSheet03`. The way one shows these textures is with the `CCSprite` class; it is a simple node that loads a texture and displays it. `CCSprite` is a bit unique in that it has two main `create` functions: `CCSprite::create` for creating nodes out of individual images, and `CCSprite::createWithSpriteFrameName` for creating nodes out of spritesheet images.

It is important to use the correct `create` function for `CCSprite`, as the wrong one will return `nullptr` and cause an error if not properly handled. If the sprite you're creating is contained in its own file, like `GJ_button_01.png`, then you should use `CCSprite::create`; otherwise, if the sprite is in a spritesheet like `GJ_infoIcon_001.png` in `GJ_GameSheet03`, use `CCSprite::createWithSpriteFrameName` instead.

```cpp
// Uh oh! This will return null as GJ_infoIcon_001.png is not its own file but 
// contained in a spritesheet
auto infoSpriteFail = CCSprite::create("GJ_infoIcon_001.png");

// You don't have to specify the name of the spritesheet you're loading from anywhere, 
// as GD has already loaded GJ_GameSheet03 into memory
auto infoSpriteCorrect = CCSprite::createWithSpriteFrameName("GJ_infoIcon_001.png");
```

Even text in Cocos2d is rendered through sprites, or more accurately, **bitmap fonts**. The class for doing this is `CCLabelBMFont`, as shown in one of the above examples. GD's default [Pusab font](https://www.fontsquirrel.com/fonts/pusab) is defined in the `bigFont.fnt` file.

How to add your own sprites to use in Geode mods will be discussed in a later chapter.

### Menus & Buttons

Another important class to know of is `CCMenu`. Most mods need some sort of [buttons](/tutorials/buttons.md) for their UI, and for that, the most common class to use is `CCMenuItemSpriteExtra`. However, all `CCMenuItem`-derived classes **must be part of a `CCMenu` to work**. In practice, this means that all of your buttons must be the direct children of some menu.

You can actually see the effects of this in-game; hold down on some button, and then without releasing move your cursor over other buttons in the same scene. You will find that some buttons display their bounce animation indicating they are usable, and others don't. This is because of the Cocos2d **touch system**, which will be discussed in detail later, but in essence, only buttons in the same menu are clickable when you start holding down from one.

```cpp
auto infoIcon = CCSprite::createWithSpriteFrameName("GJ_infoIcon_001.png");

auto button = CCMenuItemSpriteExtra::create(
    infoIcon,
    // don't worry about these parameters for now
    nullptr, nullptr
);

// The button must be a child of a CCMenu to be usable
auto menu = CCMenu::create();
menu->addChild(button);

// This would make the button visible, but it would not be usable
auto layer = CCLayer::create();
layer->addChild(button);
```

At this point, we're getting very close to writing actual mod code. However, before we can get to that, we must first discuss **GD layers** and the `$modify` macro in Geode.

### Notes

> [Note 1] You can also add nodes directly to `CCDirector` and bypass the node tree, but this is very rarely done as nodes in the director don't have access to the touch system or any input for that matter.

## Chapter 1.6: Modifying Layers

So we know how to figure out the names of layers, but how do we actually modify them and add our own stuff?

If we want to modify what another node does, we must remember how hooking works: via hooking, we can modify **functions**. This is great, but unfortunately, classes aren't functions, so we can't exactly just "hook a layer". Instead, let's break down our problem a bit.

Layers are **instances** of a specific class. Them being instances means that even if we manage to modify one instance, if the user encounters another instance of the same class, we will have to be able to modify that aswell. Extrapolating from that, **our goal is to modify every instance of a specific class**.

Since hooking is only possible on functions, what we want to do is **find some function that is called for every instance of that class**. In particular, we want something that is as close to the creation of that instance as possible. If we add our own stuff with a delay, that would cause an inconvenient user experience - buttons should be there when the user enters a layer, not 5 seconds after! The function should also **only be called once per instance**, since if our modification involves adding a button for example, we don't want to add multiple copies of the same button.

Luckily for us, the **design patterns of Cocos2d nodes** provides us with a perfect candidate for nearly all classes.

### The Structure of a Node

Let's look at **the general structure of a node**:

```cpp
class SomeNode : public CCNode {
protected:
    bool init() {
        if (!CCNode::init())
            return false;
        
        // Initialize SomeNode

        return true;
    }

public:
    static SomeNode* create() {
        auto ret = new SomeNode();
        if (ret && ret->init()) {
            ret->autorelease();
            return ret;
        }
        CC_SAFE_DELETE(ret);
        return nullptr;
    }
};
```

Every node, and as such layer, has at least two functions: `create` and `init` [[Note 1]](#notes). `create`, as explained [in the previous chapter](/handbook/vol1/chap1_5.md), is what you use to create instances of the class.

What we're interested in right now however is `init`. This is the function where the node initializes itself; adds all of its subnodes, sets its delegates, etc.. As we can see in the definition of `create`, every instance of `SomeNode` is first created, and then its `init` function is invoked. On top of this, the `init` function is only called once per node, as it doesn't make sense to initialize the same node multiple times.

This should start to sound familiar: this is exactly what we outlined to be looking for at the start of the chapter!

Although, an observant reader may be wondering right now; if our goal is to find a function that is called every time a node is created, and is only called once per node, **why don't we just hook that layer's `create` function?** This is a good question; and the explanation is two-fold.

Firstly, `create` returns the created node, whereas in `init` we have direct access to the node through the `this` parameter (as `init` is a member function, and not static). This is only a code convenience issue however.

The bigger issue with hooking `create` is due to a problem known as **inlining**. We can't get into the depths of what that means right now, but in practice what inling means is that **some functions just aren't hookable**, and this includes many classes' `create` functions. In contrast, only a few classes' `init` functions are inlined, so we can be much more confident we can actually make the hook if we target `init`.

On top of this, hooking `init` actually brings us quite close to the actual design pattern Cocos2d uses, although to see that, we need to first hook an init function, which brings us to `$modify`.

### `$modify`

One of the most important concepts in all of Geode is the `$modify` macro; it is Geode's **syntactically sugary hooking system**. What it looks like is this:

```cpp
class $modify(ClassName) {
    // Overwrite functions, etc.
};
```

You will find that this declaration closely resembles a class, and underlyingly it in fact is one; however, **modify should not be treated like a normal class**. `$modify` comes with extra superpowers and **limitations** specific to hooking functions. Something that works for normal C++ classes is not guaranteed to work the same in `$modify`, even if the compiler might let you do it.

The name and syntax of `$modify` comes from its purpose; it is to **modify classes**.

For example, to modify the main menu, it would look like this (remembering from the previous chapter that the main menu's layer is called `MenuLayer`):

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
    // Overwrite functions, etc.
};
```

For the sake of compile-time performance, Geode's modifiers are split into files, which you can include by simply including `Geode/modify/{Name of the class}.hpp`. **You can only use `$modify` on classes that have been included**; in practice, this includes nearly all GD classes but does not include any of your own classes or Geode's classes.

You will sometimes see `$modify` be used with two arguments; this is to provide a **name** for the modified class, which is useful in many common situations.

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MyModifiedMenuLayer, MenuLayer) {
    // You can now refer to the modified class as 'MyModifiedMenuLayer'
};
```

If you don't provide a name, Geode will automatically generate a random name for the class, which is meant to not cause any name collisions.

> :warning: As `$modify` does not create a normal class, you should not expect standard C++ class things like adding members to work. Geode does come with [an utility for adding members to classes](/tutorials/fields.md), which is quite close to the normal way of declaring members, but not exactly the same.

### Hooking `init`

Ok, we know how to start modifying a class, but this doesn't yet do what we want. How exactly are supposed to hook `init`?

This is where the real magic of `$modify` comes in: to hook `MenuLayer::init`, all we have to do is **redefine the function in our modified class**:

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
    // Signature is `bool MenuLayer::init()`
    bool init() {
        // Our code!
    }
};
```

That's it! Now if you compiled this mod, installed it and opened the game, you would find that `MenuLayer` has turned completely blank, as we have overridden its `init` function, which means it can't add any of its nodes. You would also most likely find that the game would crash, since a lot of things depend on `MenuLayer` actually containing stuff.

And, well, we don't want to _override_ MenuLayer; we just want to append stuff to it. Luckily, from [the hooking chapter](/handbook/vol1/chap1_2.md), we know there is a tool for this: just **call the original function**!

...well, uh. How exactly do we do that?

The answer is simple: we, uh, call the original:

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
    // Signature is `bool MenuLayer::init()`
    bool init() {
        if (!MenuLayer::init())
            return false;

        // Our code!

        return true;
    }
};
```

That's it. As previously stated, this is quite close to the standard node design pattern.

> :warning: It should be noted that not all layer's `init` functions have the same signature, and `$modify` requires the signature to **exactly match** in order to create a hook. Unfortunately, due to implementation problems, it also (currently) doesn't tell you if your signature is wrong, so you may find yourself scratching your head as to why your mod isn't working, only to realize the signature of `init` is off. Check [GeometryDash.bro](https://github.com/geode-sdk/geode/blob/main/bindings/GeometryDash.bro) to make sure the signature of your hook is correct!

Now we are at an interesting point; we know how to hook functions, we know what function from a layer to hook in order to modify it, and we know how to work with nodes. So, let's tie all of this together! Only 7 chapters in, [it's time for **Hello, World!**](/handbook/vol1/chap1_7.md)

### Notes

> [Note 1] There are some very rare cases of nodes that don't have an `init` or `create` function. However, for the purposes of this tutorial, we will pretend those don't exist.

## Chapter 1.7: Hello, World!

If you just skipped to this chapter directly after seeing it in the navigation bar, you may be wondering why it has taken us 7 chapters to reach a `Hello, World!` program. Rest assured, the previous chapters are not just a waste; modding is a complex topic and Geode is an advanced framework. As such, we need to first lay a solid foundation for the basics of modding to build upon before we can get to actual code.

From here on out however, we're starting to leave the lame theory part and beginning to enter the part where you actually get to make stuff.

And with that, let's start building our **Hello, World!** mod!

### Constructing the Mod

To begin with, we need to outline what our mod will actually do. The goal is simple: add some text to the main menu in GD that says "Hello, World!".

### Modifying the Layer

Our first step is to use what we learned in the previous chapter and modify `MenuLayer`:

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
};
```

Since we want to add things to the layer, let's hook its `init` function, making sure to call the original:

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;
		
		return true;
	}
};
```

### Adding the label

Now it's time to actually show the text. As outlined in Chapter 1.4, Cocos2d is node-based; we don't do our own rendering, we leverage other nodes to do it for us. In this case, since we want to display text, we go for the standard `CCLabelBMFont` class:

```cpp
#include <Geode/modify/MenuLayer.hpp>

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto label = cocos2d::CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		
		return true;
	}
};
```

Everything Cocos2d-related lies in the `cocos2d` namespace, so we must prefix our usage of `CCLabelBMFont::create` with it. However, as you keep modding, having to stick `cocos2d::` everywhere gets annoying really fast. In addition, Geode comes with a bunch of namespaces you probably also don't want to rewrite every time. For this reason, Geode provides a `geode::prelude` namespace that automatically brings all Cocos2d and Geode-related namespaces to scope:

```cpp
#include <Geode/modify/MenuLayer.hpp>

using namespace geode::prelude;

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		
		return true;
	}
};
```

> :information_source: If you don't want to bring Geode namespaces into scope, you can just use `using namespace cocos2d` instead.

Now, the label isn't currently a child of any layer, so it won't show up anywhere. Let's add it as a child to `MenuLayer`:

```cpp
#include <Geode/modify/MenuLayer.hpp>

using namespace geode::prelude;

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		this->addChild(label);

		return true;
	}
};
```

### Positioning the label

The default position for any node is (0, 0) which is at bottom left of the screen; however, we want to center our label in the middle of the screen. To do this, we need to first figure out what the size of the screen is, and to do this we use the `getWinSize` method on `CCDirector`:

```cpp
#include <Geode/modify/MenuLayer.hpp>

using namespace geode::prelude;

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto winSize = CCDirector::get()->getWinSize();

		auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		this->addChild(label);

		return true;
	}
};
```

Next, to actually position our label, we call the `setPosition` method on it, placing it halfway across the screen horizontally and vertically:

```cpp
#include <Geode/modify/MenuLayer.hpp>

using namespace geode::prelude;

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto winSize = CCDirector::get()->getWinSize();

		auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		label->setPosition(winSize.width / 2, winSize.height / 2);
		this->addChild(label);

		return true;
	}
};
```

All nodes are by default centered relative to their position, so doing this is enough to center our label in the middle of the screen. However, we can make this code more concise by knowing that `winSize`'s type is `CCSize`, and that dividing a `CCSize` by an integer is the same as dividing that size's width and height by the integer. Since we are dividing both `winSize.width` and `winSize.height` by the same number, we can shorten it to just `winSize / 2`. In addition, all nodes have both a `setPosition(float, float)` and `setPosition(CCPoint)` setter by default, and `CCSize` is implicitly convertible to `CCPoint`.

All of this means that we can make our `setPosition` call shorter as just:
```cpp
label->setPosition(winSize / 2);
```

### Finished Mod

And with that, **we have completed our Hello, World! mod**. Here's what the final code looks like:

```cpp
#include <Geode/modify/MenuLayer.hpp>

using namespace geode::prelude;

class $modify(MenuLayer) {
	bool init() {
		if (!MenuLayer::init())
			return false;

		auto winSize = CCDirector::get()->getWinSize();

		auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
		label->setPosition(winSize / 2);
		this->addChild(label);

		return true;
	}
};
```

To try the mod out, `geode new`, and then replace the code in `src/main.cpp` with the above. After building the mod, open up GD and you should see this:

If it works for you, **congratulations!** You have now officially built your first Geometry Dash mod! Go grab a lil orange juice from the kitchen and give yourself a little treat :)

After drinking your juice however, it's time to get back into business. So we've got a Hello, World! going, that's great. Now it's time to start crafting something actually useful.
