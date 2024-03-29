# Programmatic Webserver
*An API with an API*
- Load request data from memory, from filesystem, or from URI - then transform and serve!
- Upload a Putfile uploaded in the Body of a PUT request, with Actions that are performed on each GET, and control permissions foreach URI resource. Use the Everyone keyword for anonymous access.
- Each path can respond differently. Serve paths with static content locally, forward paths with dynamic content across an array of other hosts, and host a cached copy of remote data on another path - all in the same container.
- Use __Path Variables__ to call another internal path. These use percents instead of path slashes, such as %increment below. This helps prevent confusion with Divide functionality explained further down.
- Example Putfile:
        {
          "URI": "/increment",
          "Action": "data~%increment+1",
          "Owner": "BobbyTables",
          "AccessList": {
            "Everyone": \[
              "GET",
              "HEAD",
              "OPTIONS",
              "POST",
              "PUT",
              "DELETE",
              "MERGE"
            \]
          },
          "notes": "",
          "Data": ""
        }


### Verb Uses
- GET - Performs specified Action.
- HEAD - Normal functionality
- OPTIONS - Normal functionality
- POST - Performs the "login" action if set. Otherwise it appends the request body to the Data parameter of the Putfile. This is often served by a GET request, either directly or after transformation. 
- PUT - Upload a Putfile, like the one just above. The Data parameter can be filled in, giving another way to populate this parameter alongside a POST request.
- DELETE - Deletes the path's Putfile. 
- MERGE - Read only display of Putfile. (I ran out of verbs.)

### Available Actions

- uri
	- Format: "uri\~FullPath,Or,List,Of,Full,Paths\~cacheExpiry"
	- Performs the requested verb against the requested resource or resources, stores the data in the Putfile Data property, then serves from there. 
	- All examples here only GET the remote resource, but all standard HTML verbs are available. 
	- Automatically caches single-site Actions. Multi-site caching, and single-site cache invalidation & cache expiry functionality to come.
- fs
	- Format: "fs\~/path\~cacheExpiry"
	- Loads a file from the filesystem, stores it in the path's Data property, and serves it from there.
	- Caches indefinitely. Cache invalidation, and expiry functionality to come.
- data
	- Formats:
		- "data\~%path\~cacheExpiry"
		- "data\~%path %otherPath\~cacheExpiry"
		- "data\~%path mathOp %otherPath\~cacheExpiry"
		- "data\~%path mathOp integer\~cacheExpiry"
		- "data\~integer mathOp integer\~cacheExpiry"
		- "data\~%path &lt;htmlTag&gt;ArbitraryTextGoesHere&lt;/htmlTag&gt; %otherPath\~cacheExpiry"
	- "mathOp" here is short for mathematical operator, represented by the common symbols "+", "-", "\*", and "/". Currently only the basic 4 operations of addition, subtraction, multiplication, and division are supported, but plans are to make this section much more robust and use a C-ish language.
	- Populates the Path Variables with the Data property of their Putfile, then performs any math operations in the Action. Stores the output in this path's PutFile's Data property before responding with it.

## Filesystem source

- Filesystem source is the default Action for paths. 
- Format: fs\~/file.ext

#### Example: Serve index.html as root.

- Method: Put
- Location: http://localhost/
- Body: 
	1. URI:  / 
	2. Action:  fs\~/index.html
	3. Owner:  BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes: 
	6. Data: 

##### About the Body:
- Putfile must be in JSON format. (Sorry YAML fans.)
- Size limit coming shortly. Please keep these sensible and short until then. 
1. URI in field must match the upload path, as a validation check. It's local to the host for now, but this might change in the future to provide multi-domain support. Here, the root URI / is used. 
2. Action is populated with Action fs\~/sub/pathname.ext if the page hasn't been visited before. 
	- File is read from filesystem into the Data field of the Putfile, then read from the Putfile into the response. 
	- But if the page exists and the Putfile Action has been changed to blank, will just try to respond with the Data field and won't check the filesystem.
	- Content Type of response is determined by the file extension in the Action. So if you POST HTML at \test.txt, it will show up in the browser as plaintext HTML, not as a rendered webpage.
3. Owner can always perform GET and PUT in any situation, so the can always control that path. More functionality is planned around this field. 
4. AccessList controls which verbs will respond to which users.
    - The Everyone keyword is used to control access for all users, including not logged in. 
    - So if you specify that Everyone can GET, and you can only POST, then you'll be able to both GET and POST.
5. Notes may contain user-specified data, but some functions (currently only load balancer) use them to store data too, so don't remove what you didn't add. 
6. Data is populated from one of several Actions, then served as part of a GET request.

## URI source

#### Cache 1 remote site (CDN presence edge)

- Method: Put
- Location: http://localhost/CdnEdge.html
    1. URI: /CdnEdge.html
    2. Action:  uri\~GET\~https://www.Gilgamech.com\~0
    3. Owner: BobbyTables
    4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
    5. notes:
    6. Data: 

A single URI in the Action causes the URI to become cached in the Data property of that path's Putfile on the first GET, and served from the Putfile on subsequent GETs. Caching functionality is still in progress.

#### Load-balance between 2 websites

- Method: Put
- Location: http://localhost/LoadBalance.html
    1. URI: /LoadBalance.html
    2. Action:  uri\~GET\~https://www.Gilgamech.com , https://Gilgamech.Neocities.org\ ~0
    3. Owner: BobbyTables
    4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
    5. notes:
    6. Data: 

The Putfile Action can hold numerous URIs. Currently only Round Robin load balancing is available, but weighed and other types are planned. For traditional load balancing, these would be internal locations, such as https://server1/ and https://server2/. External URIs with very different pages are being used here mostly for demonstration.

