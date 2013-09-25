.. _model-using-label:

Models, Templates, Views
=========================
Now that we have our models set up and populated with some data, we can now put all the pieces together, sourcing data from our models in our views, and then presenting that data in our templates.

Basic Workflow: A data driven pages
-----------------------------------
To create a data driven web page requires four (five) main steps.

#. First, import the models you wish to use into your application's ``views.py`` file.
#. Within the view you wish to use, query the model to get the data you want to present.
#. Pass the results from your model into the template's context.
#. Setup your template to present the data to the user in whatever way you wish.
#. If you have not done so already, map a URL to your view.

These steps highlight how the  Django's framework separates the concerns between Models, Views and Templates, and how they are related.

Showing Categories on the Main/Index page
-----------------------------------------
One of the requirements regarding the main pages was to show the top five rango'ed categories.


To fulfil this requirement, we will go through each of the above steps. First open ``rango/views.py`` and import the ``Category`` model from Rango's ``models.py`` file.

.. code-block:: python
	
	# Import the Category model
	from rango.models import Category

Modifying the Index View
........................

With the first step out of the way, we then want to modify our ``index()`` function. If we cast our minds back, we should remember the ``index()`` function is responsible for the main page view. Modify the function as follows:

.. code-block:: python
	
	def index(request):
	    # Obtain the context from the HTTP request.
	    context = RequestContext(request)
	    
	    # Query the database for a list of ALL categories currently stored.
	    # Place the list in our context_dict dictionary which will be passed to the template engine.
	    category_list = Category.objects.ordered_by('-likes')[:5]
	    context_dict = {'categories': category_list}
	    
	    # Render the response and send it back!
	    return render_to_response('rango/index.html', context_dict, context)

Here we have performed steps two and three in one go: first we queried the ``Category`` model to retrieve the top five categories. Here we used the ``ordered_by`` function to sort by the number of likes in descending order (thus the inclusion of ``-``), and then we restricted this list to the first 5 Category objects in the list.

With the query complete, we passed a reference to the list (stored as variable ``category_list``) to a new dictionary, ``context_dict``. This dictionary is then passed as part of the context for the template engine in the ``render_to_response`` call.

Modifying the Index Template
............................

With the view updated, all that is left for us to do is update the template ``rango/index.html``, located within the ``templates`` directory. Change the HTML code of the file so that it looks like:

.. code-block:: html
	
	<!DOCTYPE html>
	<html>
	    <head>
	        <title>Rango</title>
	    </head>
	
	    <body>
	        <h1>Rango says...hello world!</h1>
	
	        {% if categories %}
	            <ul>
	                {% for category in categories %}
	                <li>{{ category.name }}</li>
	                {% endfor %}
	            </ul>
	        {% else %}
	            <strong>No categories at present.</strong>
	        {% endif %}
	
	    </body>
	</html>

Here, we make use of Django's template language to present the data using ``if`` and ``for`` control statements. 

Within the ``<body>`` of the page, we test to see if ``categories`` - the name of the context variable containing our list - actually contains any categories (i.e. ``{% if categories %}''). 

If so, we proceed to construct an unordered HTML list (within the ``<ul>`` tags). The for loop ``{% for category in categories %}`` then iterates through the list of results, printing out each category's name ``{{ category.name }}`` within a pair of ``<li>`` tags to indicate a list element. 

If no categories exist, a message is displayed instead indicating so.

As the example shows in Django's template language, all commands are enclosed within the tags ``{%`` and ``%}``, while variables are referenced within ``{{`` and ``}}`` brackets. 

