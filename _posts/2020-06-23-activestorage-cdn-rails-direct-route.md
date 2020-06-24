---
layout: default
title: "Serving ActiveStorage uploads through a CDN with Rails direct routes"
tags: ruby
comments: true
cover: /assets/images/cyprinus-aspius.jpg
---

## {{ page.title }}

ActiveStorage makes it really easy to upload files from Rails to an S3 bucket or an S3-compatible service, like DigitalOcean Spaces. Refer to the [official documentation](https://edgeguides.rubyonrails.org/active_storage_overview.html) if you'd like to know more about setting up ActiveStorage.

If your uploads are meant to be public and you were thinking of serving them directly through the CDN sitting in front of your S3 bucket, you'll soon notice a problem: ActiveStorage URLs are built to always go through your Rails app, mainly through [ActiveStorage::BlobsController](https://github.com/rails/rails/blob/bc9fb9cf8b5dbe8ecf399ffd5d48d84bdb96a9db/activestorage/app/controllers/active_storage/blobs_controller.rb#L10-L13). This controller is responsible for setting the cache headers and redirecting to the bucket URL. Your Rails app will be the first point of contact even if it's just to retrieve the bucket URL. On top of that, there's no place to specify a CDN host to replace the bucket host.

Fortunately, there is an easy way to go around this problem. In order to translate stored files into URLs, Rails provides the URL helper [rails_blob_url](https://edgeguides.rubyonrails.org/active_storage_overview.html#linking-to-files), which basically resolves to this `ActiveStorage::BlobsController`. We'd like to introduce a new helper that points directly to our CDN host.

Though there are different ways of solving this problem, I found using Rails direct routes an elegant solution. [Rails direct routes](https://guides.rubyonrails.org/routing.html#direct-routes) provide a way to create URL helpers directly from your `config/routes.rb`:

```ruby
# config/routes.rb

direct :rails_public_blob do |blob|
  File.join("https://cdn.example.com", blob.key)
end
```

You can call this route the same way you'd call the original Rails URL helper:

```ruby
class User
  has_one_attached :profile_picture
end

rails_public_blob_url(User.first.profile_picture)
# => https://cdn.example.com/j8rte71tp8xpq5afr3uqxlcqtkzn

# You can also use this outside views
Rails.application.routes.url_helpers.rails_public_blob_url(User.first.profile_picture)
```

Finally, let's refactor our route a bit:

```ruby
# config/routes.rb

direct :rails_public_blob do |blob|
  # Preserve the behaviour of `rails_blob_url` inside these environments
  # where S3 or the CDN might not be configured
  if Rails.env.development? || Rails.env.test?
    route_for(:rails_blob, blob)
  else
    # Use an environment variable instead of hard-coding the CDN host
    File.join(ENV.fetch("CDN_HOST"), blob.key)
  end
end
```

You can use this new URL helper whenever your ActiveStorage files are to be served directly through a CDN.

To conclude, Rails 6.1 will allow defining multiple storage services for the same environment, which means you'll be able to use both public and private buckets from your code. This makes using public buckets and CDNs an even more viable option than before. See this [PR](https://github.com/rails/rails/pull/34935) for more details.

