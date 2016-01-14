#Scrappy Scraper by Patricia Decker
This simple web app allows a user to enter a valid URL address and retrieve the source's HTML. The source code is displayed and highlighted using PrismJS, and a summary of the tags in the source code is presented using DataTables. The jQuery Validator Plugin is also used for form handling. A user can click on a tag in the summary to highlight it (and its siblings) in the source code, or vice versa.

##Libraries and Plugins

###Using *Requests* Module to Retrieve HTML
The requests module lets you easily download files from the Web without having to worry about complicated issues such as network errors, connection problems, and data compression. We enter the request as follows:

```
>>> page = requests.get('http://www.slack.com')
```

If the request succeeds, the downloaded web page is stored as a string in the Response object’s text variable.

```
>>> page.text
u'\n\t\n\n<!DOCTYPE html>\n<html lang="en">\n\t<head>\n\n\t\t<script>\nwindow.ts_endpoint_url = "https:\\/\\/slack.com\\/beacon\\/timing";\n\n(function(e) {\n\tvar n=Date.now?Date.now():+new Date,r=e.performance||{},t=[],a={},i=function(e,n){for(var r=0,a=t.length,i=[];a>r;r++)t[r][e]==n&&i.push(t[r]);return i},o=function(e,n){for(var r,a=t.length;a--;)r=t[a],r.entryType!=e||void 0!==n&&r.name!=n||t.splice(a,1)};r.now||(r.now=r.webkitNow||r.mozNow||r.msNow||function(){return(Date.now?Date.now():+new Date)-n}),r.mark||(r.mark=r.webkitMark||function(e){var n={name:e,entryType:"mark",startTime:r.now(),duration:0};t.push(n),a[e]=n}),r.measure|| ...
```

###Using *PrismJS* to Highlight Source Code
In its own words, "Prism is a lightweight, extensible syntax highlighter, built with modern web standards in mind." After considering Highlight.js and Google's Prettify, I chose to use Prism because (a) I preferred how it looked and (b) see (a).


###Using *DataTables* to Create Tag Summary
The DataTables plugin is powerful yet easy-to-use when it comes to making tables that are customizable. Its creators state DataTables' mission is "To enhance the accessibility of data in HTML tables".

Getting started with DataTables is as simple as including two files in your web-site, the CSS styling and the DataTables script itself.


###Using *Validator Plugin* to Validate URL
This plugin checks to see if the user has tried to enter information and returns an error as soon as invalid information is enetered. Using the plugin requires 2 stylesheets for formatting and 2 script tags -- one for basic jQuery and the other for the validation plugin.

##Code Walkthrough

###Checking for Valid URL Address
First we need to use jQuery to listen to our form:

```
$(function( handler ) {

})
```

This statement is equivalent to the following:

```
$().ready( handler );
$(document).ready( handler );
```

The jQuery Validator Plugin allows for definition of custom methods. In the below custom method we check that the string begins with either 'http' or 'https':

```
  $.validator.addMethod('httpVsHttpsProtocol',
                        function (value, element, params) {
    return /^https?:/i.test(value)
  })
```

The test() method tests for a match in a string. This method returns true if it finds a match, otherwise it returns false.

###Form Validation and Submission
Next, we want to validate the entire form. In this case, we are requiring that the user input a valid URL ('url: true') and that our custom method httpVsHttpsProtocol returns True ('httpVsHttpsProtocol: true'):

```
  $('form').validate({
    rules: {
      url: {
        required: true,
        url: true,
        httpVsHttpsProtocol: true
      }
    },
    messages: {
      url: "Please enter a valid URL."
    },
    submitHandler: function (form) {
      console.log("Submitting form ...")
      submitForm(form)
    },
    errorContainer: "#validation-errors",
    errorLabelContainer: "#validation-errors",
  })
```

We submit the form using .validate's submitHandler, a callback for handling the actual submit when the form is valid that receives the form as the only argument. The form currently being validated is a DOM Element such that validation will not be retriggered. submitHandler replaces the default submit, and it is the right place to submit a form via Ajax after it is validated. In this case, we use the  custom function submitForm, discussed in the next section.

###submitForm
When we submit the form, we override the basic submit using the submitHandler option in the jQuery .validate() method described above. We seek to do the following:

* Fetch the source code from the validated URL and return it as a string
* For successful retrieval, create the tag summary table and display the source code
* For unsuccessful events, display error messages.

####Requesting the Source Code using jQuery XMLHTTPRequest
The jQuery post method takes the url, which is retrieved from the form's 'action' attribute, and data, among other arguments. In this case, the url input is the Flask route used to grab the URL content and the data passed is the URL whose source code we will be retrieving. We serialize the information from the form because the result is a text string in standard URL-encoded notation. 

```
function submitForm(form) {
  // Fetch source code using jQuery XMLHTTPRequest 
  var jqXHR = $.post($(form).attr('action'),
                     $(form).serialize());
```

