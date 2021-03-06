---
layout: post
date: '2014-08-27T19:09:00+00:00'
title: 'Working with pp: the Ruby Pretty Printer'
tags:
- ruby
- programming
- pp
- printer
- pretty print
redirect_from:
- "/post/95931143377"
- "/post/95931143377/working-with-pp-the-ruby-pretty-printer"
kind: regular
---
{% raw %}<p>Last week, I worked on making the <a href="https://github.com/basho/riak-ruby-client/pull/168">Riak CRDTs print out nicer</a> in the <a href="https://github.com/basho/riak-ruby-client">Riak Ruby client.</a> Since my main interaction with <code>riak-client</code> is through the <a href="http://pryrepl.org"><code>pry</code> REPL</a> and <a href="http://rspec.info">RSpec</a>, I decided to use the <code>pp</code> stdlib library to make the output nicer.</p>{% endraw %}

{% raw %}<p><figure class="tmblr-full" data-orig-height="268" data-orig-width="810" data-orig-src="https://i.bf1c.us/_blog/ugky-pp.png"><img src="https://66.media.tumblr.com/f6b48d0c95fcc0b135443d0d3e1a7f8f/tumblr_inline_pbs2lgJ80P1qzmlsa_540.png" alt="ugly irb output" data-canonical-src="https://i.bf1c.us/_blog/ugky-pp.png" style="max-width:100%;" data-orig-height="268" data-orig-width="810" data-orig-src="https://i.bf1c.us/_blog/ugky-pp.png"/></figure></p>{% endraw %}

{% raw %}<p>Out of the box, a Riak CRDT set looked like:</p>{% endraw %}

```
#<Riak::Crdt::Set:0x007fbec39d87b0 @bucket=#<Riak::Bucket {bucketname},
  @key="cats", @bucket_type="sets", @options={}, @dirty=true>
```

{% raw %}<p>This was terrible for several reasons: the actual value of the set wasn&rsquo;t listed, the <code>object_id</code> is a ridiculous thing, and the bucket type/bucket/key relationship is hard to read. The end goal was something that made it easy to identify which object was which, what it contained, and elide unimportant implementation details.</p>{% endraw %}

{% raw %}<p>Ruby&rsquo;s <code>pp</code> library has some sensible defaults: print out the class name, <code>object_id</code>, and all instance variables. It&rsquo;s relatively good for objects built out of primitives, but tends to break down on deep data structures. The CRDTs for Riak tend to be deeply-nested, especially the Map type, which can contain Counters, Flags, Maps (which can continue recursing down), Registers, and Sets.</p>{% endraw %}

{% raw %}<h1>PP Callbacks</h1>{% endraw %}

{% raw %}<p>The way PP works is a bit convoluted.</p>{% endraw %}

{% raw %}<ol class="task-list"><li>You call <code>pp some_object</code>
</li>
<li>
<code>Kernel.pp</code> calls <code>PP.pp some_object</code>
</li>
<li>
<code>PP.pp</code> instantiates a <code>PP</code> instance, and calls <code>PP#pp some_object</code>
</li>
<li>
<code>PP#pp</code> calls <code>some_object.pretty_print self</code> or <code>some_object.pretty_print_cycle self</code>
</li>
</ol><p>Working with PP on your own objects isn&rsquo;t convoluted: create a <code>pretty_print</code> method that takes a <code>PP</code> instance as its only argument, and calls methods on the instance to output what your users need.</p>{% endraw %}

{% raw %}<p>In the case of a recursion, you don&rsquo;t simply want to repeat the same full representation every time, just enough to know that there&rsquo;s some recursion.</p>{% endraw %}

{% raw %}<h1>PP Methods</h1>{% endraw %}

{% raw %}<p>I ended up using very few methods on <code>PP</code>:</p>{% endraw %}

{% raw %}<ul class="task-list"><li>
<code>object_group</code> wraps its contents with angle brackets and general object information</li>
<li>
<code>breakable</code> inserts a space that can be used as a line break</li>
<li>
<code>comma_breakable</code> inserts a comma followed by a breakable space</li>
<li>
<code>text</code> inserts text</li>
<li>
<code>pp</code> recursively pretty-prints the given value</li>
</ul><h1>Examples</h1>{% endraw %}

{% raw %}<p>On <code>Riak::Crdt::Base</code>, I wanted to provide the basics that <code>Counter</code>, <code>Map</code>, and <code>Set</code> can inherit from. This method prints out the given class, its bucket type, bucket, key, and then yields to the child class:</p>{% endraw %}

{% raw %}<div class="highlight highlight-ruby"><pre># on Riak::Crdt::Base<br/>def pretty_print(pp)
  pp.object_group self do
    pp.breakable
    pp.text "bucket_type=#{@bucket_type}"
    pp.comma_breakable
    pp.text "bucket=#{@bucket.name}"
    pp.comma_breakable
    pp.text "key=#{@key}"{% endraw %}

{% raw %}    yield
  end
end
</pre></div>{% endraw %}

{% raw %}<p>On <code>Riak::Crdt::Map</code>, which inherits from <code>Base</code> above, I wrap map-specific fields in the above method. Since it has five sub-collections, I loop over those to pretty-print them individually.</p>{% endraw %}

{% raw %}<div class="highlight highlight-ruby"><pre># on Riak::Crdt::Map
def pretty_print(pp)
  super pp do
    %w{counters flags maps registers sets}.each do |h|
      pp.comma_breakable
      pp.text "#{h}="
      pp.pp send h
    end
  end
end
</pre></div>{% endraw %}

{% raw %}<p><code>Riak::Crdt::InnerMap</code> doesn&rsquo;t have a bucket type, bucket, or key. It&rsquo;s a simple version of what <code>Map</code> does:</p>{% endraw %}

{% raw %}<div class="highlight highlight-ruby"><pre># on Riak::Crdt::InnerMap
def pretty_print(pp)
  pp.object_group self do
    %w{counters flags maps registers sets}.each do |h|
      pp.comma_breakable
      pp.text "#{h}="
      pp.pp send h
    end
  end
end
</pre></div>{% endraw %}

{% raw %}<h1>Conclusions</h1>{% endraw %}

{% raw %}<p><figure class="tmblr-full" data-orig-height="768" data-orig-width="810" data-orig-src="https://i.bf1c.us/_blog/nice-pp.png"><img src="https://66.media.tumblr.com/29290a9ac8f95b97a4774166fd004efe/tumblr_inline_pbs2lgKvh31qzmlsa_540.png" alt="a nicer, colorized pretty-printed map" data-orig-height="768" data-orig-width="810" data-orig-src="https://i.bf1c.us/_blog/nice-pp.png"/></figure></p>{% endraw %}

{% raw %}<p>Pretty-printing is really nice for debugging and playing with Ruby in a REPL. On top of adding pretty-print support to your code, maybe also encourage users to use <a href="http://pryrepl.org">Pry</a>, which does a great job of colorizing output.</p>{% endraw %}
