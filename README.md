# CPSC449-Project4

## Project Members
* Brandon Le (Ble2306@csu.fullerton.edu)
* Austin Sohn (austinsohn@csu.fullerton.edu)
* Vien Huynh (Squire25@csu.fullerton.edu)
* Stephanie Cobb (stephanie_cobb@csu.fullerton.edu)

## How to create the database and start the services
To create the databases, you can use the init.sh to create the users.db and posts.db
In the terminal, make sure you're in the correct file directory and type:
```
chmod +x init.sh
sh init.sh
```
Alternatively, you can run these commands in the command line:
```
# create database via csv files
sqlite-utils insert ./var/users.db users --csv ./share/users.csv --detect-types --pk=user_id
sqlite-utils insert ./var/users.db follows --csv ./share/follows.csv --detect-types --pk=id
sqlite-utils insert ./var/posts.db posts --csv ./share/posts.csv --detect-types --pk=id

# add foreign keys (optional)
sqlite-utils add-foreign-key ./var/users.db follows user_id users user_id
sqlite-utils add-foreign-key ./var/users.db follows following_id users user_id
```

To start the services in production, run these commands in separate command lines:
```
# Run foreman instances -- cd to api directory first!
foreman start -m users=1,timelines=3,registry=1,polls=1,likes=1,async_post=1,validate_like=1,validate_poll=1

# Run haproxy load balancer -- cd to ./etc/haproxy
haproxy -f haproxy.cfg

# Run DynamoDB local server -- cd to DyanmoDB folder
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb -port 8001

# Run email server
python3 -m smtpd -n -c DebuggingServer localhost:1025
```
The web application should now be able to run on **localhost:8000**.

## REST API documentation for each service

### Microblogging services:
1) users.py
    - **getUserID(db, username)** -- local python function to find userID given username and db
    - **/users/login/{username}** -- called by get request from custom_verify in timelines service to find password given username
    - **/users** -- returns all users' public info
    - **/users/{username}** -- returns specific user's public info
    - **/users/{username}/followers** -- called by get request from timelines service to find user's followers
2) timelines.py
    - **custom_verify(username, password)** -- hug.cli() function that request to "users/login/{username}" endpoint to verify password
    - **/timelines/{username}** -- returns specific user's own posts
    - **/timelines/{username}/{post_id}** -- returns specific post from the specific user
        - ex: the url "/timeline/brandon2306/1" should bring up the user's first post
    - **/timelines/{username}/post** -- requires auth, create post as a certain user (**authentication needed**)
    - **/home/{username}** - requires auth, access user's home page and followers' posts
    - **/public** -- returns all existing posts
3) likes.py
    - connects to redis database at port 6379 locally
    - **{liker_username}/{username}/{post_id}** -- create a like on a specific post
    - **/likes/count/{username}/{post_id}** -- get request to see how many likes a specific post received
    - **/likes/{liker_username}** -- get request to retrieve a list of posts a specific user liked
    - **/likes/popular** -- get request to see a list of popular posts liked by many users
4) polls.py
    - connects to Dynamodb instance at port 8001 locally and creates table if not existed
    - **/polls** -- returns all existing polls
    - **/polls/create** -- POST method that creates post given username, question, and list of responses
        - Unsure about the syntax for inputing a list of responses in terminal. For exmaple, the terminal interpets "[1,2,3,4]" as one element for response
    - **/polls/vote/{poll_id}** -- POST method where user can vote a response from a certain post given username, post_id, and the specific response
        - To choose response, it is numbered from 0-3 
        - User cannot vote twice
5) registry.py
    - the registry assumes the url provided by registered services has a "/{servicename}/health" route that can return a 200 response to GET requests
    - **/registry** -- returns all instances of all registered services
    - **/registry/{servicename}** -- returns all instances of servicename
    - **/registry/{servicename} POST** -- registers an instance of servicename. The url is provided in the format: text="url"
        - each service also has a self-register function that happens at startup when initiated by hug
6) async_post.py
    - consume job from "posts" watchlist, insert new post to sqlite db
8) validate_like.py
    - consume job from "likes" watchlist, validate if post is legit, undoes like and sends "Invalid post" email to user who posted if post does not exist
10) validate_poll.py
    - consume jobs from "polls" watchlist, validate if poll url is legit, sends "Invalid poll" email to the user who posted if poll does not exist

## Performance Testing Results
1) Post without validation
    - **SYNC POST:** Total:	0.6918 secs
    - **ASYNC POST:** Total: 0.6150 secs
2) Post with validation
    - **SYNC POST:** Total:	0.7258 secs
    - **ASYNC POST:** Total: 0.4725 secs

For more information, look to the "post without validation.txt" and "post with validation.txt"

## Examples
You can also create a post as a certain user by running these commands in a new command line:
```
# creating post
http -a bob123:hello123 localhost:8000/timelines/bob123/post text="today sucks!"
http -a bob123:hello123 localhost:8000/timelines/terraria2/post text="i like games"
http -a bob123:hello123 localhost:8000/timelines/brandon2306/post text="i agree!" url="http://localhost:8000/timeline/bob123/1"

# post with an invalid poll
http -a bob123:hello123 localhost:8000/timelines/brandon2306/asyncpost text="i agree! http://localhost:8000/polls/10"
```

```
# liking post
http POST localhost:8000/likes/bob123/brandon2306/1

# popular posts
http GET localhost:8000/likes/popular

# liking an invalid post
http POST localhost:8000/likes/bob123/brandon2306/25

```

```
# examples using hey:
hey -m POST -H "Authorization: Basic $(echo -n bob123:hello123 | base64)" http://localhost:8000/timelines/bob123/asyncpost text="test http://localhost:8000/polls/6"
hey -m POST -H "Authorization: Basic $(echo -n bob123:hello123 | base64)" http://localhost:8000/timelines/bob123/post text="test http://localhost:8000/polls/6"
```
## NOTE:
Other than the syntax for inputing a list of responses for polls.py, all the features were implemented and should be working correctly.
