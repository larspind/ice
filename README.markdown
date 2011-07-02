#Ice Ice Baby!!!

The Ice system for templating allows people to serve Coffescript templates from Rails applications.  The advantage of this approach is that it is easy to serve these templates from a secure sandbox.  Therefore, your users can upload templates that are evaluated and served, but exist securely.

This is the approach taken by [Liquid](http://github.com/tobi/liquid) and some other template systems.  The advantage of using Ice for this is that you have a rich language at your disposal, as well as being able to provide your own functions that can be exposed to your users.

Ice builds upon [The Ruby Racer](http://github.com/cowboyd/therubyracer) (written by Charles Lowell).  This gem lets you use Google's V8 Javascript engine.

Ice allows you to write your templates in one of two formats.

  * [Eco](https://github.com/sstephenson/ruby-eco) (written by Sam Stephenson).  This gem allows you to use Coffeescript with HTML in an ERB-ish fashion.
  * [CoffeeKup](http://coffeekup.org/) (written by Maurice Machado).  This library uses Coffeescript itself to define your templates in a way reminiscent of Markaby and Haml.

 You can then write Eco templates like:

    <table>
        <tr><th>Name</th><th>Email</th></tr>
        <% for user in @users %>
            <tr>
                <td><%= user.name %></td><td><%= mailTo(user.email) %></td>
            </tr>
        <% end %>
    </table>

Eco-formatted files may also exist in your filesystem, provided they have a .eco extension.  Also, the templates may be compiled on demand with the method:

    Ice::Handlers::Eco.convert_template(template_text, variables)

The variables are whatever environment you want to pass in to the application.

The CoffeeKup equivalent to the above Eco template is:

    table ->
        tr ->
            th -> "Name"
            th -> "Email"
        for user in @users ->
            tr ->
                td -> user.name
                td ->  mailTo(user.email)

Similarly, these CoffeeKup files may exist on your filesystem provided they have a .coffeekup extension.

Eventually I'd like to bring in other template libraries, but this should suffice for now.

## Installation

Ice is currently being developed only for Rails 3.  Simply add to your Gemfile

    gem 'ice'

Ice is undergoing *very* active development so be sure to either use the most recent gem, or pull from master.

## to_ice

Every object is revealed to the templates via its to_ice method.  This helps sanitize the objects that are passed into Ice, so people editing the template only have access to a limited subset of the data.

Instances of some classes like String and Numeric just return themselves as the result of to_ice.  Hashes and Arrays run to_ice recursively on their members.

If you want an object to map to a different representation, simply define a to_ice object that returns whatever object you want to represent it within the Eco template.  These objects are referred to as "Cubes", and are equivalent to "Drops" for those used to Liquid.

## ActiveModel and to_ice

To make life easy, since most complex objects passed to the templates will be classes including ActiveModel::Serializable, the default to_ice behaviour of these classes is to pass itself in to a class with the same name, but followed by the word "Cube".

Therefore calling to_ice on an instance of a User class will invoke

    UserCube.new self

## BaseCube Class

You can have your cubes inherit from our Ice::BaseCube class.  Your cubes inheriting from it can then determine what additional attributes they want to reveal.  For example

    class BookCube < Ice::BaseCube
      revealing :title, :author_id, :genre_id

      def reviewer_names
        @source.reviewers.map(&:name)
      end
    end

would provide a cube with access to the title, author_id and genre properties of the underlying ActiveModel.  In addition, it exposes a reviewer_names function that uses the @source instance variable to get at the record which is being filtered.  Note that if no call to `revealing` occurs, the cube generates a mapping for the `@source` object's serializable `attributes`.

These cubes also have simple belongs_to and has_many associations, so you can write things like:

    class ArticleCube < Ice::BaseCube
      has_many :comments, :tags
      belongs_to :author, :section
    end

This generates association helper functions such as comment_ids, num_comments, has_comments, comments, author_id, and author.

Note that the results of all associations and revealed functions are also sanitized via to_ice.

## Partials

Partials may now be written in Eco, and included in ERB (and other) templates.


## Helpers

Two global arrays exist named `IceJavascriptHelpers` and `IceCoffeescriptHelpers`.  If you add to those arrays strings of Javascript or Coffeescript, those strings will be included in your views.  These string are also compiled in the case of Coffeescript.

This is slightly hackish, so expect this approach to shortly be replaced with a better one.  But it is a decent way to add helpers to your Eco file.

## NavBar

To make it easier to generate links, we added a `navBar` helper.  For Eco templates it appears as:

    <%= navBar (bar) => %>
        <%= bar.linkTo("Bar", "/foo") %>
        <%= bar.linkTo("http://ludicast.com") %>
    <% end %>

and in CoffeeKup the navBar is written as:

    navBar (o)=>
        o.linkTo("Bar", "/foo")
        o.linkTo("http://ludicast.com")

In either case this generates the following html

    <ul class="linkBar">
        <li><a href="/foo">Bar</a></li>
        <li><a href="http://ludicast.com">http://ludicast.com</a></li>
    </ul>

The `navBar` helper also takes options so if the Eco above was instead instantiated with:

    <% opts = nav_prefix:'<div>', nav_postfix: '</div>', link_prefix: '<span>', link_postfix: '</span>' %>
    <%= navBar opts, (bar)=> %>

it would generate

    <div>
        <span><a href="/foo">Bar</a></span>
        <span><a href="http://ludicast.com">http://ludicast.com</a></span>
    </div>

Also, if you want to make a site-wide change to the default NavBar settings, all you need to do is add these options to the NavBarConfig class like

    coffeescript = <<-COFFEESCRIPT
      NavBarConfig =
        navPrefix: "<div>",
        navPostFix: "</div>",
        linkPrefix: "<span>",
        linkPostFix: "</span>"
    COFFEESCRIPT
    IceCoffeescriptHelpers << coffeescript

Then all links will generate with these options, unless overridden in the values passed in to `navBar`.

## Routes

Assuming that all your cubes are models that you are exposing to your app, we add to your Eco templates routing helpers for every class inheriting from BaseCube.  Therefore, if you have a cube class named `NoteCube`, you will have the following helper methods available:

    newNotePath
    notesPath
    notePath(@note)
    editNotePath(@note)

which are converted to the appropriate paths.

Note that some people might claim that it is insecure to expose your resources like this, but that probably should be dealt with on a case-by-case basis.  Besides, the fact that you are exposing these resources as cubes means that you are, well, already exposing these resources.

## Note on Patches/Pull Requests

* Fork the project.
* Make your feature addition or bug fix.
* Add spec for it. This is important so I don't break it in a future version unintentionally.  In fact, try to write your specs in a test-first manner.
* Commit
* Send me a pull request.

## Todo

* Add in form builders (from clots project)
* Haml support
* Use [Moneta](http://github.com/wycats/moneta) for caching autogenerated javascript files.
* Allowing Ice to render Rails partials
* Allowing Ice to serve as Rails layout files.

## Copyright

MIT Licence. See MIT-LICENSE file for details.