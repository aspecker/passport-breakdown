# Reverse Engineering Script of Sequelize Passport Demo
## Adam Secker GWU Boot Camp March 2018

### Instructions
* write a reverse engineering script to explain the functions of the components of sequelize-passport demo
* be thorough, research and cite any concepts that aren't familiar
* use this as an opportunity to fully grasp each element of the MVC model

#### Description
1. Assessment
* looking at the file structure, I see config, models, public, routes and node_modules as the main directories
     * presence of node_modules indicates we are using node.js
     * will open package.json to see start paths and dependancies used

2. Package.json
 * by examining the main path, we can see this app begins at server.js, and is where we should start reverse engineering
 * dependencies used indicate functions
    * express, body-parser, and express-session indicate the add will be routing to pages and processing data from APIs
        * session is used to store information server side for the duration of a user's session
    * mysql and sequelize imply this app uses a database to store some kind of information
    * passport is unfamiliar to me: first look to package documentation: http://www.passportjs.org/
        * this package is used to authenticate user requests in an unobtrusive manner
        * each authentication need represents a strategy, with an individual package for each
        * general model uses passport.authenticate('strategy') to authenticate the requests
        * this is followed with a riderect to a page based on authentication status
            * often accompanied with a flash emssage indicating status to the user
        * other options that are used with passport
            * disable session: prevents passport from establishing persistent login session
            * custom callback: if built-ins arent enough, they can be customized
        * configuration: requires strategies, middleware, and (optional)sessions
            * strategies: in this app, this is found in config/passport
                * require a verify callback to check user credentials
                * middleware of initialize and session are used 
            * using passport-local for username/password authentication
                * takes input from a form, and routes with a post request
                * passport.authenticate('local, {callback}) serves as the route
                    * username and password are default expected fields
    * the last dependancy is bcrypt, which is also new see: https://www.npmjs.com/package/bcrypt
        * package is used to encrypt information, and may only work with even(stable) versions of node
        * async method of using bcrypt is preferred
 * that settles it for the package.json, let's head to server.js as the main path indicates

3. Server.js
* at the top, we see our main dependancies required
    * express, body-parser, express-session, and passport required from our configs folder
* next we see the port defined, and our database imported from models
* following, express is declared as app
* we then execute a series of functions on app using express, passport, and body-parser
    * bodyParser urlencoded is set to false, which means it will only process strings and arrays
        * .json() is also called which allows it to process .jsons
    * express.static('public') allows use to serve all documents in our public folder
    * session middleware is used, indicating the cookie.secure format
    * we intialize passport, and then call the session method which converts the user session id from cookie into the deserialized user object
* next step is the typical requirement of api and html routes
* last we will sync with databse before connecting to the server, using a promise
* there are several paths to follow here: routes, models, or config/passport
    * we will invesigate config because data needs to be authenticated using passport when accessing or writing the database

4. Config directory
* contains one directory, middleware, and a config.json and a passport.js
* config.json is a standard file to allow us to connect to a mysql database
* middleware folder contains one file: isAuthenticated.js
    * this appears to be a simple file that serves to handle the ridirect in passport
    * if the request is succesful, uses the next method as part of the callback, otherwise ridirect to root
* last we look at passport.js, which requires the passport package and constructs a LocalStrategy object
    * we also require the db from models
    * passport.use new LocalStrategy invokes the core authentication logic
        * takes in a username
        * runs a callback to find the username in db
        * afterwards, a promise contains the logic to authenticate the user
            * if the user is not present in the database- deliver a message if invalid username
            * else if the user's password is not present, using validPassword - display message
            * otherwise return on dbUser
    * next two callbacks are used for serializeUser and deserializeUser
        * used to handle user session status see: https://stackoverflow.com/questions/27637609/understanding-passport-serialize-deserialize
    * finally we export the code in here as passport
* the models directory was required as our database, so that is a natural place to head next

5. Models Directory
* models contains two files, index and user
* we will first look at the user.js file
    * first bcrypt is required outside of the export
    * inside the export, we use sequelize to define the user and its email and password objects
    * outside the callback, we call User.prototype.validPassword() and use bcrypt.compareSync() 
    * this function will check if the submitted password is the same as the one in the database
    * next we call User.hook('beforeCreate', callback(user)) using sequelize's hook method
        * this runs a function before the data entry is created
        * bcrypt.hashSync() is responsible for encrypting the user's password
* next we will look at index.js    - NEEDS MORE EXPLANATION
    * first we set up and require packages
        * in here, we require fs, path, Sequelize and set them to variables
        * next we use path.basename to set the bast path for our model
        * env is set to process.env.NODE_ENV or 'development'
        * config is required from its path in the config directory
        * and an empty database object is created
    * next we have an if statement to detect if we are using an env variable (like on heroku), and use that with sequelize
        * else we will use the local paths specified in our config.json
    * fs.readdirsync is used here to synchronously read the contents of a directory, in this case the models directory
    * filter is used to return all the files in the directory
    * forEach imports the files using sequelize 
    * a constructor object uses the keys method to search over the keys from the model
        * associate method I have no idea what it's doing here
    * last three lines export the db using sequelize
* the next file to study is less clear than in previous sections
    * in the models folder we have very few routes outside the directory
    * we are somewhat at an impasse here. but where is passport required?
        * we saw it in the server, but that won't help us here
        * however, because we know passport authenticates data **when it passes from the client to the server**
            * we can infer that it will be in the file involved in this aspect of the app's functioning
            * for most purposes, requesting data from a website would not require authentication
                * so we aren't looking for get requests
            * it will most likely be involved in the changing or adding of data
                * so we're post or put requests, it is. into the api routes!
        * there is another path we could have taken
            * when we looked into middleware in our config, we saw that the redirect method went to ('/)
            * we know this is an html route that refers to the root of our site 
            * we could then have gone from the html routes into the api routes
            * from the data flow perspective, it makes sense to head to api routes first

6. Routes Directory
* inside api routes, we do in fact see passport required at the beginning, along with our models
* before looking deeper into this page, we want to be thinking about two things
    * where these routes are headed to
        * this will allow us to follow the user flow through the page
    * how these events are triggered will help us anticipate client side script
* we see our first route, a post, which is where we are using passport
    * this simple route uses passports authenticate function
        * heads to members page
        * likely triggered on user clicking login button
* our second route is a post route that results from user signup
    * it uses our model to create an object in our api
        * bcrypt acts here through the user object to encrypt the password
    * then we route to the login method described above
* the next route is very simple - a redirect route to main after using logout
* the last route is specific to the function of this app
    * if the request is a user (not what this req.user comes from, if it came from the model i'd expect it to be capitalized)
    * returns the api of user data
    * otherwise, it seems to check if the user is logged in?
        * **seems simple** but what exactly is happening here?
* as that was the end of our api routes, next we'll head to html routes
* interestingly we require isAuthenticated here
* our three html routes here are very simple
    * they serve our three html pages
    * note that isAuthenticated is called when reaching the members page
* that's it for our routes, last folder is public

7. Public Directory
* we will look at the javascript files and html files in tandem to trace their functionality
* login - will handle our login page and with its accompanying post request
    * sets variables to login form, input email and password
        * we can infer these are the elements tied to the corresponding script
    * on form submit, gather user data, fun loginuser function
        * this is where the ajax call of our post request is made
* members - this displays the list of members  
    * not much going on here, as the heavy lifting has been done further back
* signup - this file operates extremely similarly to the login form
    * note they both use posts requests
    * also note the alert function present to handle login errors

8. Summary
    * that settles our route through the paths of this simple MVC app
    * to reiterate our path in extreme short hand
        * we started at the server, which pretty much serves as the middle point to the app
        * we went into our models, which determined how our data would be processed for it (in this case we have yet to receive client side data, but we assume its there) to be ready for the database
        * we went to config, which contained our connections to mysql and passport for authenticating the data sent to and retrieved from the database
        * we went to our api routes, which would pass data from the client to our models
        * we went to the client side js, which was tied to events and made the ajax calls 
        * we went to the html pages, where the users would submit data that would flow down this path
    * it would also have been feasible to start at the html pages and trace the data back as if it was a login that had been sent by the user
    * im impressed by the server - in the end its whats tying all these functional components together, even though it is not doing much by itself