Before serving up the information, we reset the #error and #success to class 'hidden'.

```
  $('#error, #success').addClass('hidden');
```

####Handling the Request Results
The .done function argument is called when the request terminates. Here, if the request returns an error we want to display the #error div and associated error messages. Otherwise, upon a successful return we want to:

* Display the source code highlighted with Prism in #source div
* Create the tag summary object, tagToCount
* Create the DataTable from tagToCount


```
  jqXHR.done(function (data) {
    
    if (data.status === "error") {
      $('#error-message').text(data.message);
      $('#error').removeClass('hidden');
      $('div#error-container').removeClass('hidden');
      return
    }
    // Prism case-sensitive to DOCTYPE
    var source = data.content.replace(/<!doctype/, '<!DOCTYPE');
    console.log("This is source in jqXHR:");
    console.log(source);
    // Show source HTML
    $('#source').text(source);
    
    // calls Prism.highlightElement() on all lanugage-markup elements
    Prism.highlightAll()

    // Find the opening and closing tags for above elements
    var allTags = $('span.token.tag:has(span.token.tag)');
    console.log(allTags)
    var tagToCount = {};
    allTags.each(function() {
      var tagName = this.firstChild.lastChild.data;
      highlighter($(this), tagName);
      // Count opening tags
      if (this.firstChild.firstChild.innerText === "<") {
        tagToCount[tagName] = (tagToCount[tagName] || 0) + 1;
      }
    })

    // Tag Summarization
    var rowData = $.map(tagToCount, function (value, key) {
      return [[key, value]]
    })
    $('#tag-summary').DataTable()
                     .clear()
                     .rows.add(rowData)
                     .rows().every(function () {
                       var tagName = this.data()[0];
                       highlighter($(this.node()), tagName);
                     })
                     .draw();
    $('#success').removeClass('hidden');
    $('div#success-container').removeClass('hidden');
  })
}
```

####Create the Tag Summary
When using Prism, every token that is highlighted gets two classes: token and a class with the token type (e.g. comment). When we call Prism.highlightAll(), the  highest-level function in Prism’s API, the method fetches all the elements that have a .language-markup class and then calls Prism.highlightElement() on each one of them. The class is specified in the HTML lower in the document:

```
<h3>Source</h3>
<!-- for Prism usage, 'language-markup' class for HTML -->        
<pre>
  <code class="language-markup" id="source"></code>
</pre>
```

Also note that Prism case-sensitive to 'DOCTYPE', because of the following RegEx used in its Markup language definition:

```
Prism.languages.markup = {
    'doctype': /<!DOCTYPE[\w\W]+?>/,
    ...
```

Because of the way Prism identifies and highlights tokens, to create the tagToCount object, we need to retrieve all of the span elements with class 'token' and class 'tag' as follows using the ':has' selector, which selects all elements that have one or more elements inside of them, that matches the specified selector:

```
var allTags = $('span.token.tag:has(span.token.tag)');
```

See [Prism](http://prismjs.com/) documentation for details. For each tag, find the tag name by taking the data for the last child of the first child of the tag:

```
var tagName = this.firstChild.lastChild.data;
```

Looking at allTags[0] results in something like the following:

```
<span class="token tag pointer tag-html">
  <span class="token tag">
    <span class="token punctuation">&lt;</span>html
  </span>
  <span class="token attr-name">lang</span>
  <span class="token attr-value">
    <span class="token punctuation">=</span>
    <span class="token punctuation">"</span>en
    <span class="token punctuation">"</span>
  </span>
  <span class="token punctuation">&gt;</span>
</span>
```

Such that:

allTags[0].firstChild --> <span class="token tag"> ... </span>
          .firstChild.lastChild --> "html"
          .firstChild.lastChild.data --> html

####Create the DataTable from tagToCount
We use the function highlighter() to add helper classes to the tags, where 'pointer' is a class for changing the cursor and 'tag-tagName' is used for selecting the elements for highlighting on click. If we click on a tag that is not yet highlighted, remove the highlight class from the previously clicked tag in the dataTable and the source code, then select the current tag and highlight it in the source code and in the dataTable:

```
// use DataTable to present tag summary
function highlighter(jQuery, tagName) {
  jQuery.addClass('pointer')
        .addClass('tag-' + tagName)
        .data('tagName', tagName)
        .click(function() {
          if (!$(this).hasClass('highlight')) {
            var dataTable = $('#tag-summary').DataTable()
            $('.highlight').removeClass('highlight')
            dataTable.$('.highlight').removeClass('highlight')
            var tagSelector = '.tag-' + $(this).data('tagName')
            $(tagSelector).addClass('highlight')
            dataTable.$(tagSelector).addClass('highlight')
          }
        })
}

```

According to the jQuery documentation that the jQuery.data() method allows us to attach data of any type to DOM elements in a way that is safe from circular references and therefore free from memory leaks. jQuery ensures that the data is removed when DOM elements are removed via jQuery methods, and when the user leaves the page.
