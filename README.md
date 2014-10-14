# README
## Introduction
This is a basic rails app that demonstrates how to upload files using the [aws-sdk gem](https://github.com/aws/aws-sdk-ruby). This app utilizes [Amazon's Simple Storage Service (S3)](http://aws.amazon.com/s3/). Amazon S3 is an excellent service for storing uploads.

## Rails
- version 4.2

## Pre-requisites
### Create your Amazon Web Services (AWS) Account
Before you get started with this app you will need to create an Amazon Web Services (AWS) account (if you don't already have one). AWS has a [free tier](http://aws.amazon.com/free/) that you can utilize which will suffice for this app. After creating your AWS account, log in.
After creating your account and logging in go to your AWS Management console. It should look similar to this (as of October 2014):
<img src='https://dl.dropbox.com/s/28q9x7zztfpkcor/Screenshot%202014-10-13%2013.36.05.png?dl=0'>
Under 'Storage & Content Delivery' click on 'S3'.

### Create a bucket
After clicking on 'S3' under your management console, you will then be taken to a screen that looks similar to this:
<img src='https://dl.dropbox.com/s/xexkkmq6dd0jz2u/Screenshot%202014-10-13%2013.38.05.png?dl=0'>
This is your S3 dashbaord. You will need to create a bucket. All files that you upload to S3 are stored in buckets. I usually create one bucket per rails application that I build. Create your first bucket by clicking on the 'Create Bucket' button. I'll name mine 'my-first-bucket-tutorial' (bucket names must be unique across regions so pick a bucket name you like). Select the region that is closest to you. 
<img src='https://dl.dropbox.com/s/4sogo7vjpkw3nmf/Screenshot%202014-10-13%2013.44.56.png?dl=0'>
After you create your bucket it should show up in the list of buckets. Go ahead and click on that bucket. Nothing's there? Good. It should be empty because you haven't uploaded anything yet! Let's build a rails application that will upload files to your bucket.

## Getting Started
First create your rails app:
```
$ rails new upload_app
```
Next add the aws-sdk gem to your `Gemfile`:
```ruby
gem `aws-sdk`
````
Don't forget to run bundle install:
```
$ bundle install
```

### Uploads Controller & Routes
Great, we're off to a good start. What do we need next? We need some routes. Add this line to your `config/routes.rb` file.
```ruby
resources :uploads
```
Run rake routes in your terminal. We've got some new routes now. Lets an add upload controller for these routes:
```
$ rails g controller Uploads new create index
```
We now have routes, and a controller to handle requests to those routes
- `app/controllers/uploads_controller.rb`

We also have some view files to handle those routes (create these if you don't already have them):
- `app/views/uploads/new.html.erb`
- `app/views/uploads/create.html.erb`
- `app/views/uploads/index.html.erb`

### The upload form
We're going to need a form to upload our files. Open up your `app/views/uploads/new.html.erb` file and type this in:
```html
<h1>Upload a file</h1>

<%= form_tag uploads_path, enctype: 'multipart/form-data' do %>
	<%= file_field_tag :file  %>
	<%= submit_tag 'Upload file' %>
<% end %>
```
Notice the use of `form_tag`. We aren't going to be dealing with an object so we don't need to use `form_for` instead we use `form_tag`. We're going to use `uploads_path` for the action. The `POST /uploads` request maps to the `uploads#create` action which you can find by running `$ rake routes` in your terminal. For the sake of navigation, let's route our root_path to `/uploads/new`.
Add this to your `routes.rb` file:
```ruby
root 'uploads#new'
```

### AWS Credentials
You're going to need credentials to communicate with AWS. Learn more about how to get your AWS security credentials [here](http://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html)
After you have your security credentials be sure to set them in your environment.
```
$ export AWS_ACCESS_KEY_ID=INSERT_YOUR_ACCESS_KEY_ID_HERE
```
```
$ export AWS_SECRET_KEY=INSERT_YOUR_SECRET_KEY_HERE
```
**Keep your security credentials secure. Never expose your AWS_SECRET_KEY**

### Put your bucket in the ENV
It can help to put the bucket your working with in the ENV as well. Substitute in the name of your bucket you created earlier.
```
$ export S3_BUCKET=INSERT_YOUR_BUCKET_NAME
```

### Configure your application with AWS
Before we begin using the AWS::S3 API we need to configure our app with the AWS sdk. We do this by using the security credentials we set up in the ENV earlier. 
Add an initializer file called `aws.rb` in your `config/initializers/` directory. Put this code inside your `aws.rb` initializer file:
```ruby
AWS.config(
  :access_key_id => ENV['AWS_ACCESS_KEY_ID'],
  :secret_access_key => ENV['AWS_SECRET_ACCESS_KEY']
)

S3_BUCKET =  AWS::S3.new.buckets[ENV['S3_BUCKET']]
```
Now your application can talk to AWS.

### Actually upload the file
Type the following code into your controller. We'll go through this line by line after.
```ruby
class UploadsController < ApplicationController
  def new
  end

  def create
    # Make an object in your bucket for your upload
    obj = S3_BUCKET.objects[params[:file].original_filename]
    
    # Upload the file
    obj.write(
      file: params[:file],
      acl: :public_read
    )
    
    # Create an object for the upload
    @upload = Upload.new(
			url: obj.public_url,
			name: obj.key
		)
    
    # Save the upload
    if @upload.save
      redirect_to uploads_path, success: 'File successfully uploaded'
    else
      flash.now[:notice] = 'There was an error'
      render :new
    end
  end
  
  def index
  end
end
```

### Making an object in your bucket
Lets take a look at the first line:
```ruby
# Make an object in your bucket for your upload
obj = S3_BUCKET.objects[params[:file].original_filename]
```
We can create objects in our S3 bucket using the following syntax:
```ruby
bucket.objects['insert-your-key'];
````
We've already defined the bucket in the `aws.rb` initializer file we created earlier. So we can just use `S3_BUCKET` to reference what bucket we are trying to create an object in.
Next we call the `objects['insert-your-key']` method to create the object in our bucket. The key we insert is the name it gets in the bucket.
Our code looks a little different. Our key looks like this: 
```ruby
params[:file].original_filename
```
This is because we want our object to have the name of the file we uploaded. File uploads get stored in the params hash so we can get the name of the uploaded file using `params[:file].original_filename`.
Using this line we create the object in S3 and save the object for further use.
### Writing the file into your object (uploading the file)
```ruby
# Upload the file
obj.write(
  file: params[:file],
  acl: :public_read
)
```
Using the object we created in the first line we can use the `write` method to actually write the file to S3. We pass in two params:
- file: params[:file]
- acl: :public_read

The `:file` param points to the actual file we need to upload.
The `acl: :public_read` tells S3 to make this file publicly accessible.

### Create a new upload object
We're going to need some way to keep track of what uploads got uploaded to S3 and where we can find them. We can do this by saving an upload object to the database that contains the URL and name of the S3 object.
```ruby
# Create an object for the upload
@upload = Upload.new(
	url: obj.public_url, 
	name: obj.key
)
```
If the object is successfully uploaded we can call `.public_url` on it to get the URL to access it
You might be wondering, wait... we didn't create an Upload model yet to save to our database? You're right. We'll get to that.

### Save the upload object in our database
Save and redirect to all of our uploads or simply render the new form.
```ruby
# Save the upload
if @upload.save
  redirect_to uploads_path, success: 'File successfully uploaded'
else
  flash.now[:notice] = 'There was an error'
  render :new
end
```

## Create the Upload model
Let's create an upload model to store information about our uploaded files:
```
$ rails g model Upload url name
```
Migrate the database:
```
$ rake db:migrate
```
Great we're pretty close to done.
### Upload index
Now lets create a view that shows all of the uploaded files we've saved.
Put this view into your `app/views/uploads/index.html.erb`:
```html
<h1>All uploads</h1>
<% @uploads.each do |upload| %>
	<div>
		<%= link_to upload.name, upload.url %>
	</div>
<% end %>
```
## Done
Start up the rails server and start uplooading files to see your work in action.
