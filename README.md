rowlevelsecurity_for_rails_devise_mysql
=======================================

If you’re building a web application with a database, you may want more than one user, which means adding row-level security, and authentication.  Rails helps with a cool gem called Devise.  Devise creates a User table and views to authenticate into the web app with a session, and then you can add authentication to each view to make sure you have a valid session to see that view.

That’s all fine and dandy but doesn’t solve my other problem, which is to restrict data in the database by user.  So I chose to use the Devise User model, in particular the id field, and altered by other database tables to include the id field ( I called it user_id ).  This all works when you then tweak the rest of the Rails app like the model calls to use the session user_id as part of each SQL query.   Let me take a stab at explaining the basic steps, which have some detail, but you’ll need to improvise for your case.

The process in my eyes is divided into Devise-specific steps, and row-level security steps.

Devise Steps

Adding Devise to your Rails Project

add devise gem to gem file

gem ‘devise’

bundle install



rails generate devise:install

( installs devise into rails app - be in the app directory root )

 

rails g devise:views

( Optional step: this copies views so we can customize and style )

rails generate devise User

( creates the user model )

bundle exec rake db:migrate

( creates the table - note you need privileges in the yml to change the database structure )

Remaining Devise-related Steps

Add a top div to application.html.erb checking if user is logged in or not, and showing login id or new user and login link

<div class=”top”>

<p class=”notice”><%= notice %></p>

<p class=”alert”><%= alert %></p>

<p>

<% if user_signed_in? %>

  Logged in as <strong><%= current_user.email %></strong>.

  <%= link_to ‘Edit profile’, edit_user_registration_path %> |

  <%= link_to “Logout”, destroy_user_session_path, method: :delete  %>

<% else %>

  <%= link_to “Sign up”, new_user_registration_path  %> |

  <%= link_to “Login”, new_user_session_path %>

<% end %>

</div>

created a ‘guest’ controller and view to show when not logged in - also updated routes.rb to make this the root - i.e., the first time someone goes to your site, you should take them to the guest controller, not the data! You don’t have a session yet.

updated guest/root controller, in particular, index action to go to logged in controller if user is logged in 

  if user_signed_in?

    redirect_to :controller=>’home’, :action=> ‘index’

  end

In every other data/loggedin controller, add the following to the top of the controller before the actions:  This will ensure no one can use the URL paths to land on a page without being authenticated.

before_filter :authenticate_user!

Row-level steps

Alter the tables where you want Row-level security “RLS” with user_id column.

mysql> ALTER TABLE mytable ADD COLUMN user_id VARCHAR(255) AFTER id;

Update the tables with a valid user_id from users table ( this assumes you already created a user with the register form. Or just use 1, since it appears Devise starts the first user id at 1 )

mysql> UPDATE mytable SET user_id = ( SELECT id FROM users WHERE email = ‘mememe@me.com’ );

Handle adding of user_id to table inserts

In data/loggedin controller CREATE action, update the user_id field in the parameter object. 

def create

params[:mytable][:user_id] = current_user.id

…

end

Handle selects / queries by passing user_id as a parameter

In a data/loggedin model, pass user_id into a new my query scope and then combine that new scope with other scopes ( making sure to pass user_id into the other scopes; there’s other ways to do this, but I love scopes ):

scope :my, -> (user_id) { where(“user_id = ?”, user_id) }

scope :mylovelyquery, -> (user_id) { my(user_id).where(:mylovelyselectioncriteria=>”A”).order(“mysortfield”)  }

In the data/logged controllers, change index action to get current_user.id and pass it to the ActiveRecord scopes you created.  This step is tedidious and could be done often - look for any controller indexes or any controller action doing a SELET - that need RLS!   This applies to Ajax calls too.

def index

@user_id = current_user.id

@mytables = Mytable.my(@user_id).order(mylovelysortfield DESC”)

end

Lessons Learned

Probably the one thing that hung me up the most was forgetting to get the current_user.id, before doing any retrieving the database.
