# Creeper

We want to make an application that has the ability to take an uploaded image and plot where it was taken on a map. To do so, we're going to need to use a few gems:

1. `pry-rails`: the worlds most badass gem there ever was. Pry is a general debugging tool that allows you to set breakpoints (with something like `binding.pry`), lookup method definitions (with prefacing the method with `$`), and lots of great syntax highlighting.
2. `better_errors` and `binding_of_caller`: is an amazing way to transform the standard rails error messages with something that has a REPL that allows you to see what the code is like at the moment of failure.
3. `paperclip`: a very popular tool to allow people to upload files to your applications and store them with a model
4. `geocoder`: a location management gem that allows us to do lookups of locations by latitude and longitude.
5. `mini_exiftool`: as the NSA knows, all images have something called metadata associated with them. Modern cell phones (which a large population of today's images are captured with) store geolocation data inside of the metadata. We can extract that data with this gem.
6. `dotenv-rails`: dotenv allows us to hide certain credentials in a file that's not checked into version control.


### Installing dependencies

Unfortunately `miniexiftool` requires a dependency on your computer called `exiftool`. We first have to install it:

```bash
brew install exiftool
```

### Creating the application

```bash
rails new creeper
cd creeper
rails g scaffold image title:string body:text
rake db:migrate
```

### Adding our gems

At this point, we'll want to add the gems that we know that we'll be working with. At the end of your `Gemfile`, paste the following:

```ruby
group :development do
  gem 'pry-rails'
  gem 'better_errors'
  gem 'binding_of_caller'
end

gem 'paperclip'
gem 'geocoder'
gem 'mini_exiftool'
```

And run `bundle` to install them.

### Configuring paperclip

In order to properly use paperclip, we need to tell our model how to interact with the gem.

To do so, add the following to `app/models/image.rb`:

```ruby
has_attached_file :image
validates_attachment_content_type :image, :content_type => /\Aimage\/.*\Z/
```

And create a migration: `rails g migration AddImageToImages`. And add the following to the newly created migration in `db/migrate`:

```ruby
def self.up
  add_attachment :images, :image
end

def self.down
  remove_attachment :images, :image
end
```

Then run `rake db:migrate`.

Great. Now the model is set up, but we need to make sure that our views have a way of uploading images. Let's update `app/views/images/_form.html.erb` and add the following immediately before the submit button:

```erb
<div class="field">
  <%= f.label :image %><br>
  <%= f.file_field :image %>
</div>
```

Also, let's make sure that strong params allows us to add an image. In `app/controllers/images_controller.rb`, let's add `:image` to the `.permit()`:

```ruby
params.require(:image).permit(:title, :body, :image)
```

Sweet! We can now add images. The last step is to make sure that we can see the images once we've added them. In `app/views/images/show.html.erb`, let's add a way to view our images before the `<%= link_to 'Edit' %>`:

```erb
<%= image_tag @image.image.url, width: 600 %><br>
```

### Configuring geocoder

Geocoder requires that we have a latitude and longitude on our model. So let's add that:

```bash
rails g migration AddLatitudeAndLongitudeToImage latitude:float longitude:float
rake db:migrate
```

And then we have to make sure that we tell geocoder how it can figure out what the latitude and longitude of our model is. In `app/models/image.rb`:

```ruby
reverse_geocoded_by :latitude, :longitude
after_validation :geocode
```

### Making creeper creepier

Awesome. Now we have a way to upload images. The next step is to make sure that we're copying the exif data to the model when paperclip saves an image. Luckily, paperclip gives us a method called `after_post_process` that we can call in our model that will direct paperclip to run the method passed as an argument right after it finishes the post processing of the image. Let's take advantage of that in `app/models/image.rb`:

```ruby
after_post_process :copy_exif_data
```

Once we add that, let's add the method `copy_exif_data` as well as a helper method that will allow us to parse latitude and longitudes from the exif data:

```ruby
private

def copy_exif_data
  exif_data = MiniExiftool.new(image.queued_for_write[:original].path)
  self.latitude = parse_latlong(exif_data['gpslatitude'])
  self.longitude = parse_latlong(exif_data['gpslongitude'])
end

def parse_latlong(latlong)
  latlong.scan(/(.*) deg (.*)' (.*)" (.*)/).map do |d,m,s,r|
    calc = d.to_f + m.to_f/60 + s.to_f/3600
    if ['S', 'W'].include? r
      -calc
    else
      calc
    end
  end.last
end
```

### Google Maps

Sweet, so we have a creepy application set up on the backend, let's show the user where the image was taken on the frontend. We're going to be using an iframe to embed a google map on the `Image#show` page. In order to do that, we have to get an API key from Google. We can get one, here: [https://code.google.com/apis/console](https://code.google.com/apis/console).

This will allow us to query google maps, but we want to make sure that if this is an open source repository, we're never adding that API key to version control, since if it got into the wrong hands, our google maps functionality could be disabled by Google.

Enter `dotenv-rails`. Dotenv is a way for us to store sensitive information in a file (`.env`) and not have it checked into version control.

First, let's add `dotenv-rails` to our Gemfile. We _have_ to make sure that this is done at the top of the `Gemfile`, right after `source`:

```ruby
gem 'dotenv-rails', groups: [:development, :test]
```

And then we add our API key to `.env`:

```
GOOGLE_MAPS_API_KEY=AIzaSyAfSI6RpkFNMbrnUDlIV4MvAbaRGiU-2k
```

This will now allow us to access the key as so anywhere in our Rails code: `ENV['GOOGLE_MAPS_API_KEY']`. Which is perfect, becasue we'll need that in our view to display the map. Let's edit `app/views/images/show.html.erb` and put the following code after the `image_tag`:

```erb
<iframe
    width="600"
    height="450"
    frameborder="0" style="border:0"
    src="https://www.google.com/maps/embed/v1/place?key=<%= ENV['GOOGLE_MAPS_API_KEY'] %>
    &q=<%= "#{@image.latitude},#{@image.longitude}" %>">
</iframe>

```

Now we should be displaying the location of where our image was taken.

### What else?

We can do `Image.last.nearbys(20)`
