---
layout: single
author_profile: false
title: "Using Active Storage with Active Model Serializers"
date:   2018-07-60 21:17:08 -0500
categories: [tech]
---

Attachment is the root of all suffering. If you’re a web developer, attachments are the root of all suffering.

[Paperclip](https://github.com/thoughtbot/paperclip){:target="_blank"}, a longtime go-to gem for managing file attachments, has been deprecated in favor of [Active Storage](https://guides.rubyonrails.org/active_storage_overview.html){:target="_blank"}, which is now officially part of Ruby on Rails as of version 5.2. So for a recent greenfield app, I took the dive into Active Storage.

Switching out dependable gems for new ones can be anxiety-inducing, but that’s web developer life. While Active Storage ended up being a breath of fresh air for many reasons, it came with a few of it’s own gotchas, particularly when dealing with querying data.

### Active Storage and N+1 queries

Let’s say a user has one avatar. While Paperclip would add a few columns to your users table to store the avatar’s file data, file type, etc., Active Storage will create new tables that are solely responsible for keeping track of asset data and associating this data to your records.

These tables are called [blobs](https://github.com/rails/rails/blob/master/activestorage/app/models/active_storage/blob.rb){:target="_blank"} and [attachments](https://github.com/rails/rails/blob/master/activestorage/app/models/active_storage/attachment.rb){:target="_blank"}. Blobs represent the actual metadata of the files, whereas Attachments are what join blobs (file data) to your application’s records (users). For every file you upload, a blob is created and associated to your record through an attachment.

These relationships are created when you attach a file through Active Storage using the `has_one_attached` shorthand:

```ruby
class User < ApplicationRecord
  has_one_attached :avatar
```

It’s important to understand these relationships between blobs and attachments when dealing with record queries that may have attachments.

For `has_one_attached` relationships, you can avoid N+1 queries by adding the `with_attached_X` method that Active Storage generates for you based on your attachment name:

```ruby
User.with_attached_avatar.where(active: true)
```

This is basically saying:

```ruby
User.includes(avatar_attachment: :blob).where(active: true)
```

I had to take special care of these relationships when using Active Model Serializers. Normally, when dealing with child records in a serializer, you can define a relationship using `has_one` or `has_many`:

```ruby
class GameSerializer < ActiveModel::Serializer

  attributes :title

  has_many :users
```

If my User Serializer is using the avatar in any way, this would create an N+1 scenario:

```ruby
class UserSerializer < ActiveModel::Serializer
  attributes :avatar_url

  def avatar_url
    ...object.avatar_url...
```

In order to avoid N+1 queries here, I had to ditch the built-in has_many and create a custom users method for the Game Serializer:

```ruby
class GameSerializer < ActiveModel::Serializer
  attributes :title, :users
  def users
    User.with_attached_avatar.where(game_id: object.id)
```

### Finding the right asset URL for serialization

When passing image urls through serializers, I had to search through some [Active Storage Github](https://github.com/rails/rails/issues/32500#issuecomment-380004250){:target="_blank"} issues and [documentation](https://edgeguides.rubyonrails.org/active_storage_overview.html#linking-to-files){:target="_blank"} to find the right url to use.

Creating images of smaller sizes is common for real-life use of your assets. Active Storage calls these [variants](https://edgeguides.rubyonrails.org/active_storage_overview.html#transforming-images){:target="_blank"}. You’ll most likely use variants rather than the original asset in serializers.

If you want a url of the asset variant, include the `Rails.application.routes.url_helpers` module in your serializer, then use `rails_representation_url` in your url method:

```ruby
class UserSerializer < ActiveModel::Serializer
  include Rails.application.routes.url_helpers
  attributes :avatar_url
  def avatar_url
    variant = object.avatar.variant(resize: "100x100")
    return rails_representation_url(variant, only_path: true)
  end
```

If you want to use the original asset, use `rails_blob_path` instead:

```ruby
return rails_blob_path(object.avatar, only_path: true)
```

So there’s a little extra work when serialization is concerned, but overall I really enjoyed working with Active Storage and appreciate what a great thing this is for the Rails community.
