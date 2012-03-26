Lessons learned with CSS Regions and Shadow DOM
=====
       

About
-----
This is a quick overview of lessons learned working with CSS Regions and Shadow DOM during the Adobe Romania WebKit Hackathon. 
 

Scope
-----
By definition, it is possible to have a Shadow DOM-based component that will handle pagination of a named flow created with CSS Regions.    

At the time of this writing - March 2012, the Shadow DOM  as well as CSS Regions implementations are still works in progress. Some parts are not implemented, thus workarounds have to be used. Further developments may validate or invalidate any of the functionality described in this document. 

Take this document as a guideline and learning report, not as a tutorial!  
  

Use case
-----
                                                   
I want to flow the content of an article in a set of regions.
As a developer, I want to create placeholder elements (regions) under a Shadow DOM in order to keep the main DOM clean of empty, non-semantic placeholders. 
I want to listen to NamedFlow and/or region events in order to create or remove placeholders in the Shadow DOM based on how the content fits.
                                                       
       
Crash course into Shadow DOM
-----

[Shadow DOM](http://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html) is a new proposal backed by Google that enables developers to build self-contained web application modules based on HTML, CSS and JavaScript. 

The developer can choose to encapsulate these modules as loosely or as strictly as required. This way outside code and stylesheets may or may not affect to the inner workings of the module.

Dimitri Glazkov, initiator of the proposal, explains the [Shadow DOM concept](http://glazkov.com/2011/01/14/what-the-heck-is-shadow-dom/).  

Shadow DOM is laying the groundwork for Web Components. Christian Cantrell briefly explains this in a video of [Shadow DOM and Web Components](http://www.youtube.com/watch?v=pQOuHNm5seY). 


Shadow DOM features
-----
     

**Shadow Root encapsulation** 

A new Shadow DOM may be created under any DOM element with the `ShadowRoot` constructor. The DOM element acts as a host for the Shadow DOM root and it defines its boundaries.

<pre>
var host = document.querySelector("#module");
var root = new WebKitShadowRoot(host);    
</pre>                                                                                                                                                

Working under `root` you may create other DOM elements, attach event handlers and styles just as you would on a normal node.

<pre>
var paragraph = document.createElement('p');
paragraph.textContent = 'in the shadow';
            
// the paragraph belongs to the shadow root.
root.appendChild(paragraph);
</pre>                  

Element queries made outside the shadow root's boundaries do not match its contents. 
You need to query the shadow root for its child nodes. The shadow root makes available its own query methods (querySelector, getElementById, etc.). 

<pre>
var elements = document.querySelectorAll('p'); // 0 nodes
var elements = root.querySelectorAll('p'); // 1 node    
</pre>
       
       
**Stylesheets in ShadowDOM**                                                      

By default, styles defined outside the shadow root **will not** be applied on the child elements under the shadow root.           

<pre>
&lt;style type="text/css"&gt;
    p {
        color: green;
    }
&lt;/style&gt; 

&lt;script type="text/javascript"&gt; 
    var p = root.querySelector('p');
    
    console.log(window.getComputedStyle(p, null)['color']) // rgb(0, 0, 0) black, not green   
&lt;/script&gt; 
</pre> 
    
Styles defined under the shadow root **should apply** only to the root's child elements and not outside it. 

In my brief experiments I also found that the stylesheets defined under the root need to have the `scoped` attribute in order to apply. This may be just a temporary implementation bug. More on [scoped styles](http://dev.w3.org/html5/markup/style.html#style.attrs.scoped). 

<pre>
&lt;script type="text/javascript"&gt;

    // take the stylesheet
    var style = document.querySelector('style');    

    // apply the stylesheet only under its container element
    style.scoped = true;    

    // move the stylesheet under the shadow root
    root.appendChild(style);

    var p = root.querySelector('p');
    console.log(window.getComputedStyle(p, null)['color']) // rgb(0, 128, 0) green, ok!     
    
&lt;/script&gt;    
</pre>  

**Inheriting outside styles**  

The CSS Shadow DOM proposal specifies that there is a way to allow stylesheets defined outside the boundaries apply on elements within the boundaries.

The boolean [applyAuthorStyles](https://dvcs.w3.org/hg/webcomponents/raw-file/tip/spec/shadow/index.html#api-shadow-root-apply-author-styles) flag on the shadow root should toggle this behavior.
At the time of this writing - March 2012, this flag isn't implemented yet so the behavior does not work.

<pre>                             
// apply stylesheets defined outside the shadow root
root.applyAuthorStyles = true; 
</pre> 
      
Current Shadow DOM limitations for CSS Regions
-----

**applyAuthorStyles and flow-from**

In relation to CSS Regions, the `applyAuthorStyles` flag is a requirement for my use case. 

Because the styles defined outside the shadow root cannot be applied within its boundaries, the `flow-from` properties cannot read from the named flows defined at the document level. 

<pre>
&lt;style type="text/css"&gt; 
    article {
        -webkit-flow-into: myFlow;     
    }                             
    
    .region {
        -webkit-flow-from: myFlow;
    }
&lt;/style&gt; 

&lt;script type="text/javascript"&gt;

    // create a new placeholder
    var region = document.createElement('div');
    
    // make it a region
    region.className = 'region';
        
    // add the region to the shadow dom
    root.appendChild(region);
    
    console.log(window.getComputedStyle(region, null)['-webkit-flow-from']) // 'none', not 'myFlow'   
&lt;/script&gt; 
</pre>  

When implemented, the use of `applyAuthorStyles` allows me to define named flows outside the Shadow DOM and regions in the Shadow DOM

<pre>
// create a new placeholder
var region = document.createElement('div');

// make it a region
region.className = 'region'; 
                              
// apply stylesheets defined outside the shadow root
root.applyAuthorStyles = true;
    
// add the region to the shadow dom
root.appendChild(region);

console.log(window.getComputedStyle(region, null)['-webkit-flow-from']) // should be 'myFlow'    
</pre>
____    

**Encapsulation and flow-into**

Because the content nodes of my article are defined outside the shadow root, stylesheets defined under the shadow root do not apply to them.
Thus, I cannot grab the content nodes and flow them into a named flow.  

As a work-around I also need to **move** the content nodes under my shadow root, where the stylesheet can see them. This is a limitation, because other styles, defined outside may not apply. Also, event handlers referencing the `parentNode` may get unexpected results. 

<pre>       
// find the article in the main DOM
var content = document.querySelector('article');

// move the article node under the shadow root
root.appendChild(content)    

// .region will now see the content of the 'myFlow' named flow  
</pre>   

____ 

**NamedFlow interface and Shadow DOM**

The CSS Object Model `NamedFlow` interface is declared on the document.   
Because of the encapsulation, named flows defined under the shadow DOM root **are not** exposed to the `document`.

Thus, even though the "myFlow" named flow exists under the shadow root, `document.getFlowByName("myFlow")` will not return the correct reference. Any properties on the NamedFlow object, such as "overflow", have invalid values.

  
 
Partial conclusions
----

* Work is in progress on the Shadow DOM implementation to support the `applyAuthorStyles`. This will enable defining named flows outside the shadow root boundaries, then using them on regions defined within the boundaries.

* There is currently no way of grabbing references to named flows defined under a shadow root. This happens because the `getFlowByName` is defined on the `document` and the shadow root encapsulates its styles and child nodes. 


