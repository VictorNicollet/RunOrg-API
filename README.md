This is a request for feedback and comments. Below, you will find a tentative description of the general principles of an API for one of my projects. I would be glad if you took some time to read through this and provide some feedback [here](https://github.com/VictorNicollet/RunOrg-API/issue), especially if you have experience building APIs or cursing at badly designed APIs.

## Usage ##

To use the API, you must first obtain an application identifier and key, which is free and easy but involves a CAPTCHA. The app-id may be safely shared, but the api-key is private and losing it may jeopardize the security of your application. Let's assume your app-id is 'TheAppIdent".

Your server will access the API at https://api.runorg.com/TheAppIdent/ using REST conventions. Both POST and PUT data must be an UTF-8 JSON string and have a content-type of application/json. For safety reasons, do not access the API from the client unless you have absolute confidence in the client. 

Any responses will have an UTF-8 JSON string as their body. 

## Authentication ##

You MUST authenticate every request by appending the SHA1-HMAC of that request. This requires that the request be placed in a canonical format, which is: 

  - The method (`GET`, `POST`, ...) followed by a space. 
  - The path segment of the URL, starting with `/`, in lowercase, followed by `\r\n`
  - The contents of the HTTP `Date:` header, in `'D, d M Y H:i:s \G\M\T'` format, followed by `\r\n` 
  - If the request includes a payload (`POST` and `PUT` requests do), the payload.

**Example 1**: a GET request and the corresponding canonical string

    curl https://api.runorg.com/TheAppIdent/user/38421668914?auth=<HMAC>

    GET /TheAppIdent/user/38421668914\r\n
    Mon, 19 Nov 2007 23:47:33 GMT\r\n

**Example 2**: a PUT request and the corresponding canonical string

    curl -X PUT https://api.runorg.com/TheAppIdent/user/38421668914/email?auth=<HMAC> \
      -d '{"value":"test@example.com"}'

    PUT /TheAppIdent/user/38421668914/email\r\n
    Mon, 19 Nov 2007 23:47:33 GMT\r\n
    {"value":"test@example.com"}

If the HMAC does not match the request, or if the date header is off by more than 10 minutes, the server responds with HTTP/1.1 400 BAd Request.

## Contexts ##

The structure of almost every API URL is as follows: 

    https://api.runorg.com/TheAppIdent/[context]/[resource]/[filter]

The context is a well-defined set of values that determine what the resource is and whether it is accessible. These values are, in that order: 

  - What organization the application is working on (`/in/<id>`). 
  - What user the application is working as (`/as/<id>`).
  - What user the application is working *on* (`/user/<id>`). 
  - What entity the application is working on (`/entity/<id>`).

**Example**: working on organization 42 as user 1337 to read the profile information of user 666, one would use the following:

    GET https://api.runorg.com/TheAppIdent/as/1337/in/42/user/666/profile

This examples would return the information that user 1337 can see about user 666 in organization 42. 

## Territory ##

Users and organizations must allow your application to perform operations before you can perform them. They do so by providing your application with *territories*.

A *territory* is a resource identifier path filled with wildcards, along with the allowed methods on that resource. If you perform a request and the path for that request matches at least one of your territories, then you are allowed to perform it.

**Example 1**: In order to see the profiles for all the users in organization 42, your application needs the following territory: 

    "/in/42/user/*/profile" : ["GET"]

Your application needs to list all territories that might be useful. This involves a different wildcard character `#` that may only appear as part of `/in/#/`, `/as/#/` and `/entity/#/` contexts. The meaning of `#` is &laquo;If you wish to use the application in this context, you must grant me this territory first.&raquo;

**Example 2**: In order to be granted the territory from the previous example, an application would have to provide the following territory request: 

    "/in/#/user/*/profile" : ["GET"]

This means that an **organization administrator** may install your application for the **entire organization**, and this will involve granting the application the ability to see all user profiles in that organization.

**Example 3**: if you only require access to be granted for individual entities, you may instead require: 

    "/in/#/entity/#/user/*/profile" : ["GET"]

This means that an **entity administrator** may install your application for the **entity**.

**Example 4**: the least level of access is acting as an user: 

    "/in/#/as/#/user/*/profile"

This means that **any user** may install your application. Of course, you can only see what they can see and do what they can do!

Similarly, `/as/*/` territories are even more restrictive, but you only need the assent of the user themselves. 

The list of current territory requests for your applications can be found at: 

    GET https://api.runorg.com/TheAppIdent/territory-requests

    {
      "optional": {
        "/in/#/user/*/profile/"      : ["GET","PUT"]
        "/in/#/as/#/user/*/profile/" : ["GET","PUT"]
      },
      "required": {
        "/in/#/entity/#/user/*/profile/" : ["GET"]
      }
    }

You can `PUT` a new list of territories to this URL. Our server will detect the changes and ask the users for any new permissions you might need.

To see what territories have been granted in a particular context, `GET` the territories resource in that context: 

    GET https://api.runorg.com/TheAppIdent/in/42/territories

    {
      "granted": {
        "/in/42/user/*/profile/" : ["GET"],
        "/in/42/as/1337/user/*/profile" : ["GET","PUT"]
      }
    }

Attempting to perform a request that is outside your territories will result in a `403` return code, along with a list of possible territories you might want to try. 

    PUT https://api.runorg.com/TheAppIdent/in/42/user/666/profile

    HTTP/1.1 403 Forbidden
    ...
    { 
      "error" : "territory", 
      "try"   : [ "/in/42/as/1337/user/666/profile" ]
    }

