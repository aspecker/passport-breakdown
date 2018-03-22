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