## In-memory source

#### Increment on GET

- Method: Put
- Location: http://localhost/increment
	1. URI: /increment
	2. Action: data\~%increment+1
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1

The Action specifies to take the value stored in the Data property of the /increment path's Putfile, adds one to it, stores it in the current path's Putfile Data property, then responds with it as the body. This makes a very good website visitor counter. And it can also call different paths, add two paths, and add two static numbers. It only takes one function per Action currently - nesting Actions are coming soon.

#### Decrement on GET

- Method: Put
- Location: http://localhost/decrement
- Body: 
	1. URI: /decrement
	2. Action: data\~%decrement-1
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1000000

This path's Action specifies to take the /decrement Putfile's Data property and subtract one. The use case is less clear here - maybe as a countdown?. As before, this works with other paths and also with static numbers.

#### Multiply on GET

- Method: Put
- Location: http://localhost/multiply
- Body: 
	1. URI: /multiply
	2. Action: data\~%multiply*2
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1

This path's actions multiply. Same situations and caveats as previous. Useful as a conversion tool - POST a number to this path and GET the converted number.

#### Divide on GET

- Method: Put
- Location: http://localhost/divide
- Body: 
	1. URI: /divide
	2. Action: data\~%divide/2
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1000000

This path divides. Same situations and caveats as previous. Also useful as a conversion tool.

#### Data concatenation

- Method: Put
- Location: http://localhost/RobbieRotten.html
	1. URI: /RobbieRotten.html
	2. Action: data\~%header.html &lt;div class="textBubbleBG"&gt;&lt;h1&gt;We Are Number 1!&lt;/h1&gt;&lt;/div&gt;  %footer.html
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 

This Action shows the real power of Path Variables, by creating a webpage with one line of Action code. First it replaces the %header variable with your webpage's header and menu section that you've already stored at /header.html. Then it adds the short message, referencing a CSS class in a file loaded by the header. Finally, the website footer is loaded from its Putfile at /footer.html, before storing the page in RobbieRotten.html's Putfile and then responding with it. This could be combined with transforming the data with an above function before serving. Future updates will increase the functionality and robustness of this dynamic page generation method. 


## Validation Testing

#### URI mismatch test

- Method: Put
- Location: http://localhost/test
	1. URI: /decrement
	2. Action: data\~%decrement-1
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1000000

This test should fail, since the URI specifies the /decrement path, but the location is for the /test path.

#### User-specific permissions

- Method: Put
- Location: http://localhost/test2
	1. URI: /test2
	2. Action: data\~%test2/2
	3. Owner: BobbyTables
	4. AccessList: BobbyTables: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 1000000

This test lets you test the permissions system by trying verbs as Everyone, which should all fail. We'll use this again after logging in.

## Login Functionality

#### Login Page Setup

- Method: Put
- Location: http://localhost/login
	1. URI: /login
	2. Action: login
	3. Owner: BobbyTables
	4. AccessList: Everyone: \[GET HEAD OPTIONS POST PUT DELETE MERGE\]
	5. notes:
	6. Data: 

The login Action here pushes POST requests on this path down the Login code path, instead of populating the Data parameter. 

#### Login Request

- Method: Post
- Location: http://localhost/login
- Body:
	- username: BobbyTables
	- password: HorseBatteryStapleCorrect
	- emailAddress: (optional)

Here the Body won't be uploading a Putfile, but instead authentication data. The server uses the same process for login and sign up - if the user isn't in the data, add them with the password they sent. If they are in the data, check the password. Email address would be used for password reset, but that functionality requires email integration, which might take longer than other planned features.

#### Login Response

- Method: Response
- Location: http://localhost/login
- Body: Bearer 133770ce9b1201375deba7ec0ce01223251

Response body will include the Bearer token, which is the word "Bearer" and a space followed by a random string from a cryptographic module. All subsequent requests must include this token in a header named "token" to receive a logged-in response.

#### Login Test

- Method: Get
- Location: http://localhost/test2
- Headers
    - token: Bearer 133770ce9b1201375deba7ec0ce01223251

Reusing the user-specific permissions test from above - this time we're logged in as the user with access (BobbyTables) so we should get a 200 OK response, with 500k in the body (since it performed a division operation on its data).

## Planned features:
- Modify header page script to put the token in the cookies and send back with every request - and say like "Hello userName" or "Click to Login" to verify state.
- Hash passwords
- Add LastCacheUpdateTime to Notes to support cache expiry. 
- Owner can only be changed by Owner.
	- Should Owner have other controls on page?
- Save the individual POSTed file, along with the cache.
- Put size checking (what's a good max size? 10 actions?)
- List of vars to move to the top and set as defaults
	- Everyone verbs
	- Login expiry
- More robust data Actions, including IF
	- on false, delete if and following sections, then evaluate else.
	- on true, delete if and else sections, then evaluate following.
	- Nested math operations
- mq Action - POST appends, GET shifts the top item into the response. 
- HTTPS and Let's Encrypt
- Better file checking? const files = fs.readdirSync(wwwFolder);
- Other LB modes
- Replace comma-split list of sites with semicolon-split list of actions. (Optional - would allow a path to round-robin between local files, remote sites, and transformed data in a bewildering series of responses. Unsure of the use case here.)
- API Keys
- Payload storage.
- Messaging relay.
- IsItUp?
- Improve login (Need to set up logout, password reset, and delete account. Then Passport or similar?)
- Chat
- Ticketing system
- Shopping cart?
- Time tracker 
- MFA OTPs
- Add Password Reset (Choose an email provider etc.) and logout? functionality. (Why logout? Feel free to delete the cookie.)
