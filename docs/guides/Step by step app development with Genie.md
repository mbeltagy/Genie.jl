# Scaffolding our app
The first thing that we need to do is to setup the file structure of our app. Genie uses the "convention over configuration" design pattern, preferring to use sensible defaults represented by files of certain structures and in certain locations. Genie however does not require setting up these files ourselves - instead, it provides a rich set of generators for scaffolding every component of a web app.

The only requirement to get things started is installing Genie itself:
```julia
julia> using Pkg
julia> Pkg.clone("htts://github.com/essenciary/Genie.jl")
julia> using Genie
```

Now we can ask Genie to scaffold our app -- which we'll name "Chirper".
```julia
julia> Genie.REPL.new_app("chirper")
```

You'll see output in the console informing you about the status of the operation:
```julia
info: Done! New app created at /Users/adrian/Dropbox/Projects/chirper
info: Looking for dependencies
info: Checking for Flax rendering engine support
info: Finished adding dependencies
info: Starting your brand new Genie app - hang tight!
```

Once the app is ready, it will be automatically started. This means that:
* it will automatically `cd()` into the app's folder
* will load the Genie app environment
* will take you to a Genie REPL -- which is a Julia REPL, so you have all the power of Julia at your disposal
* the Genie REPL is indicated by the custom prompt `genie ❱❱`

We can check that everything worked well by starting up a server and taking it for a spin:
```julia
genie> server = AppServer.startup()
```
You can now visit `http://localhost:8000` in your browser - you will be welcomed by our very smart and helpful genie.

___

# Setting up the home page
Genie uses a route file to map web requests to functions. These functions process the request info and return the response. The route are defined within the `config/routes.jl` file. Right now, the file looks like this:
```julia
using Router

route("/") do
  Router.serve_static_file("/welcome.html")
end
```

At the top it brings the `Router` module into scope. Then it defines a `route` for the root page `/` -- which is handled by an anonymous function which calls `Router.serve_static_file`. The `serve_static_file` function returns the content of the `welcome.html` static file. This is the file we have seen when we accessed our website.

Let's replace the default Genie home page with a simple custom one. This time we'll do it the manual way so you can understand how things work - but in the future we'll use Genie's generators. We need to setup a new folder under `app/resources` -- let's call it `home`, cause it's for our _home_ page. And under `home`, add a `views` folder. You can use your file manager if you want -- I'll use Julia/Genie:
```julia
genie> mkpath("app/resources/home/views")
```

Inside views, let's add the file for our home page. Genie supports both HTML and Markdown pages. In this case, let's setup a simple markdown page called `home_page.md`:
```julia
genie> touch("app/resources/home/views/home_page.md")
```

We can now edit it:
```julia
genie> edit("app/resources/home/views/home_page.md")
```
This command will open the "home_page.md" file with your default editor. Add the following content, save the file and close the editor when done:
```markdown
# Welcome to Chirper!
Chirper is a cool website which allows you to post short public messages for the Chirper community -- and to follow other chirpers and see their messages!
```

Finally, we need to edit our routes file. We can no longer use the `serve_static_file` function because the markdown file needs to be parsed and converted into HTML. Instead, we'll call the `respond_with_html` function -- which takes at least two arguments: the name of the resource (in our case `home`, the name of the folder) and the name of the view file (in our case, `home_page`).

---

#Configuring the database
SearchLight, Genie's ORM layer works transparently with all major relational databases supported by Julia. Thanks to its DSL for managing and querying databases, you can, for instance, prototype on SQLite and deploy on MySQL. Let's add a SQLite backend for our app.

For start, make sure that you have the `SQLite.jl` package installed. You can check the package's README at https://github.com/JuliaDatabases/SQLite.jl.

Once you have it, edit the `config/database.yml` file:
```julia
genie> edit("config/database.yml")
```
The file contains placeholders for database configuration for each of the three environments: dev, test and prod. Our application runs in dev mode, and that's what we'll need to configure. Setup the `adapter: ` key to `SQLite` and the `host: ` to `db/dev.sqlite`. You'll have to restart the Genie app so the changes are loaded. Close the terminal window (or the Julia process). Then make sure you open the terminal in the app's root (or `cd` into the app's root). You can load a Genie app with:
```
$ bin/repl
```
Now that we're back into the app's environment, with the database configured, let's allow Genie to set things up. This needs to be run only once, once we've configured a new database:
```julia
genie> Genie.REPL.db_init()
```
This creates the database, if it does not exist, at `db/dev.sqlite`. And then creates a new table within the database -- this table, called `schema_migrations`, is used for database versioning and schema management.

---

#Working with resources
The concept of resource is central to Genie apps. A resource is a "thing" - a business object which is accessible over the internet. Such resources can be created, read, updated and deleted (in what is called a CRUD workflow).

In order to implement a complete CRUD workflow, the full MVC stack is involved. We'll need routing, controller files, models (and the underlying database table), views -- and optionally, model data validators and controller access rules. But don't worries, we don't need to create all these files by hand: we have a powerful genie sidekick.

We can ask Genie to create a new _chirp_ resource -- which will represent a user message in our system.
```julia
genie> Genie.REPL.new_resource("chirp")

info: New model created at /Users/adrian/Dropbox/Projects/chirper/app/resources/chirps/Chirps.jl
info: New migration created at /Users/adrian/Dropbox/Projects/chirper/db/migrations/20180311124824247_create_table_chirps.jl
info: New ChirpsValidator.jl created at /Users/adrian/Dropbox/Projects/chirper/app/resources/chirps/ChirpsValidator.jl
info: New chirps_test.jl created at /Users/adrian/Dropbox/Projects/chirper/test/unit/chirps_test.jl
info: New ChirpsController.jl created at /Users/adrian/Dropbox/Projects/chirper/app/resources/chirps/ChirpsController.jl
info: New ChirpsChannel.jl created at /Users/adrian/Dropbox/Projects/chirper/app/resources/chirps/ChirpsChannel.jl
info: New authorization.yml created at /Users/adrian/Dropbox/Projects/chirper/app/resources/chirps/authorization.yml
info: New chirps_test.jl created at /Users/adrian/Dropbox/Projects/chirper/test/unit/chirps_test.jl
```
Genie creates the full range of MVC files. We'll cover each one of them as we'll use them to develop our app.

---

#Database versioning with migrations
SearchLight, Genie's ORM layer comes with database migration functionality. Migrations are scripts used to change the database -- by creating and altering tables, for example. These scripts are put under version control and shared with the whole development team. Also, using the migration's API, the changes can be managed properly (for instance, they need to be run in the proper order).

Asking Genie to create a new resource has added a new migration. It will be called {timestamp}