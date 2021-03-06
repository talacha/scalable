# Scalable Path Migration

This is a sample module developed during a pair programming excercise.

The test instructions were the following:

## Before Starting

- You have 1 hour to complete the test
- We will take the last 5 minutes to review the result
- You can use any tools you like (Google, Stack Overflow, documentation, etc.)
- You can use additional libraries and reference existing code that you may have
- It’s okay to be silent during the test or feel free to talk - whatever you’re comfortable with

## Task
- Build a Drupal page that displays a list of posts and the author-name
- The page should list each post in a list (`<li>`) with the user name next to it: `<li><a href="/post/1">My post</a> (John Doe)</li>`
- The data should be pulled from this posts resource and using this users resource to get the user name
- Add a 1-hour cache to the page (programatically)
- Create a second page/template that shows the content of the post. No need to style this page.

## Approach
There are many ways to skin a cat. In the interest of time, one could just follow
the instructions and create a page callback handler, map it out to a route say `/posts`
and write code to fetch the JSON data using Guzzle (http_client). Likewise one
would need another route, say `/posts/{{ id }}` and write logic to only find
that post in the JSON file.

This is a quick excercise for a senior Drupal developer, however I dared to
support my approach with a real world scenario and a list of pros and cons.

So in summary, I took a slightly different approach, instead of building the
above I used the `migrate` module to fetch, process and import the data from the
JSON sources.

Then I created the content types that I thought I needed with bare minimum
fields, and added the remaining fields progressively.

The result was a solid solution, that can be throttled by scheduling the
execution of the drush scripts using OOTB cron or a proper crontab where the
site would be hosted.

Migrate is a robust module that supports using a highwater mark to "know" where
it left of and pick up only the nodes updated after the last migration.
Unfortunately, the source data lacked the timestamps to do this.

Additionally, if the entire set were to be upserted on every run, that's easy to
do as well with the `--update` flag.

More importantly, a consultant must question everything when approaching a site
build. for instance in my analysis I asked myself the following questions:

- What is this data?
- Should it be volatile or should it persist?
- If it should persist; do other actors (like site editors or site admins) need to be able to
edit it?

## Pros v Cons
The main con of this approach was time. It was a bold move to code a migration,
add content types, add fields and glue it all together.

The main pro was: flexibility, by having the content live in Drupal, one could
tap into Drupal's awesomeness and create listings with Views, format things
quickly using Drupal's robust theming layer. Provide a user friendly experience
for the devs that would need to support the site later on.

This approach future proofs the solution. A 60 minute solution that only coders
can understand and maintain is slighlty less effective than one that can be
improved upon by a wider range of site users.

# Setup
Before enabling this module be aware that composer requirements were used and
they live outside of the scope of this module. You would need to add them
yourself and can do so via composer (or your method of choice).

- `composer require 'drupal/migrate_plus:^4.2'`
- `composer require 'drupal/migrate_tools:^4.5'`
- `composer require 'drupal/address:^1.9'`

# Installing

After downloading this module, enable it via Drush:

`drush pm-enable scalable_migration`

That should recreate all the content types and fields you might need.

You can confirm everything is set up for the migration by running the following
command:

`drush ms`

Then run the following commands to start the migration:

- `drush mim companies --update`
- `drush mim authors --update`
- `drush mim posts --update`

After running those commands one can verify the created content by going to the
content admin screen (`/admin/content`) or the newly created posts listing page (`/posts`).

# What's cool about this?

This test was taken by a twice Drupal Grand Master, with vast experience doing
migrations like this, so a cool couple of tricks can be found in the migration
file itself.

## Authors
See `config/install/migrate_plus.migration.authors.yml`

I purposely made `field_website` a link field. In D8 the link field goes through
validation that would make the JSON data not valid for the field. You should
notice the use of constants and the concat plugin to accommodate that issue.

Next look at `field_address`, which I purposely made and address field to show
some variety in the types of field one can import into.

Then look at `field_company` and notice the use of an entity_lookup plugin to
link out to existing company nodes.

## Companies
See `config/install/migrate_plus.migration.companies.yml`

This migration is almost identical to Authors, since the data does live in the
same JSON payload, but notice how we removed any fields we don't care about.

This migration only pays attention to company data, which could receive any
node id and won't matter to us because we will be using the company name to do
the lookup. This approach isn't flawless of course, but for the sake of the test
it did a great job.

Look at `field_bs` and notice how I chose to make that field a taxonomy term
reference field and in fact generate the tags on the fly during migration.

## Posts
See `config/install/migrate_plus.migration.posts.yml`

Posts is ironically the most boring one. I would only point out that here
the nid is preserved by using the source id and the field id
`field_author/target_id` maps out to  existing nodes directly and not via a
lookup because we didn't create separate fields for their ids and used the OOTB
entity ids. This of course could lead to errors, since there's nothing ensuring
the nid is reserved for a particular node. But again, for the sake of the test
it showed simplicty and effiency.

## Bonus points

If you take a peek inside the Posts view, you will notice how we set the time
based cache expiration there (not programmatically like the test requested)
and also we used the View's UI to display just the right amount of markup as
required by the test. The use of a hidden field and the rewriting of the title
field is what achieves that inline effect the test called for.
