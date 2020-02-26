# pythonista-webview

WKWebView implementation for Pythonista.

The underlying component used to implement ui.WebView in Pythonista is
UIWebView, which has been deprecated since iOS 8. This module implements a
Python webview API using the current iOS-provided view, WKWebView. Besides
being Apple-supported, WKWebView brings other benefits such as better
Javascript performance and an official communication channel from
Javascript to Python. This implementation of a Python API also has the
additional benefit of being inheritable.

Available as a [single file](https://github.com/mikaelho/pythonista-webview) 
on GitHub, or install with:

    pip install pythonista-wkwebview
    
in stash.

Run the file as-is to try out some of the capabilities; check the end of the
file for demo code.

Credits: This would not exist without @JonB and @mithrendal (Pythonista
forum handles).

## Basic usage

WKWebView matches ui.WebView API as defined in Pythonista docs. For example:

```
v = WKWebView()
v.present()
v.load_html('<body>Hello world</body>')
v.load_url('http://omz-software.com/pythonista/')
```

For compatibility, there is also the same delegate API that ui.WebView has,
with `webview_should_start_load` etc. methods.

## Deviations from ui.WebView API

### Synchronous vs. asynchronous JS evaluation

Apple's WKWebView only provides an async Javascript evaluation function.
This is available as an `eval_js_async` method, with an optional `callback` 
argument that will be called with a single argument containing the result of 
the JS evaluation (or None).

We also provide a synchronous `eval_js` method, which essentially waits for 
the callback before returning the result. For this to work, you have to call 
the `eval_js` method outside the main UI thread, e.g. from a method decorated 
with `ui.in_background`.

### Handling page scaling

UIWebView had a property called `scales_page_to_fit`, WKWebView does not. See 
below for the various `disable` methods that can be used instead.

## Additional features and notes

### http allowed

Looks like Pythonista has the specific plist entry required to allow fetching 
non-secure http urls. 

### Cache and timeouts

For remote (non-file) `load_url` requests, there are two additional options:
  
* Set `no_cache` to `True` to skip the local cache, default is `False`
* Set `timeout` to a specific timeout value, default is 10 (seconds)

You can also explicitly clear all data types from the default data store with 
the `clear_cache` instance method. The method takes an optional parameter, a 
plain function that will be called when the async cache clearing operation is 
finished:

    def cleared():
      print('Cache cleared')
    
    WKWebView().clear_cache(cleared)

### Media playback

Following media playback options are available as WKWebView constructor 
parameters:

* `inline_media` - whether HTML5 videos play inline or use the native 
  full-screen controller. The default value for iPhone is False and the
  default value for iPad is True.
* `airplay_media` - whether AirPlay is allowed. The default value is True.
* `pip_media` - whether HTML5 videos can play picture-in-picture. 
  The default value is True.

### Other url schemes

If you try to open a url not natively supported by WKWebView, such as `tel:` 
for phone numbers, the `webbrowser` module is used to open it.

### Swipe navigation

There is a new property, `swipe_navigation`, False by default. If set to True, 
horizontal swipes navigate backwards and forwards in the browsing history.

Note that browsing history is only updated for calls to `load_url` - 
`load_html` is ignored (Apple feature that has some security justification).

### Data detection

By default, no Apple data detectors are active for WKWebView. You can activate 
them by including one or a tuple of the following values as the 
`data_detectors` argument to the constructor: NONE, PHONE_NUMBER, LINK, 
ADDRESS, CALENDAR_EVENT, TRACKING_NUMBER, FLIGHT_NUMBER, LOOKUP_SUGGESTION, 
ALL.

For example, activating just the phone and link detectors:
  
    v = WKWebView(data_detectors=(WKWebView.PHONE_NUMBER, WKWebView.LINK))

### Messages from JS to Python

WKWebView comes with support for JS-to-container messages. Use this by 
subclassing WKWebView and implementing methods that start with `on_` and 
accept one message argument. These methods are then callable from JS with the 
pithy `window.webkit.messageHandler.<name>.postMessage` call, where `<name>` 
corresponds to whatever you have on the method name after the `on_` prefix.

Here's a minimal example:
  
    class MagicWebView(WKWebView):
      
      def on_magic(self, message):
        print('WKWebView magic ' + message)
        
    html = '''
    <body>
    <button onclick="window.webkit.messageHandlers.magic.postMessage(\'spell\')">
    Cast a spell
    </button>
    </body>
    '''
    
    v = MagicWebView()
    v.load_html(html)
    
Note that JS postMessage must have a parameter, and the message argument to 
the Python handler is always a string version of that parameter. For 
structured data, you need to use e.g. JSON at both ends.

### User scripts a.k.a. script injection

WKWebView supports defining JS scripts that will be automatically loaded with 
every page. 

Use the `add_script(js_script, add_to_end=True)` method for this.

Scripts are added to all frames. Removing scripts is currently not implemented.

Following two convenience methods are also available:
  
* `add_style(css)` to add a style tag containing the given CSS style
  definition.
* `add_meta(name, content)` to add a meta tag with the given name and content.

### Making a web page behave more like an app

These methods set various style and meta tags to disable typical web
interaction modes:
  
* `disable_zoom`
* `disable_user_selection`
* `disable_font_resizing`
* `disable_scrolling` (alias for setting `scroll_enabled` to False)

There is also a convenience method, `disable_all`, which calls all of the 
above.

Note that disabling user selection will also disable the automatic data 
detection of e.g. phone numbers, described earlier.

### Javascript debugging

Javascript errors and console messages are sent to Python side and printed to 
Pythonista console. Supported JS console methods are `log`, `info`, `warn` and 
`error`.

For further JS debugging and experimentation, there is a simple convenience 
command-line utility that can be used to evaluate load URLs and evaluate 
javascript. If you `present` your app as a 'panel', you can easily switch 
between the tabs for your web page and this console.

Or you can just create a WKWebView manually for quick experimentation, like 
in the usage example below. 

    >>> from wkwebview import *
    >>> v = WKWebView(name='Demo')
    >>> WKWebView.console()
    Welcome to WKWebView console. Evaluate javascript in any active WKWebView.
    Special commands: list, switch #, load <url>, quit
    js> list
    0 - Demo - 
    js> load http://omz-software.com/pythonista/
    js> list
    0 - Demo - Pythonista for iOS
    js> document.title
    Pythonista for iOS
    js> quit
    >>> v2 = WKWebView(name='Other view')
    >>> WKWebView.console()
    Welcome to WKWebView console. Evaluate javascript in any active WKWebView.
    Special commands: list, switch #, load <url>, quit
    js> list
    0 - Demo - Pythonista for iOS
    1 - Other view - 
    js> switch 1
    js> load https://www.python.org
    js> document.title
    Welcome to Python.org
    js> window.doesNotExist.wrongFunction()
    ERROR: TypeError: undefined is not an object
    (evaluating 'window.doesNotExist.wrongFunction')
    (https://www.python.org/, line: 1, column: 20)
    None
    js> quit

### Setting a custom user agent

WKWebView has a `user_agent` property that can be used to retrieve or set a 
value reported to the server when requesting pages.

### Customize Javascript popups

Javascript alert, confirm and prompt dialogs are now implemented with simple 
Pythonista equivalents. If you need something fancier or e.g. 
internationalization support, subclass WKWebView and re-implement the 
following methods as needed:
  
    def _javascript_alert(self, host, message):
      console.alert(host, message, 'OK', hide_cancel_button=True)
      
    def _javascript_confirm(self, host, message):
      try:
        console.alert(host, message, 'OK')
        return True
      except KeyboardInterrupt:
        return False
      
    def _javascript_prompt(self, host, prompt, default_text):
      try:
        return console.input_alert(host, prompt, default_text, 'OK')
      except KeyboardInterrupt:
        return None

