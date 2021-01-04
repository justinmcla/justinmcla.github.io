---
layout: post
title:      "JS Project - Workbench, and Dealing with JWTs"
date:       2021-01-04 19:05:45 +0000
permalink:  js_project_-_workbench_and_dealing_with_jwts
---


For my next project, I decided to tackle a relatively familiar domain, maintenance requests/work orders. The SPA (single-page application) would be simple. An end-user should be able to browse a list of facilities, submit a maintenance request for a facility, and receive a confirmation notice that it was created. From there, they should be able to check (i.e. read) the status of a work order they submitted, using the confirmation number. Additionally, a user should be able to submit some kind of information to populate a privately-managed facility (such as their apartment, office space, etc.) and submit a work order for that private facility.


Submitting public work orders was easy enough. The API endpoints could be openly available and no authentication would be needed upon starting the app. Private data, however, should only be made accessible and usable to authorized end-users. I decided to utilize JSON Web Tokens (JWTs) to handle each request to the server that dealt with private data. Even so, a number of problems arose that needed to be dealt with.

First, and easily handled enough, Rails does not have built-in functionality to encode and decode JWTs. This is easily solved by adding the `jwt` gem to the `Gemfile`, and following Chris Oliver's example on his (AMAZING) website, [Go Rails](http://www.gorails.com).

```ruby
# /Gemfile

# Use JWT for API authentication
gem 'jwt'

```

This mainly provides two simple methods:

```ruby

payload = { 'sub': 1 }
token   = JWT.encode payload 
#>> 'eyJhbGciOiJI . . .

data    = JWT.decode token
#>> [ { 'sub': 1 }, { 'alg': 'HS256', 'typ': 'JWT' } ]

```

These methods also take some optional arguments. The first optional argument allows you to set a secret key signature for the JWT. I created a wrapper model for JWT to default to using the secret key base. I also used this to set a default expiration time of 30 minutes for the JWTs, and to return only the payload after the JWT is decoded.

```ruby

class JsonWebToken
  SECRET_KEY = Rails.application.secrets.secret_key_base
	
	def self.encode payload
	  JWT.encode payload.merge(exp: 30.minutes.from_now.to_i), SECRET_KEY
	end
	
	def self.decode token
	  JWT.decode(token, SECRET_KEY).first
	end

```

From there, I wrote a handful of controller methods to handle authentication in my API controllers, as well as rescue from bad tokens by rendering error messages.

```ruby

# app/controllers/api/v1/api_controller.rb
. . .
def authenticate_token
  payload = JsonWebToken.decode(auth_token)
  @code   = payload['code']
rescue JWT::ExpiredSignature
  render_expired and return
rescue JWT::DecodeError
  render_expired and return
end
def auth_token
  @auth_token ||= request.headers.fetch('Authorization, '').split(' ').last
end
```

I set up my various API resource controllers to inherit from API controller, and so now all I needed to do server-side was call `authenticate_token` before any protected action.

```ruby

# app/controllers/api/v1/work_orders_controller.rb
class Api::V1::WorkOrdersController < Api::V1::ApiController
  before_action :authenticate_token, only: %i[show update]
  def show
    work_order ? render_work_order(work_order) : render_no_record
  end
  . . . 
  private
  def work_order
    @work_order ||= WorkOrder.find_by confirmation: @code
  end
. . .

```

With server-side set up, handling the JWT client-side proved to be a little confounding. A quick Google search will yield all sorts of blogs and videos and other opinions on where to store the JWT client-side. There was a pretty clear consensus that storing the JWT in local storage was not a great idea due to XSS concerns. Some folks suggested storing it in a cookie, but that also could yield some XSRF concerns. After a bit of searching, Ben Awad suggested storing it in memory, which I eventually settled on. The main downfall is that memory is wiped upon a page refresh, but because of that, it is the most preferable option in terms of security. I set my tokens to expire in 30 minutes regardless, and the data being rendered shouldn’t persist beyond a refresh anyway. It’s a SPA for a reason. I comfortably went ahead and built a JS class to handle getting and setting JWT tokens from the server.

```javascript
# public/js/jwt.js
class JsonWebToken {
  static get() {
    return this.token
  }
  static set(token) {
    this.token = token
  }
  static clear() {
    this.token = null
  }
  static requestToken(requestType, code, password) {
    this.clear()
    return fetch(API.auth(), {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      }, body: JSON.stringify({
        'request_type': requestType,
        'code': code,
        'password': password
      })
    }).then(res => res.json())
    .then(json => {
      if (json.status == 200) {
        JsonWebToken.set(json.token)
        return json
      } else {
        return Promise.reject(json.errors)
      }
    }).catch(error => console.error(error))
}
```

`JsonWebToken.requestToken` sends a request to the API for a JWT. The server verifies the parameters that were sent, and either returns a JWT or an error message. The returned JWT is set in memory and can be called later on to be sent along with requests involving private data. Once the page is refreshed, everything is cleared out of memory and no data is persisted, as intended. The user will have to reauthenticate themselves by requesting a new token, ensuring security.
