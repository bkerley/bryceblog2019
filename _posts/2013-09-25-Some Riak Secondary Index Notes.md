---
layout: post
date: '2013-09-25T14:12:10+00:00'
title: Some Riak Secondary Index Notes
tags: []
redirect_from:
- "/post/62241876936"
- "/post/62241876936/some-riak-secondary-index-notes"
kind: regular
---
{% raw %}<p>Riak Secondary Indexes (2i) are pretty nice. As of Riak 1.4, results come back
in order, can be paginated, and streaming works pretty well with the Ruby
client (with Protocol Buffers and Excon HTTP). There’s a few notes and gotchas
though.</p>{% endraw %}

{% raw %}<p>For these examples, you have a bucket with a hundred keys in it, numbered 1-100,
and indexed by that number.</p>{% endraw %}

{% raw %}<h2>Secondary Indexes Aren&rsquo;t Consistent</h2>{% endraw %}

{% raw %}<p>2i makes the same consistency guarantees as Riak itself. If you’ve queried for 0-200, which matches all 100 records,
you might not get a hundred records back. If, by the time the index scan hits
50, you’ve deleted, records 67, 15, and 86, you might not get 67 or 86, and
depending on how fast you deleted it, you might not have got 15 either. If
somebody adds records, they may or may not show up in the results either.</p>{% endraw %}

{% raw %}<h2>Pagination is nice but not as complete as SQL</h2>{% endraw %}

Riak 1.4 added pagination for secondary indexes. It’s not as nice as
traditional pagination as seen in
<a href="https://rubygems.org/gems/will_paginate">will_paginate</a>, which has the luxury
of making SQL queries:

```sql
SELECT COUNT(*)
    FROM posts
    WHERE
        published_at IS NOT NULL AND
        user_id = 12345;

SELECT *
    FROM posts
    WHERE
        published_at IS NOT NULL AND
        user_id = 12345
    LIMIT 5
    OFFSET 30;
```

Riak 2i has no equivalent of the former short of fetching all the keys, and
if it did implement it, you’d be better off just querying for all
the posts in range and paginating in-client.

```ruby
i = Riak::SecondaryIndex.new(
    posts_bucket,
    'user_publish_bin',
    ('12345_0000000000'..'12345_1380074400')
    )

# these calculations almost certainly contain bugs
total_pages = i.keys.length / 5
offset = params[:page] * 5
current_page = pages[offset..(offset + 5)]

posts = posts_bucket.get_many current_page
```

{% raw %}<p>With that said, the pagination features are useful if you don’t
mind jamming client state into links: the continuation slug from pagination
means that users that stop and read posts won’t see the same post at the top
of the next page when you make a new one. If you can get away with “Previous,” “Next,” and maybe a list of previous pages, pagination is right up your alley.</p>{% endraw %}

{% raw %}<h2>Streaming is Useful</h2>{% endraw %}

{% raw %}<p>Riak 1.4 also brings streaming to 2i; you can get little lumps of keys
delivered to your client as Riak sorts them, instead of all at once. If you’re
feeding these keys into a processing queue to be handled elsewhere, this is
nice, can save you some memory (and therefore GC pauses), and isn’t even
difficult.</p>{% endraw %}

```ruby
i = Riak::SecondaryIndex.new buck, 'index_int', 0..50

i.keys {|k| puts k.inspect}
# ["0", "1", "2", "3", "4", "5", "6"]
# ["7", "8", "9", "10", … "49"]
# []
```

{% raw %}<p>You&rsquo;ll notice in this case that the stream is chunky, and that the chunks aren&rsquo;t evenly sized. What happened is that the first few results became available right away, and by the time that message was out the door, all the rest of the results were ready to go.</p>{% endraw %}

{% raw %}<p>The only caveat is that it’s not a “get out of consistency jail free” card.</p>{% endraw %}

{% raw %}<h2>How It Works</h2>{% endraw %}

{% raw %}<p>Secondary indexes are stored much like regular data, but instead of key/value
pairs, they’re index_key/key pairs. A range query in a vnode involves seeking through
leveldb to where the start of the range should be, and reading all the entries
until it passes the end of the range. The entries read by each vnode are then
merge-sorted by the index state machine, and finally returned to the client.</p>{% endraw %}

{% raw %}<h2>Recommended Reading</h2>{% endraw %}

{% raw %}<ul><li><a href="https://github.com/basho/riak-ruby-client/tree/1-4-stable#secondary-index-examples">Riak Ruby Client 2i docs</a>, high-level usage.</li>
<li><a href="http://leveldb.googlecode.com/svn/trunk/doc/impl.html">LevelDB implementation</a>, so you know what each vnode has to deal with.</li>
<li><a href="https://github.com/basho/riak_kv/blob/1.4/src/riak_kv_index_fsm.erl#L101">Riak KV Index FSM</a>, where the results are actually merged and processed.</li>
</ul>{% endraw %}
