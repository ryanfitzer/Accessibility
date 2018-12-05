# Javascript Accessibility Techniques #

**This repository has been archived. Most, if not all, of this information is probably no longer accurate**

A disorganized list of tips and tricks I've found handy when developing accessible javascript-heavy interactions. Any recommendations for techniques are welcomed (or pull requests).

**Note**: these techniques are generalized due to the complexity of different brower/SR combinations. What works for one combo may not apply to another.

## Index ##

* [Focus Proper Element After an Asynchronous Event](#focus-element)
* [Trigger a Virtual Buffer Update](#trigger-vb-update)
* [Accessibility References](#references)
* [Articles of Note](#articles)

## Glossory ##

AT: [Assistive Technology](http://wikipedia.org/wiki/Assistive_Technology)  
SR: [Screen Reader](http://wikipedia.org/wiki/Screen_Reader)  
VB: Virtual Buffer  

---

<a name="focus-element"></a>
### Focus Proper Element After an Asynchronous Event ###

Upon revealing/concealing and element (whether or not an AJAX request was made), a user's AT needs focus to be applied to new element that has been revealed (opening), or the original triggering element (closing) used to reveal the element. Some examples are:

* Opening/closing an overlay
* Opening/closing a sliding drawer
* Showing errors when validating a form
* Opening/closing a custom dropdown menu
* Opening/closing a tooltip
    
Unfortunately, it's not always as simple as using an element's `focus` method. The predominate SR, [JAWS][][^1], has trouble finding the element that is currently focused if certain events, AJAX for instance, take place at the same time an element is revealed, or concealed. It's very confusing and there is little documentation on the subject, but JAWS VB has something to do with it. The VB holds a representation of the DOM to enable users to interact with the page. This is in contrast to other SRs (OSX's [VoiceOver][]) that simply interact with the live DOM.
  
A technique found to be successful is calling an element's `focus` method from inside a `window.setTimeout`. Browser javascript engines are murky, but what is known is that `window.setTimeout`, at a minimum, places its supplied callback into the execution queue once the delay has expired. The theory is that it enables the call to the element's `focus` method to occur after the VB has updated. I'm sure this is an oversimplification.
  
To illustrate, here is an example function (using an object to hold our methods helps organize things):

    myNameSpace = {
        
        shiftFocus: function( element, delay ) {
            
            window.setTimeout( function() {
                
                element.focus();
            
            }, delay || 0 );
        }
    }

This should be used anytime an element needs focus after an asynchronous event. Notice the use of the default `delay` of `0`. I have found this to work 99% of the time. Every so often a `delay` of `1000` will need to be used. I've seen no benefit in more milliseconds being used (or any value between `0` and `1000`).

**ToDo**: Elements that aren't natively focusable (using the tab key) need their `tabindex` set to `-1`. This makes sure the element can be programmatically focused (using the `focus` method), but preserves its non-natively focusable state. Natively focusable elements become non-natively focusable when their `tabindex` is set to `-1`, which becomes a problem. Instead of testing for native focusability, `shiftFocus` should always apply and then remove the element's `tabindex` (or set it back to its original value). Due to older versions of IE using a camelCased syntax (`tabIndex`), the lowercase variant is ignored. Not a problem for a cross-browser DOM library, but without it, a conditional check needs to be made. I'll add this in soon.

---

<a name="trigger-vb-update"></a>
### Trigger a Virtual Buffer Update ###

[Juicy Studio][] published an article that demonstrated, in JAWS 7.1, a forced VB update could be triggered by toggling the [value of a hidden input][updateBuffer]. This is a very handy technique for making sure JAWS recognizes page updates via AJAX. I found the technique was still valid in JAWS 12.
  
An example implementation (assuming [jQuery][] is included):
    
    // Run myNameSpace.init() when DOM is ready
    
    myNameSpace = {
        
        init: function() {
            
            var self = this;
            
            self.updateBuffer();
        },
        
        updateBuffer: function() {
            
            var self = this,
                me = self.updateBuffer;
            
            if ( me.inputReady ) {
                
                me.input.val( me.input.val() === '0' ? '1' : '0' );
                
            } else {
                
                me.input = $( '<input/>', {
                    
                    id: 'virtualBufferUpdate',
                    
                    type: 'hidden',
                    
                    value: '0'
                });
    
                me.input.appendTo( 'body' );
                me.inputReady = true;
                $( document ).bind( 'updateBuffer', $.proxy( me, self ) );
    
            }
        },
    
        shiftFocus: function( element, delay ) {
            
            window.setTimeout( function() {
                
                element.focus();
            
            }, delay || 0 );
        }
    }
    
As you can see, the `updateBuffer` method also contains a listener on the `updateBuffer` custom event. Invoking it by triggering the jQuery custom event allows for others to react if needed. So now, whenever you need to do an update, just trigger the `updateBuffer` custom event:

    $( document ).trigger( 'updateBuffer' );
    
Using this in conjunction with the `shiftFocus` method, JAWS can be forced to act as expected. While triggering a VB update just before the `window.setTimeout` is set has been the most successful so far, I've found that it depends on the interaction. Some interactions have required the the update to occur inside the `window.setTimeout`. Since this was the exception, I used an altered version of `shiftFocus` for those cases. Try all configurations.

---

<a name="references"></a>
#### Accessibility References ####

* [Accessible Culture][] - In depth articles and tests for AT/Browser combinations.
* [Juicy Studio][] - In depth articles and techniques for accessible experiences.

---

<a name="articles"></a>
#### Articles of Note ####

* [Using `role=alert` and `aria-live=assertive` to force JAWS to read focused content](http://www.accessibleculture.org/research-files/test-cases/aria/alert/index.html).
* [Forcing JAWS to update its virtual buffer][updateBuffer]


[^1]: As tested in JAWS' versions 11 & 12.  
  
  
[JAWS]:http://wikipedia.org/wiki/JAWS_(screen_reader)
[VoiceOver]:http://www.apple.com/accessibility/voiceover/
[jQuery]:http://jquery.com
[Juicy Studio]:http://juicystudio.com
[Accessible Culture]:http://accessibleculture.org/
[updateBuffer]:http://juicystudio.com/article/improving-ajax-applications-for-jaws-users.php

