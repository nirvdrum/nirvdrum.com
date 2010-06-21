---
layout: post
title: How to Take Full Page or Full Canvas Screenshots in Windows
show_ads: true
---

Introduction
------------

While developing [MogoTest](http://mogotest.com/), a service for detecting Web browser rendering issues, I found it necessary to be able to capture the contents of the entire browser canvas rather than just the current viewport.  A full page screenshot lets the user see how their content looks from a holistic perspective.  Unfortunately, capturing all of the content clipped by the scrollable viewport is not a very straightforward process.

One approach to grabbing the full canvas is to take a screenshot and then scroll the viewport to display all the previously hidden sections, stitching all of the images together to form one composite that represents the full canvas contents.  While this generally works, it fails if there are any fixed position elements on the page or if there is ECMAScript in place to modify the DOM on scroll events because in both cases the very act of scrolling modifies the canvas's contents.  It would be better then to capture the entire canvas without scrolling.

The Problem with Making Windows Large Enough to Remove Scrollbars
-----------------------------------------------------------------

Windows does not allow windows to be larger than the virtual screen resolution by default.  The virtual screen resolution is defined as the vertical and horizontal span of all connected displays.  If you have a single display, the current screen resolution will have a 1:1 match with the virtual screen resolution and if you have two displays side-by-side, the virtual screen resolution will be the height of the lowest resolution by the sum of the two horizontal screen resolutions.

Resizing the window to display the entire canvas contents, thus, requires some trickery in handling the virtual screen dimensions.  As it turns out, every window is sent a [`WM_GETMINMAXINFO` message](http://msdn.microsoft.com/en-us/library/ms632626%28VS.85%29.aspx) just before the window's screen coordinates or size is changed.  `WM_GETMINMAXINFO` passes as its `lParam` a [`MINMAXINFO` value](http://msdn.microsoft.com/en-us/library/ms632605%28VS.85%29.aspx) which contains the virtual screen dimensions as the the `ptMaxTrackSize` member.  Modifying the `lParam` before the window receives the message would allow us to effectively change the virtual screen resolutions on a per-process basis.

If you have access to the source code of the application you'd like to capture the full canvas contents of, the solution is simple: just modify your message processing loop to handle `WM_GETMINMAXINFO` messages and modify the `lParam` as necessary.  In the general case, however, you won't be able to modify the binary so you'll have to modify the executing process's address space to inject your own message handler.

While the general concept is rather straightforward, the implementation is fairly convoluted.  At the core of it, the complexity is caused by the `WM_GETMINMAXINFO` message being sent rather than retrieved by the window.


Tricking Windows into Letting you Resize the Window Larger than the Screen
--------------------------------------------------------------------------

The Win32 API allows applications to monitor global event message traffic by setting up a hook procedure via the [`SetWindowsHookEx` function](http://msdn.microsoft.com/en-us/library/ms644990%28VS.85%29.aspx).  This is how utilities like Spy++ are able to tell what messages are being sent to a program.  As of this writing, there are thirteen different [types of hooks](http://msdn.microsoft.com/en-us/library/ms644959%28VS.85%29.aspx#types), each with its own context and execution point in the global chain.

If you're like me, you'd probably try to register a `WH_GETMESSAGE` hook with `GetMsgProc`.  On the outset it seems logical enough: intercept the `WM_GETMINMAXINFO` message before `GetMessage` or `PeekMessage` is called and modify accordingly.  This will not work, however, and you will waste a lot of time trying to make it work.  The nuance here is that `WM_GETMINMAXINFO` is sent to the window -- the window does not poll for it -- and as such a `WH_GETMESSAGE` hook will never see the message.

The next seemingly logical hook type to try is `WH_CALLWNDPROC`, which you register with the [`CallWndProc` function](http://msdn.microsoft.com/en-us/library/ms644975%28VS.85%29.aspx).  This type of hook will indeed intercept the `WM_MINMAXINFO` message before the window will, but unfortunately, the hook procedure cannot modify the message.  This is by design and Windows will enforce it; trying to get a reference to the `lParam` to modify the value in memory will not work.

And so it goes with all the hook types.  Many look like they'll do what you need, but will fail in some way.  It seems that modifying the `WM_GETMINMAXINFO` message from out of process is not possible.  And largely that's true.  However, we can get creative by supplanting the process's window procedure, which executes in process, by using `SetWindowLongPtr` from the `WH_CALLWNDPROC` hook.  Example 1 shows what that interaction may look like.

<pre class='brush: cpp;'>
// The WH_CALLWNDPROC hook procedure, executed out-of-process.
LRESULT WINAPI CallWndProc(int nCode, WPARAM wParam, LPARAM lParam)
{
    CWPSTRUCT* cwp = (CWPSTRUCT*) lParam;

    if (WM_GETMINMAXINFO == cwp->message)
    {
        // Inject our own message processor into the process so we
        // can modify the WM_GETMINMAXINFO message.  It is not possible
        // to modify the message from this hook, so the best we can do
        // is inject a function that can.
        LONG_PTR proc = SetWindowLongPtr(cwp->hwnd, GWL_WNDPROC, (LONG_PTR) MinMaxInfoHandler);
        
        // Store a handle to the original window procedure in the window's
        // property list so we can restore the procedure from the custom processor.
        SetProp(cwp->hwnd, L"__original_message_processor__", (HANDLE) proc);
    }

    return CallhkHookEx(hkHook, nCode, wParam, lParam);
}

// Install the hook procedure in the global hook chain.  DLL_PATH must be
// a fully qualified path to the DLL file containing the WH_CALLWNDPROC hook procedure.
HINSTANCE hinstDLL = LoadLibrary(DLL_PATH);
HOOKPROC hkprcSysMsg = (HOOKPROC)GetProcAddress(hinstDLL, "CallWndProc");
HHOOK hkHook = SetWindowsHookEx(WH_CALLWNDPROC, hkprcSysMsg, hinstDLL, 0);
</pre>
<div class='caption'>Example 1: Registering the <code>WH_CALLWNDPROC</code> hook procedure.</div>

[`SetWindowLongPtr`](http://msdn.microsoft.com/en-us/library/ms644898%28VS.85%29.aspx) is an amazing feature of Windows that lets you supply a new function pointer for a restricted set of functions in a Windows process.  The new function can then call out to the original function through a handle to that function.  One of the functions allowed to be replaced is the window procedure.  By supplying our own we will be able to finally modify that `WM_MINMAXINFO` message.  In `Example 1` we showed how to call `SetWindowLongPtr`.  `Example 2` shows what the custom window procedure looks like:

<pre class='brush: cpp;'>
// The custom window procedure, executed in-process, to manipulate the WM_MINMAXINFO message.
LRESULT CALLBACK MinMaxInfoHandler(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    // Grab a reference to the original message processor.
    HANDLE originalMessageProc = GetProp(hwnd, L"__original_message_processor__");
    RemoveProp(hwnd, L"__original_message_processor__");
 
    // Uninstall this custom window procedure so the next message will use the original procedure.
    SetWindowLongPtr(hwnd, GWL_WNDPROC, (LONG_PTR) originalMessageProc);
 
    // Only handle the message we're interested in.
    if (WM_GETMINMAXINFO == message)
    {
        MINMAXINFO* minMaxInfo = (MINMAXINFO*) lParam;
 
        // ptMaxTrackSize corresponds to the screen's virtual dimensions.
        // MAX_WIDTH and MAX_HEIGHT should be the width and height of the window allowing the full
        // canvas to be displayed without scrollbars.  This is application dependent.
        minMaxInfo->ptMaxTrackSize.x = MAX_WIDTH;
        minMaxInfo->ptMaxTrackSize.y = MAX_HEIGHT;
 
        // We're not going to pass this message onto the original message processor, so we should
        // return 0, per the contract for the WM_GETMINMAXINFO message.
        return 0;
    }
 
    // All other messages should be handled by the original message processor.
    return CallWindowProc((WNDPROC) originalMessageProc, hwnd, message, wParam, lParam);
}
</pre>
<div class='caption'>Example 2: Modifying the <code>WM_GETMINMAXINFO</code> message with a custom window procedure.</div>

Note that we only handle the `WM_GETMINMAXINFO` message and delegate all others to the original window procedure.  Additionally, we uninstall the custom procedure as soon as we've accomplished what we need to.

We modify the `ptMaxTrackSize` component of the `MINMAXINFO` struct, which is itself a `POINT` struct, having an `x` and a `y` component.  These should be set large enough to handle the full canvas plus the window chrome that surrounds the main client area.  Once this is done, you should be able to size the window large enough to obviate the need for scrollbars.

Fig. 1 shows how this all ties together between a theoretical screenshot.exe process taking full canvas screenshots in Internet Explorer.

<div class="figure">
  <img src="/images/how-to-take-full-page-screenshots-in-windows/comprehensive-system-diagram.png" />
  
  Fig. 1: Comprehensive interaction diagram for taking full page screenshots.
</div>


Capturing the Canvas Contents
-----------------------------

Now that your application can be sized large enough to capture the canvas contents, you must resize the window to that maximum size.  This calculation is application dependent.  For Internet Explorer much of the work is done with the IWebBrowser2 interface, for example.

One caveat is that if the window is already maximized, Windows will not send it a sizing message.  My solution to this problem is to first check if the window is already maximized and if so note that fact, change the maximized state to restored, then resize the window to be large enough for the full canvas contents.  Once done, I then re-maximize the window if it was previously maximized, effectively restoring the window to its original dimensions.  It is a bit kludgy, but I haven't been able to come up with a better solution.  I suspect there is a way by intercepting a different window message, but I couldn't figure out which one if it is in fact possible.  This process can be seen in `Example 3`.

All that remains now is to capture the contents, unregister the `WH_CALLWNDPROC` hook, and resize the window to its original dimensions so the user doesn't have to deal with a massive window.  `Example 3` pulls all this code together.

<pre class='brush: cpp;'>
// Check if the window is maximized.
BOOL isMaximized = IsZoomed(hwnd);
if (isMaximized)
{
    ShowWindow(hwnd, SW_SHOWNORMAL);
}
else
{
  // Store the window's original dimensions into some local variables.
}

// Set the window to its new dimensions.  There are a variety of ways to do this.

// Note that CImage is part of ATL.  If you want to use strict Win32 API for DIBs, you
// can do so; it's just much more complicated.
CImage image;
image.Create(imageWidth, imageHeight, 24);
CImageDC imageDC(image);

// Capture the contents of the client area to our image DC.
PrintWindow(hwnd, imageDC, PW_CLIENTONLY);

// Remove our `WH_CALLWNDPROC` hook procedure from the global hook chain.
// hkHook was the return value from the SetWindowsHookEx function call.
UnhookWindowsHookEx(hkHook);

// Restore the window to the original dimensions.
if (isMaximized)
{
    ShowWindow(hwnd, SW_MAXIMIZE);
}
else
{
    // Set the window to its original dimensions.
}

// Actually save the image file.
image.Save(CW2T(outputFile));
</pre>
<div class='caption'>Example 3: Taking the full canvas screenshot.</div>


Conclusion
----------

Taking full page or full canvas screenshots in Windows can be tricky, but the method discussed in this article should be widely applicable.  In my particular case I was enhancing the [SnapsIE](http://snapsie.sf.net/) utility.  My [SnapsIE fork](http://github.com/nirvdrum/SnapsIE/blob/master/CoSnapsie.cpp) illustrates how I use all of these techniques.  Note that SnapsIE is written as an ActiveX control for Internet Explorer, so the code is likely more complex than is warranted in many cases.

Acknowledgments
---------------

- Haw-Bin Chai for [SnapsIE](http://snapsie.sourceforge.net/), which served as a basis for much of the work I did.
- Jim Evans for more [IE screenshot work on selenium](http://code.google.com/p/selenium/issues/detail?id=326&colspec=ID%20Stars%20Type%20Status%20Priority%20Milestone%20Owner%20Summary&start=100), which handled IE8 a bit more gracefully than SnapsIE did.
- [sunnyandy](http://www.codeguru.com/forum/showthread.php?p=1889928), who had the closest answer on how to take full screen screenshots that I was able to find.
- Igor Tandetnik, who knows VC++ better than any human I'm aware of.
- [Jeff Rafter](http://neverlet.be/), who helped me debug all sorts of issues while I was developing the foundation for this article and then served as a peer reviewer of the content.
- [MogoTest](http://mogotest.com/) for allowing me to spend all this time solving the problem.
