# APIs with Devise

## Preface
Authentication with Rails can be a touchy subject. But I'm not going to get into
the debate of whether or not you should roll your own, use something like
Devise, throw Devise away and just use Warden, or follow any other of
available methods for building authentication into your application.

This last week I fielded more questions in IRC and at a
[local beginner rails group](http://rails.mn) about handling authentication
via an API than I did about setting up an actual authentication framework,
so that's what I'm going to write about.

## Handling API Authentication with Devise

You may have already seen
[Ryan Bates' Railscast on Securing an API](
htte://railscasts.com/episodes/352-securing-an-api), which discusses token
authentication over an HTTP header, but are wondering how to integrate that into
your existing authentication paradigm. Since I end up using Devise for
authentication on a majority of my smaller projects, that's what I'll assume
you're using for the purposes of this tutorial.

Even though there are other guides to
[Token Authentication with Devise](
https://github.com/plataformatec/devise/wiki/How-To:-Simple-Token-Authentication-Example)
out there, they're a bit hidden in documentation, and most aren't exactly
up-to-date.

The crux of the idea is this: 

1. We want to create a minimal authentication system whereby we receive a
username, email, or other user identifier, and a password; after authenticating
the user, we'll return an authentication token for use in subsequent 
requests[1].
2. We should, in all subsequent requests, allow a user to be authenticated via
their authentication token.
3. We should be reasonably careful in handling this authentication information.  

To satisfy our criteria, I tend to use something similar as to what's suggested
on [this gist by @josevalim on github](
https://gist.github.com/josevalim/fb706b1e933ef01e4fb6) combined with a few
other ideas[2], with a couple of tweaks:

```ruby
  # ApplicationController.rb
  protect_from_forgery with: :exception
  # In Rails 3.x:
  # skip_before_filter :verify_authenticity_token,
                :if => Proc.new { |c| c.request.format == 'application/json' }
  # In Rails 4.x:
  protect_from_forgery with: :null_session,
                :if => Proc.new { |c| c.request.format == 'application/json' }

  before_filter :authenticate_user_from_token!
  before_filter :authenticate_user!

  ... 

  def authenticate_user_from_token!
    user_email = request.headers["X-API-EMAIL"].presence
    user_auth_token = request.headers["X-API-TOKEN"].presence
    user = user_email && User.find_by_email(user_email)

    # Commentary:
    #
    # Notice how we use Devise.secure_compare to compare the token
    # in the database with the token given in the params, mitigating
    # timing attacks.
    if user && Devise.secure_compare(user.authentication_token, user_auth_token)
      sign_in(user, store: false)
    end
    # End Commentary
  end
  ...
```

And of course, the accompanying methods for our User.rb:

```ruby
  # After creating the authentication_token text column on the users table...
  ...
  before_save :ensure_authentication_token!

  def generate_secure_token_string
    SecureRandom.urlsafe_base64(25).tr('lIO0', 'sxyz')
  end

  # Commentary:
  #
  # Sarbanes-Oxley Compliance:
  # http://en.wikipedia.org/wiki/Sarbanes%E2%80%93Oxley_Act
  def password_complexity
    if password.present? and not
        password.match(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[\W]).+/)
      errors.add :password, <<-EOF
        must include at least one of each: lowercase letter, uppercase letter,
        numeric digit, special character."
      EOF
    end
  end
  # End Commentary

  def password_presence
    password.present? && password_confirmation.present?
  end

  def password_match
    password == password_confirmation
  end

  def ensure_authentication_token!
    if authentication_token.blank?
      self.authentication_token = generate_authentication_token
    end
  end

  def generate_authentication_token
    loop do
      token = generate_secure_token_string
      break token unless User.where(authentication_token: token).first
    end
  end

  def reset_authentication_token!
    self.authentication_token = generate_authentication_token
  end
  ...
```

What we're doing in the ApplicationController is protecting all requests to our
 application by raising an exception if the authenticity token is invalid;
 however, we're making sure that the session for all JSON requests is treated as
 an empty session. You can read up on all of that at the
 [ActionController::RquestForgeryProtection](
 http://api.rubyonrails.org/classes/ActionController/RequestForgeryProtection/ClassMethods.html#method-i-protect_from_forgery)
 doc site.

In this fashion, we simply have the CSRF session return null, and we neglect to
 store it while authenticating via authentication token. This mitigates any
 session based attacks on your application's JSON endpoints.

A couple of tweaks that I've made compared to the gist linked above are related
 to the way authentication information is sent over to the application:

```ruby
...
user_email = request.headers["X-API-EMAIL"].presence
user_auth_token = request.headers["X-API-TOKEN"].presence
...
```

Rather than sending the data over parameters, we're expecting the client
application to send it via two headers: "X-API-EMAIL" and "X-API-TOKEN";
this cleans up the endpoint URIs.

## SessionsController.rb
What's a SessionsController? Well, if you're using Devise for authentication,
you might not realise that you have one.  It handles (simply put) the creating
and destroying of logged in user sessions for your application.

We'll need to override this slightly to handle user authentication via the API.
The [Ember Rails4 Starter Kit](
https://github.com/amaanr/ember-rails4-starter-kit/blob/master/app/controllers/sessions_controller.rb)
had a good starting place, and I believe that's where I got the initial template
 for this:

```ruby
  class SessionsController < Devise::SessionsController
    skip_before_filter :authenticate_user!, :only => [:create, :new]
    skip_authorization_check only: [:create, :failure, :show_current_user,
                                    :options, :new]
    respond_to :json

    def new
      self.resource = resource_class.new(sign_in_params)
      clean_up_passwords(resource)
      respond_with(resource, serialize_options(resource))
    end

    def create

      respond_to do |format|
        format.html {
          super
        }
        format.json {

          resource = resource_from_credentials
          #build_resource
          return invalid_login_attempt unless resource

          if resource.valid_password?(params[:password])
            render json: {
                                user: {
                                    email: resource.email,
                                    auth_token: resource.authentication_token
                                }
                            }, success: true, status: :created
          else
            invalid_login_attempt
          end
        }
      end
    end

    def destroy
      respond_to do |format|
        format.html {
          super
        }
        format.json {
          user = User.
                    find_by_authentication_token(request.headers['X-API-TOKEN'])
          if user
            user.reset_authentication_token!
            render json: { :message => 'Session deleted.' },
                    :success => true,
                    :status => 204
          else
            render json: { :message => 'Invalid token.' },
                    :status => 404
          end
        }
      end
    end

    protected
    def invalid_login_attempt
      warden.custom_failure!
      render json: {
                        success: false,
                        message: 'Error with your login or password'
            }, status: 401
    end

    def resource_from_credentials
      data = { email: params[:email] }
      if res = resource_class.find_for_database_authentication(data)
        if res.valid_password?(params[:password])
          res
        end
      end
    end
  end
```

Without delving into a conversation about the inner workings of
[Devise](https://github.com/plataformatec/devise) and
[Warden](https://github.com/hassox/warden), what we're doing here is overriding
 Devise's SessionsController with our own. Ours sends html requests back up to 
 Devise's SessionsController, but handles JSON requests a bit differently: 

1. On **#destroy**, we authenticate via the 'X-API-TOKEN' header sent by the 
client, and reset the users' authentication token.
2. On **#create**, we return the user's email and authentication token. These 
are all that are required for subsequent API requests - until we either reset 
the token (typically via TTL) or the user resets the token via #destroy.  

## Enabling CanCan Authorization
Oh, and as an added bonus for all of you CanCan users out there still trying to 
wrangle it to work with Rails 4/Strong Parameters and Devise, here's a magical 
snippet to toss into your ApplicationController.rb:

```ruby
  # Commentary:
  #
  # Fix for CanCan/Strong Parameters (expecting format ex: /api/v1/resource)
  after_filter do
    resource = controller_path
                .singularize
                .gsub(/api\/v[\d]+\//i, '')
                .gsub('/', '_')
    method = "#{resource}_params"
    params[resource] &&= send(method) if respond_to?(method, true)
  end
  # End Commentary

  # Commentary:
  #
  # Enable CanCan authorization by default
  check_authorization unless: :devise_controller? 
  # End Commentary
```

Rather than delving into CanCan's inner workings here, I'll just direct you to
 these two issues: 

- [How does CanCan handle mass-assignment? (Issue #571)](
https://github.com/ryanb/cancan/issues/571)
- [Make check_authorization conditional with block or options (Issue #284)](
https://github.com/ryanb/cancan/issues/284)

[1] Setting and enforcing a [TTL](http://en.wikipedia.org/wiki/Time_to_live) 
is an exercise left to the reader.

[2] There are a number of [gists](http://gist.github.com) that re-use some of 
the same code snippets I'm pasting below, but tracing back the original source 
can be tough. 

