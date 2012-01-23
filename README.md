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

**Example 1**: if you send a GET request to:

    curl https://api.runorg.com/TheAppIdent/user/38421668914?auth=<HMAC>

Then `<hmac>` will be the SHA1-HMAC of your app-key and this string :

    GET /TheAppIdent/user/38421668914\r\n
    Mon, 19 Nov 2007 23:47:33 GMT\r\n

**Example 2**: if you send a PUT request to: 

    curl -X PUT https://api.runorg.com/TheAppIdent/user/38421668914/email?auth=<HMAC> \
      -d '{"value":"test@example.com"}'

Then `<hmac>` will be the SHA1-HMAC of your app-key and this string :
 
    PUT /TheAppIdent/user/38421668914/email\r\n
    Mon, 19 Nov 2007 23:47:33 GMT\r\n
    {"value":"test@example.com"}

If the HMAC does not match the request, the server responds with : 

    HTTP/1.1 400 Bad Request 
    ...
    { "error":"auth", "hmac":"...", "raw":"..." }

The `hmac` field contains the received HMAC, and `raw` contains the request placed in canonical format that the server used to compute the HMAC, so that you may check it against your own canonical format. 

If the HMAC matches the request, but the date header is off by more than 10 minutes, the server responds with : 

    HTTP/1.1 400 Bad Request
    ...
    { "error":"date", "date":"...", "offset": ... }

The `date` field echoes the contents of the `Date:` HTTP header on the request, and `offset` field contains the offset between that time and the time when it was received by the server, in seconds (positive values means the request arrived too long after the expected time).

If the HMAC matches the request and the date header is valid, the request is considered to come from your server, and processing will commence. 

## Contexts ##

A given resource may be accessed in several contexts, which may determine whether an operation is possible or how it is performed. The context appears in the resource identifier. Context information includes: 

  - What organization the application is working on (`/in/<id>`). 
  - What user the application is working as (`/as/<id>`). 
  - What entity the application is working on (`/on/<id>`).

**Example 1**: working on organization 42 as user 1337 to read the profile information of user 666, one would use the following:

    GET https://api.runorg.com/TheAppIdent/as/1337/in/42/user/666/profile

The relative order of context identifiers is irrelevant as long as they appear in first position in the path. 

**Example 2**: this is equivalent to the example above.

    GET https://api.runorg.com/TheAppIdent/in/42/as/1337/user/666/profile

These examples would return the information that user 1337 can see about user 666 in organization 42. 

## Territory ##

Users and organizations must allow your application to perform operations before you can perform them. A *territory* is a description of a set of possible operations. 

**Example**: If your application must be able to see user profile information for all users of an organization, the corresponding territory request would be: 

    GET /in/*/user/*/profile

This request would have to be accepted by an organization administrator. 

If your application needs to access user profiles, but only for users that are members of a certain entity, it would be: 

    GET /on/*/user/*/profile

This is a more restrictive territory, but it is safer for the user (as they can grant you access on a per-entity basis instead of opening up their entire member directory to you), and you only need the assent of an entity manager (as opposed to the organization administrators themselves). 

Similarly, `/as/*/` territories are even more restrictive, but you only need the assent of the user themselves. 

The list of current territory requests for your applications can be found at: 

    GET https://api.runorg.com/TheAppIdent/territories

    {
      "optional": {
        "/in/*/user/*/profile/" : ["GET","PUT"]
        "/in/*/as/*/user/*/profile/" : ["GET","PUT"]
      },
      "required": {
        "/in/*/on/*/user/*/profile/" : ["GET"]
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

