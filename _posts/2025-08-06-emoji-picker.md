---
layout: post
title: "Missing Emoji Picker for iOS"
date: 2025-08-06
author: "Andrii Zinoviev"
tags: [swift, ios, reverse engineering]
---

## Missing Emoji Picker for iOS
So, I've been working for a small iOS app for a while, and I have stumbled upon needing some kind of emoji picker. Quick googling revealed some third-party libraries (like [MCEmojiPicker](https://github.com/izyumkin/MCEmojiPicker) and [Elegant-Emoji-Picker](https://github.com/Finalet/Elegant-Emoji-Picker)), but I thought "Well, this is common scenario, I can't believe that UIKit has no premade way of doing this". Additionally, relying on the third-party library that is trying to mock system UI feels like a bad idea for me. 

After some more googling I [found](https://www.reddit.com/r/SwiftUI/comments/1i2kfco/comment/m7j64vp/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) interesting "private" `UIKeyboardType`:
```swift
extension UIKeyboardType {
    static let emoji = UIKeyboardType(rawValue: 124)
}
```

But `UIKeyboardType` is applicable only to text inputs! Certainly, emoji picker is not a full text input by any means. I noticed `UIKeyInput` protocol to create simple text input components, so I gave it a try with the new shiny `UIKeyboardType`:
```swift
final class EmojiPickerUIView: UIView, UIKeyInput {
    var hasText: Bool = true

    func insertText(_ text: String) {
        print(text)
    }

    func deleteBackward() {}

    override var canBecomeFirstResponder: Bool {
        true
    }

    var keyboardType: UIKeyboardType = .init(rawValue: 124)!

    override func didMoveToWindow() {
        self.becomeFirstResponder()
    }
}

struct EmojiPicker: UIViewRepresentable {
    func makeUIView(context: Context) -> EmojiPickerUIView {
        EmojiPickerUIView()
    }
    
    func updateUIView(_ uiView: EmojiPickerUIView, context: Context) {}
}

struct ContentView: View {
    var body: some View {
        EmojiPicker()
    }
}
```

Here is the result:

![1.png](/assets/images/emoji-picker/1.png)

Looks pretty good, I'd say even good enough for simple apps. You can add additional logic to resign the first responder after one character was entered, state management and additional stuff.
### The Apple way
After some time, I found this interesting UI inside system iOS Reminders app:

![2.png](/assets/images/emoji-picker/2.png)

Wait, this is the same exact scenario that I have been looking for! And there is 2 key differences from my "homebrew" implementation:
- Emoji search is fully functional.
- The dictation button is missing, which is a good thing, because does not actually do anything useful for this particular control.

So, if Apple managed to implement this, there is definitely a way for mere mortals. Right?
### Digging deeper
To inspect apps on iOS, you have different options, but the best option is somehow getting your debugger to attach to the app. Unfortunately, Reminders app is impossible to hook into without jailbreaked iPhone, because iOS allows the debugger to attach only when the target application was signed with the `get-task-allow` entitlement. And obviously, every production application is prohibited to have it.

However, there is a way to attach the debugger to **all** Simulator processes. For this we will need to disable [System Integrity Protection](https://developer.apple.com/documentation/security/disabling-and-enabling-system-integrity-protection) (SIP) on our host machine. 

***Eww, that sounds not very secure.*** That's why I have a dedicated virtual machine with SIP disabled. So, let's fire up a Simulator inside a SIP-disabled virtual machine, and start our Reminders app:

![3.png](/assets/images/emoji-picker/3.png)

Wait... Where is our beloved emoji feature? 
Turns out that Reminders app does not fully work without iCloud sync enabled (pretty sure that same pattern would be instantly rejected in the App Store). 
There is a sneaky "Update" button on the home screen:

![4.png](/assets/images/emoji-picker/4.png)

Upon tapping, this will redirect to the system settings and prompt you to sign in to the Apple Account. No problem, I already have my simulator on the host machine with Apple Account, should not be a problem in a VM.

![5.png](/assets/images/emoji-picker/5.png)

Well, there is actually a problem. Turns out that it is impossible to sign in into Simulator running under macOS VM. Hence, we have no luck in even seeing out needed feature in the Reminders app. How inconvenient. Because I had no desire of putting my main machine under SIP disablement security risks, I gave up. For a while.
### Enabling debugger access without security risks
Reverse engineering was always my excitement point. Unsolved mysteries about closed-source apps are periodically haunting me, reminding about them. One day, I found this fabulous command:
```
csrutil enable --without debug
```
This allows debuggers to be attached to any system process, but still hold all other SIP enforcements:
```
➜  ~ csrutil status
System Integrity Protection status: unknown (Custom Configuration).

Configuration:
	Apple Internal: disabled
	Kext Signing: enabled
	Filesystem Protections: enabled
	Debugging Restrictions: disabled
	DTrace Restrictions: enabled
	NVRAM Protections: enabled
	BaseSystem Verification: enabled
	Boot-arg Restrictions: enabled
	Kernel Integrity Protections: enabled
	Authenticated Root Requirement: enabled

This is an unsupported configuration, likely to break in the future and leave your machine in an unknown state.
```
After checking it in VM, I decided to try it on my main machine to continue working on the Reminders reverse engineering. And it worked!

Remember, **ALWAYS** turn SIP on after experimenting. You never know when hackers will be able to abuse the debugger access to steal all your data!
### Getting in
So, finally we have an access to the debugger. No we just need to launch our simulator, log into Apple Account, launch Reminders and attach to the process in Xcode:

![6.png](/assets/images/emoji-picker/6.png)

Here it is:

![7.png](/assets/images/emoji-picker/7.png)

First thing first - let's navigate to the needed screen and run Debug View Hierarchy:

![8.png](/assets/images/emoji-picker/8.png)

How cool is that?
First thing that caught my attention is this strange vertical bar on the custom emoji button. What can it be? Focusing on it, we get the anticipated answer:

![9.png](/assets/images/emoji-picker/9.png)
```
**<_TtC9RemindersP33_4FE8BD245B420E02FDB196B5E5563CD014EmojiTextField: 0x10185d400; baseClass = UITextField; frame = (29.5 450.5; 53 53); text = ''; alpha = 0; opaque = NO; gestureRecognizers = <NSArray: 0x600000d7c9c0>; borderStyle = None; background = <_UITextFieldNoBackgroundProvider: 0x6000000056c0: textfield=<_TtC9RemindersP33_4FE8BD245B420E02FDB196B5E5563CD014EmojiTextField: 0x10185d400>>; layer = <CALayer: 0x600000380a60>>**
```
So, the Reminders app **does** in fact use plain old `UITextField` to drive the emoji picker! Also, you may notice that it has its `alpha = 0`. This allows it to still become the first responder and accept keystrokes, despite being invisible for the end user. But how does it achieve this emoji-only keyboard layout? Let's see:
```lldb
(lldb) eco [0x10185d400 keyboardType]
0x000000000000007c
```

`eco` is just an alias for executing Objective-C code in a fast way:
```
command alias eco expression -l objective-c -o --
```

You can already recognize our old `UIKeyboardType` friend (`0x7c = 124`), and now we in fact know that Apple uses it internally.

Let's try to upgrade our emoji picker by utilizing the `UITextField` approach:

```swift
final class EmojiPickerUIView: UITextField {
    init() {
        super.init(frame: .zero)
        self.keyboardType = .init(rawValue: 124)!
        self.alpha = 0
    }
    
    override func didMoveToWindow() {
        self.becomeFirstResponder()
    }
}
```

![10.png](/assets/images/emoji-picker/10.png)

Much better! And the search is now working properly:

![11.png](/assets/images/emoji-picker/11.png)

However, there is still one minor difference, and I was really aimed at perfection after all these adventures. This small "Dictation" button is still present in our implementation. Let's get rid of it!
### The ugly bit
Turns out that there is no public option for disabling dictation on the keyboard. But we still have one last chance to inspect Reminders: the binary itself. Firstly, let's try to find some symbols related to dictation in our runtime:
```lldb
(lldb) image lookup -rn DisableDictation
```
There are a lot of matches, but some of them caught my attention:
```
Summary: UIKitCore`-[UITextInputTraits forceDisableDictation]
Address: UIKitCore[0x0000000001228544] (UIKitCore.__TEXT.__text + 19026260)
Summary: UIKitCore`-[UITextInputTraits setForceDisableDictation:]
Address: UIKitCore[0x000000000134a528] (UIKitCore.__TEXT.__text + 20214072)
```
Let's try setting a symbolic regex breakpoint on all symbols that contain `setForceDisableDictation`:
```lldb
(lldb) br set -r setForceDisableDictation
Breakpoint 1: 5 locations.
```
Now, run the Reminders application, reopen the list editor and click on the custom emoji button:

![12.png](/assets/images/emoji-picker/12.png)

Looks like we're getting somewhere. `-[UITextInputTraits setForceDisableDictation:]` is actually a pretty simple method, just storing the value of `w2` (the first parameter in ObjC world) into self (`x0`) instance:

![13.png](/assets/images/emoji-picker/13.png)

We can even print the description of `$x0` (`self` argument):
```lldb
(lldb) po $x0
<UITextInputTraits : 0x106f0d410>
public
   autocapitalization:                  2
   autocorrection:                      0
   spellchecking:                       0
   keyboard type:                       124
   kb appearance:                       0
   return key type:                     0
   auto return key:                     N
   secure text entry:                   N
   was ever secure text entry:          N
   device passcode entry:               N
   smart insert/delete type:            0
   smart quotes type:                   0
   smart dashes type:                   0
   writingToolsBehavior:            0
   allowedWritingToolsResultOptions:     0
private
   text trimming set:                   0x0
   ins. pt. color:                      (null)
   ins. pt. width:                      3
   text loupe vis.:                     0
   selection behavior:                  0
   text suggest. del.:                  (null)
   single-line document:                1
   single value:                        0
   has default contents:                0
   accepts payloads:                    0
   accepts emoji:                       1
   acceptsInitialEmojiKeyboard:         0
   accepts dictation search results:    0
   use automatic endpointing:           0
   show dictation button:               0
   force-enable dictation:              0
   force-disable dictation:             0
   force-spelling dictation: 0
   force default dictation info         0
   force dictation keyboard type:       9223372036854775807
   prefer online dictation              0
   disabled return key:                 0
   return key goes to next responder:   0
   accepts floating keyboard:           1
   force floating keyboard:             0
   floating keyboard edge insets:       {0, 0, 0, 0}
   accepts split keyboard:              1
   displaySecureTextUsingPlainText:     0
   displaySecureEditsUsingPlainText:    0
   learnsCorrections:        1
   shortcut conversion:                 0
   suppress return key styling:         0
   localized with UI language:          0
   defer becomeFirstResponder:          0
   enables return key for NWS content:  N
   autocorrection context:              (null)
   response context:                    (null)
   input context history:               (null)
   disable prediction:                  N
   hide prediction:                     N
   inline prediction type:              0
   disableInputBars:                    N
   isCarPlayIdiom:                      N
   loadKeyboardsForSiriLanguage:        N
   textScriptType:                      0
   supplemental lexicon:                (null)
   supplemental lexicon ambiguous item icon: (null)
   disableHandwritingKeyboard:          0
   math expression completion:          0
   text animations allowed:             0
```
As you can see we have a private `force-disable dictation` field in the `UITextInputTraits`. So, let's see how we can use it in our emoji picker implementation. Obviously, `-[UITextInputTraits setForceDisableDictation:]` is a private API, so you should not use it in the production apps (or at least obfuscate it).

It is not straightforward to call it using `performSelector`, because, as we can see from the assembly, the method expects raw `BOOL` (or integer), and `performSelector` always wraps the arguments in `NSNumber`-ish boxes.

So, let's add this innocent and unsafe as hell code to our `UITextField.init`:
```swift
let selector = NSSelectorFromString("setForceDisableDictation:")
typealias PrivateIMP = @convention(c) (AnyObject, Selector, Bool) -> Void
let imp = class_getMethodImplementation(object_getClass(self), selector)!
let function = unsafeBitCast(imp, to: PrivateIMP.self)
function(self, selector, true)
```

![14.png](/assets/images/emoji-picker/14.png)

Finally, we get the exact same result as the Reminders app!
### Some assembly required
Sometimes, it is beneficial to debug and disassemble the binary at the same time, cross-referencing symbols and behavior. To obtain system app binary in the first place, you can either grab it from the simulator runtime, or extract from real iOS IPSW. 
I decided to take a look at the Reminders in Hopper Disassembler. Firstly, I have found all symbols, related to `EmojiTextField` to see if I missed something:

![15.png](/assets/images/emoji-picker/15.png)

Looks like `EmojiTextField` is pretty simple, since there is no configuration logic in init:

![16.png](/assets/images/emoji-picker/16.png)

### Text input source override
However, for some reason, `EmojiTextField` overrides private `_textInputSource` and `set_textInputSource:` methods:

![17.png](/assets/images/emoji-picker/17.png)

Here, `_textInputSource` returns `0x1` always, and `set_textInputSource` does nothing. Digging a little into `UIKitCore` binary I found this little C function: `_UITextInputSourceToString`. We can execute it from lldb with all interesting values of `textInputSource`:
```lldb
(lldb) im loo -n _UITextInputSourceToString
1 match found in /Library/Developer/CoreSimulator/Volumes/iOS_22F77/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS 18.5.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/UIKitCore.framework/UIKitCore:
        Address: UIKitCore[0x0000000000db41ac] (UIKitCore.__TEXT.__text + 14355900)
        Summary: UIKitCore`_UITextInputSourceToString
```
To obtain the symbol address, we can add its relative offset (`0x0000000000db41ac`) to the library load address (`0x184DFB000`, taken from the Xcode debugger):
```lldb
(lldb) p/x 0x0000000000db41ac+0x184DFB000
(long) 0x0000000185baf1ac
```
After it, we can call the function by the pointer using plain old C magic, and observe the return value:
```lldb
(lldb) eco ((void*(*)(int))(uintptr_t)0x0000000185baf1ac)(1);
KBTap
```
Using this approach, we can get all valid values for `textInputSource`:
```
0 -> UNK  
1 -> KBTap  
2 -> Dict  
3 -> Penc  
4 -> KBHW  
5 -> KB3P  
6 -> KBPath  
7 -> KBCam  
8 -> KB-Indirect  
9 -> UNK  
```
So apparently, `EmojiTextField` overrides input source to always be equal to `KBTap`. I found no real reason to do this, but I guess Apple engineers have stumbled upon some bug with external keyboards or Apple Pencil.
### Gluing logic
Now, let's try to find the logic for enabling and disabling the emoji text input. Given that `EmojiTextField` has no real logic inside, my guess was to look for its delegate. It is fairly easy to find using `lldb` that its delegate is `TTRIListDetailEmblemsTableCell`. We can find implemented delegate methods using Hopper:

![18.png](/assets/images/emoji-picker/18.png)

Leveraging built in pseudo-decompiler we can see the actual implementation. Let's look at the  `-[_TtC9Reminders30TTRIListDetailEmblemsTableCell textField:shouldChangeCharactersInRange:replacementString:]:`
```c
/* @class _TtC9Reminders30TTRIListDetailEmblemsTableCell */
-(int)textField:(int)arg2 shouldChangeCharactersInRange:(int)arg3 replacementString:(int)arg4, ... {
    static ();
    objc_retain_x19();
    objc_retain_x20();
    sub_1005fb214(r21, arg1);
    objc_release_x19();
    objc_release_x20();
    swift_bridgeObjectRelease(arg1);
    r0 = r21 & 0x1;
    return r0;
}
```
Here we are just passing control to Swift native function `sub_1005fb214`, performing all needed bridging.
```c
int sub_1005fb214(int arg0, int arg1) {
    // Prolog
    ...
    
    CEMStringIsSingleEmoji();
    objc_release_x22();
    if (r23 != 0x0) {
            r0 = swift_unknownObjectWeakLoadStrong();
            if (r0 != 0x0) {
                    r8 = *qword_10079c1b0;
                    r8 = r0 + r8;
                    r22 = *r8;
                    if (r22 != 0x0) {
                            sub_100077710(r22, *(r8 + 0x8));
                            swift_bridgeObjectRetain(r21);
                            loc_2216(0x0, 0x1, r20, r21);
                            sub_100046f0c(r22, r23);
                            swift_unknownObjectRelease(r24);
                            swift_bridgeObjectRelease(r21);
                    }
                    else {
                            swift_unknownObjectRelease(r0);
                    }
            }
            *(int8_t *)(r19 + *objc_ivar_offset__TtC9Reminders30TTRIListDetailEmblemsTableCell_emojiWasSelected) = 0x1;
    }
    [sub_1005f9fd4() resignFirstResponder];
    objc_release_x20();
    [sub_1005f9fd4() setHidden:0x1];
    objc_release_x20();
    r0 = swift_unknownObjectWeakLoadStrong();
    if (r0 != 0x0) {
            [r0 removeFromSuperview];
            objc_release_x20();
    }
    swift_unknownObjectWeakAssign();
    return 0x1;
}
```
We can see that it is calling a private `CEMStringIsSingleEmoji` function from private `CoreEmoji` framework to check if the inputted string is a single emoji, and if it is, we are doing some logic (likely notifying the delegate). In any case, after character input, we resign the first responder from the text field and hide it. So, for the end user, the text field is virtually invisible at all times. Pretty neat technology, Apple!
### Conclusion
So, now we have everything we need to construct real native emoji picker, with 0 third-party dependencies, and using system keyboard. Just be careful, because the private API's may break at any point.

I plan to release this small component it on my [GitHub](https://github.com/shivatinker) as soon as possible, so watch for the updates.
