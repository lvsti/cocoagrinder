---
layout: post
disqus: true
title:  "One storey my a(s)s"
date:   2018-01-14 22:38:00 +0100
excerpt: "Join me on a typographical adventure sparked by a High Sierra upgrade!"
---

I usually wait until any major OS update gets consolidated and only then do I upgrade; and this was no different with macOS High Sierra, which I decided to install last weekend. I wasn't expecting any surprises but then this happened:

![Oh no!]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/retarded.png)

I don't suffer from heavy OCD but this seemingly arbitrary typesetting just got me started. Questions were flooding my mind as I was desperately browsing the Preferences panel:

- What the hell is this ugly 'a' glyph? Is this a joke?
- What was the intention? Better readability? Unique character?
- Has anybody from the design department seen this before it got approved (if at all)?
- Why did Apple choose to deviate from the font used all over the system and in other built-in apps, making Notes.app the only exception?
- Last but not least: why don't they provide a means to disable it?

... because as you might have guessed, there is no setting in Notes for the default font. There has never been, but previously the app wasn't pretending to be a special snowflake. Now it is, and it annoys the hell outta me. So I decided to investigate it a bit.

### Shredding notes

My first trip was to the innards of the Notes.app package, where I was looking for the custom font&mdash; but found none. So this must be a system-provided font then. And indeed, if we open the fonts panel (_Format > Font > Show Fonts_), we will see "System Font / Regular" as selected. How is this possible?

I wanted to see how the font is actually set on the UI elements so I turned to my old friend, the [Hopper Disassembler](https://www.hopperapp.com) for help. I first tried searching for "font" and was soon greeted with a "DefaultNoteFont" setting in the user defaults:

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/hopper0.png)

I didn't know how this `UserStyleSheetGenerator` class might relate to the text editor font but I thought I might give it a try. I spent some time hacking up a small app that serializes an `NSFont` instance in the way that this code is expecting it, but no matter what font I chose, Notes.app seemed to ignore the preference altogether. 

Next I tried the top-down approach, and after making my way through the app delegate and the main window controller initialization, I found a promising clue in `awakeFromNib`:

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/hopper1.png)

Now that I knew what I should search for, I found that this method is called from multiple sites all over the code. The `fontWithSingleLineA` selector is not something we'll find on a regular `NSFont`: I gave it a chance with `performSelector:` but just got the usual doesNotRecognizeSelector exception back. Consequently, the method must be defined in one of the frameworks Notes.app uses. On to the terminal!

```
$ otool -L /Applications/Notes.app/Contents/MacOS/Notes | grep Private

    /System/Library/PrivateFrameworks/PencilKit.framework/Versions/A/PencilKit (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/NotesUI.framework/Versions/A/NotesUI (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/NotesShared.framework/Versions/A/NotesShared (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/AppleIDSSOAuthentication.framework/Versions/A/AppleIDSSOAuthentication (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/AOSAccounts.framework/Versions/A/AOSAccounts (compatibility version 1.0.0, current version 1.9.95)
    /System/Library/PrivateFrameworks/AOSKit.framework/Versions/A/AOSKit (compatibility version 1.0.0, current version 264.0.0)
    /System/Library/PrivateFrameworks/AuthKit.framework/Versions/A/AuthKit (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/AuthKitUI.framework/Versions/A/AuthKitUI (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/ContactsFoundation.framework/Versions/A/ContactsFoundation (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/CrashReporterSupport.framework/Versions/A/CrashReporterSupport (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/DataDetectorsCore.framework/Versions/A/DataDetectorsCore (compatibility version 1.0.0, current version 590.3.0)
    /System/Library/PrivateFrameworks/LocalAuthenticationUI.framework/Versions/A/LocalAuthenticationUI (compatibility version 1.0.0, current version 425.30.28)
    /System/Library/PrivateFrameworks/MarkupUI.framework/Versions/A/MarkupUI (compatibility version 1.0.0, current version 1.0.0)
    /System/Library/PrivateFrameworks/Notes.framework/Versions/A/Notes (compatibility version 1.0.0, current version 351.0.0)

```

Notes.app dynamically links a sh!tload of frameworks, many of them private. Dumping the framework symbols quickly revealed that `NotesUI.framework` would be my new best friend. Let's see what that pesky method actually does:

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/hopper2.png)

This code is basically constructing a new `NSFont` by altering attributes of the font instance passed in the argument. Here, the key to understanding was to resolve the magic numbers `0x23` and `0xe`, which turn out to be constants defined in `CoreText/SFNTLayoutTypes.h`: `kStylisticAlternativesType` and `kStylisticAltSevenOnSelector`, respectively. Let's reimplement this function in a test app:

```swift
func applicationDidFinishLaunching(_ aNotification: Notification) {
    let baseFont = NSFont.boldSystemFont(ofSize: 13)
    referenceTextField.font = baseFont

    let featureSettings: [NSFontDescriptor.FeatureKey: Any] = [
        .typeIdentifier: kStylisticAlternativesType,
        .selectorIdentifier: kStylisticAltSevenOnSelector
    ]
    let desc = baseFont.fontDescriptor.addingAttributes([
        .featureSettings: [featureSettings]
    ])
    
    let oneStoreyAFont = NSFont(descriptor: desc, size: baseFont.pointSize)
    oneStoreyTextField.font = oneStoreyAFont
}
```

Lo and behold:

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/onestorey1.png)

### Stylistic alternatives

So what exactly are these "stylistic alternatives"? Googling the constant names brings up Apple's [TrueType Reference Manual](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM09/AppendixF.html), according to which: _"this feature allows for different stylistic alternatives for glyphs [...] It is intended specifically to be used as the AAT equivalent of the OpenType 'ss01' through 'ss20' features."_ It turns out, any font can come with zero, one, or more glyphs (visual representations) for any character in the character table, and the mentioned alternatives are a means to switch between them in batch. 

In our case, the LATIN SMALL LETTER A (U+0061) presumably has at least 2 glyphs in the system font. To verify this assumption, we just  need to open `/System/Library/Fonts/SFNSText.ttf` with any sufficiently advanced font editor (I used the [Glyphs app](https://glyphsapp.com)):

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/glyphs1.png)

OK, this glyph indeed looks like the "one-storey a" but how to make sure it really is the one? Let's take a look at the font info panel:

![]({{ site.baseurl }}/assets/images/2018-01-14-one-storey-my-ass/glyphs2.png)

We've seen that the `ssNN` features define the alternatives, we know that Notes.app is referring to the 7th alternative, and right at the first line of the `ss07` definition there is a substitution rule that replaces the standard `a` glyph with `g604` aka. the "one-storey a". The jigsaw is complete.

### Conclusion

Looking back at the adventures I have gone through, you might think that, just like in those predictable Hollywood movies featuring character development (no pun intended), I've got to like the "one-storey a". Well, that cannot be further from the truth. I still hate it with my whole heart and I dare say Steve would agree with me. If you happen to know of a less intrusive way of getting rid of it than:

- replacing the system font with the alternative disabled
- replacing the NotesUI framework

please let me know.

Until then, I'm reverting to Sierra's Notes.app. One storey my a(s)s!
