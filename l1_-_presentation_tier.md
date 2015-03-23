UI
==

### L1 - presentation tier

* presentation logic
* client tier
* stickiness, usability
* visually appealing layout
* intuitive navigation structure


### L2 - HTML

* 1995 HTML2.0 
* www.w3.org
* separation of content and presentation
	* same content rendered differently
	* changes in one place
	* capturing the meaning of a doc
	
* no flow control
* what should appear on a webpage, not how appear or how behave
* HTML doc structured as tree
* start tag, attribute, content, end tag
* attributes - additional meaning  
	* name - value pair inside start tag
* id must be unique 
* class applied to a collection
* anchor \<a> creates hyperlink

### L3

* meta - data about datas
* body - text level (no break), block( has a break)

### L4 - forms
*  form element
*  submit form -> user agent(browser) -> processing agent(server side)
*  action - url of the processing agent on server side
*  method - http request 
*  GET -> idempotent, no side effects
*  no sensitive data with GET
*  no large amount of form data
*  cannot be used with file control

* POST -> side effect
* sensitive data -> encryption 
* form data encoded by user agent according to the content type:
	* application/x-www-form-urlencoded: encoded as name value paris
	* multipart/form-data: encoded as a message, with a separate part for each control
	* text/plain: plain text
	
### L5 - template

* controller makes var available to associated view
* Embedded Ruby - erb
* just a filter
*  <%= %> ruby code, result substituted back into file
*  <% %> no substitution after execution
*  HTML specify structure of doc; ruby provide information
*  Rails helper methods
*  template is passed to layout file: comment informations
*  <%= yield %> where insert block template
*  default layout: /views/layouts/application.html.erb

### L6 - CSS

* presentation semantics of HTML 
* how appear
* SASS -> interpreted into CSS
* CSS rule
	* selector {declaration}
	* selector - elements that rule will be applied to
	* declaration - property:value pairs
* class selector . (class="")
* type selector h1 {color:purple}
* id selector #chapter1 {text-align:center}

Pseudo-classes: applied to concept

Box model

* border-style, border-width etc.
* control position
	* position property
	
### L7 - js & jquery

* scripting language - client side js
* run locally in the browser
* js runs in a sandbox
* same origin policy - script can not access DOM on others websites

* query - js library
* unobtrusive js
* retrieving elements - selector jQuery(<selector>) $(<selector>)
* returned elements are wrapped with additional functions - wrapped set
* chain

### L8 - ajax

* group of tech
* XMLHttpRequest(XHR) -  API available 
* asynchronous HTTP request 
* any text data can be retrieved
* $.ajax() method
* unobtrusive js adapter - jquery_ujs
* data-remote="true"
* 