Now if you visit the index page (http://127.0.0.1:8000/rango/) you should see a list of three categories underneath the page title just like in Figure :num:`fig-rango-categories-simple`. 

.. _fig-rango-categories-simple:

.. figure:: ../images/rango-categories-simple.png
	:figclass: align-center

	The Rango homepage - now dynamically generated - showing a list of categories.


Creating a Details Page
-----------------------

According to Rango's specification, we also need to show a list of pages that are associated with each category.
We have a number of challenges here to overcome - we need to create a new view/page, we need to parameterise this view, and we need to create URL patterns and URL strings that encode the category names.

URL Design and Mapping
......................

Let's start by considering the URL problem. One way we could handle this problem is to use the unique ID for each category within the URL. For example, we could create URLs like ``/rango/category/1/`` or ``/rango/category/2/``, where the numbers correspond to the categories with unique IDs 1 and 2 respectively. However, these URLs are hardly human readable. Although we could probably infer that the number relates to a category, how would a user know what category relates to unique IDs 1 or 2? The user wouldn't know without trying. 

Instead, we could just use the category name as part of the URL. ``/rango/category/sport/`` should give us a list of pages related to the sport category. An even simpler approach would be to remove ``category`` altogether, leaving URLs such as ``/rango/fun/`` or ``/rango/sport/``. URLs like this are much nicer from a usability point of view because they are readable, meaningful and predictable. Of course, if we go this approach, we will have to handle categories which have multiple words, like 'Other Frameworks', etc.

.. note:: Designing clean URLs is an important aspect of web design. See `Wikipedia's article on Clean URLs <http://en.wikipedia.org/wiki/Clean_URL>`_ for more details.  


Category Page Workflow
......................

With our URLs design chosen let's get started, where our workflow will be as follows:

#. Import the Page model into ``rango/views.py``
#. Create a new view in ``rango/views.py`` - called ``category`` - The ``category`` view will take an additional parameter, ``category_name_url`` which will stored the encoded category name. 
	* We will need some help functions to encode and decode the category_name_url
#. Create a new template, ``templates/rango/category.html``.
#. Update Rango's ``urlpatterns`` to map the new ``category`` view to a URL pattern in ``rango/urls.py``.

We'll also need to update the index page view and index template to provide links to the category page view.

Category View
.............

In ``rango/views.py`` we first need to import the ``Page`` model so add the following import statement at the top of the file:

.. code-block:: python
	
	from rango.models import Page

Now, let's add our new view, ``category``:

.. code-block:: python
	
	def category(request, category_name_url):
	    # Request our context from the request passed to us.
	    context = RequestContext(request)
	    
	    # Change underscores in the category name to spaces.
	    # URLs don't handle spaces well, so we encode them as underscores.
	    # We can then simply replace the underscores with spaces again to get the name.
	    category_name = category_name_url.replace('_', ' ')
	    
	    # Build up the dictionary we will use as our template context dictionary.
	    context_dict = {'category_name': category_name}
	    
	    try:
	        # Can we find a category with the given name?
	        # If we can't, the .get() method raises a DoesNotExist exception.
	        # So the .get() method returns one model instance or raises an exception.
	        category_model = Category.objects.get(name=category_name)
	        
	        # Retrieve all of the associated pages.
	        # Note that filter returns >= 1 model instance.
	        pages = Page.objects.filter(category=category_model)
	        
	        # Adds our results list to the template context under name pages.
	        context_dict['pages'] = pages
	    except Category.DoesNotExist:
	        # We get here if we didn't find the specified category.
	        # Don't do anything - the template displays the "no category" message for us.
	        pass
	    
	    # Go render the response and return it to the client.
	    return render_to_response('rango/category.html', context_dict, context)

Our new view follows the same basic steps as our index page view. We obtain the context of the request, build a context dictionary, render the template, and send the result back. The difference here is that the context dictionary building is a little more complex - we need to check the database for the category we supply as argument ``category_name_url``, and build the context dictionary depending on the result we get. 

In constructing this view, we are making the assumption that we will be passed in the ``category_name_url`` so we will have to create a URL mapping to handle this, and we are also assuming that we have a template called ``rango\category.html`` which we will have to create as well.

You will have also seen in the ``category()`` view function we assume that the category_name_url is the category name where spaces are converted to underscores. And so we replace all the underscores with spaces. This is a pretty crude way to handle the decoding/encoding of the category name within the URL. As an exercise later it will be your job to create two functions to encode and decode category name.

.. warning:: While you can used spaces in URLs it is considered to be unsafe to use spaces in URLs (as pointed out in the `IETF Memo on URLs <http://www.ietf.org/rfc/rfc1738.txt>`_). 


Category Template
.................

Now let's create our template for the new view.  In ``<workspace>/tango_with_django_project/templates/rango/`` directory, create ``category.html`` and add the following code:

.. code-block:: html
	
	<!DOCTYPE html>
	<html>
	    <head>
	        <title>Rango</title>
	    </head>
	
	    <body>
	        <h1>{{ category_name }}</h1>
	
	        {% if pages %}
	        <ul>
	            {% for page in pages %}
	            <li><a href="{{ page.url }}">{{ page.title }}</a></li>
	            {% endfor %}
	        </ul>
	        {% else %}
	            <strong>No pages currently in category.</strong>
	        {% endif %}
	    </body>
	</html>

The HTML code example again demonstrates how we utilise the data passed to the template via its context. We make use of ``category_name``, and our ``pages`` list. If ``pages`` is undefined, or contains no elements, we display a message stating there are no pages present. Otherwise, the pages within the category are presented in a HTML list. For each page in the ``pages`` list, we present their ``title`` and ``url`` attributes.

Parameterised URL Mapping
.........................

Now let's have a look at how we actually pass the value of the ``category_name_url`` parameter to the ``category()`` function. To do so, we need to modify Rango's ``urls.py`` file and update the urlpatterns as follows:

.. code-block:: python
	
	urlpatterns = patterns('',
	    url(r'^$', views.index, name='index'),
	    url(r'^(?P<category_name_url>\w+)$', views.category, name='category'),) # New!

As you can see we have added in a rather complex tuple entry call ``category`` that will invoke  ``views.category()`` when the regular expression ``r'^(?P<category_name_url>\w+)$'`` is matched. We set up our regular expression to look for any sequence of word characters (e.g. a-z, A-Z, _, or 0-9) before the end of the URL, or a trailing URL slash - whatever comes first. This value is then passed to the view ``views.category()`` as parameter ``category_name_url``, the only argument after the mandatory ``request`` argument. Essentially, the name you hard-code into the regular expression is the name of the argument that Django looks for in your view's function definition.

.. note:: Regular expressions may seem horrible and confusing at first, but there are tons of resources online to help you. `This cheat sheet <http://cheatography.com/davechild/cheat-sheets/regular-expressions/>`_ provides you with an excellent resource for fixing pesky regular expression problems.

Modifying the Index View and Template
.....................................

Our new view is set up and ready to go - but we need to do one more thing. Our index page view needs to be updated to provide users with a means to view the category pages that are listed. Update in the ``index()`` in ``rango\views.py`` as follows:

.. code-block:: python
	
	def index(request):
	    # Obtain the context from the HTTP request.
	    context = RequestContext(request)
	    
	    # Query for categories - add the list to our context dictionary.
	    category_list = Category.objects.ordered_by('-likes')[:5]
	    context_dict = {'categories': category_list}
	    
	    # The following two lines are new.
	    # We loop through each category returned, and create a URL attribute.
	    # This attribute stores an encoded URL (e.g. spaces replaced with underscores).
	    for category in category_list:
	        category.url = category.name.replace(' ', '_')
	    
	    # Render the response and return to the client.
	    return render_to_response('rango/index.html', context_dict, context)

As explained in the commentary, we take each category that the database returns, then iterate through the list of categories encoding the name to make it URL friendly. This URL friendly value is then placed as an attribute inside the category object (i.e. we take advantage of Python's dynamic typing to add this attribute on the fly). 

We then pass the list of categories - ``category_list`` - to the context of the template so it can be rendered. With a ``url`` attribute now available for each category, we can update our ``index.html`` template to look like this:

.. code-block:: html
	
	<!DOCTYPE html>
	<html>
	    <head>
	        <title>Rango</title>
	    </head>

	    <body>
	        <h1>Rango says..hello world!</h1>

	        {% if categories %}
	            <ul>
	                {% for category in categories %}
	                <!-- Following line changed to add an HTML hyperlink -->
	                <li><a href="/rango/{{ category.url }}">{{ category.name }}</a></li>
	                {% endfor %}
	            </ul>
	       {% else %}
	            <strong>No categories at present.</strong>
	       {% endif %}

	    </body>
	</html>

Here we have updated each list element (``<li>``) adding a HTML hyperlink (``<a>``). The hyperlink has an ``href`` attribute, which we use to specify the target URL defined by ``{{ category.url }}``. 

Demo
....

.. _fig-rango-links:

.. figure:: ../images/rango-links.pdf
	:figclass: align-center

	What your link structure should now look like. Starting with the Rango homepage, you are then presented with the category detail page. Clicking on a page link takes you to the linked website.
	
Let's try it out now by visiting the Rango's homepage. You should see your homepage listing all the categories. The categories should now be clickable links. Clicking on ``Python`` should then take you to the ``Python`` detailed category view, as demonstrated in Figure :num:`fig-rango-links`. If you see a list of links like ``Official Python Tutorial``, then you've successfully set up the new view. Try navigating a category which doesn't exist, like ``/rango/computers``. You should see a message telling you that no pages exist in the category.

Exercises
---------

	* Modify the index page to also include the top 5 most viewed pages.
	* The encoding and decoding of the Category name to a URL is pretty sloppy. Create a better way for encoding/decoding the url/name so that it handles special characters and ignores case.
	* Now, instead of messing about with the url encoding/decoding in the View, fix your code to let the Model handle this responsibility directly.
	* Undertake the `Part Three of Offical Django Tutorial <https://docs.djangoproject.com/en/1.5/intro/tutorial03/>`_ if you have not done so already to reinforce what you have learnt here.

Hints
.....

	* Update the population script to add some value to the views count for each page.
	* Create an encode and decode function to convert category_name_url to category_name and vice versa.


