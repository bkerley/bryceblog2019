---
layout: post
date: '2013-02-27T01:38:35+00:00'
title: 'Ruby: Blocks and Procs'
tags:
- ruby
redirect_from:
- "/post/44105519238"
- "/post/44105519238/ruby-blocks-and-procs"
kind: regular
---

<p>Ruby has two kinds of anonymous function: blocks, and procs.</p>
<dl><dt>Block</dt>
<dd>A block is a syntactic element. The only things you can do with these are:
<ul><li>pass them to methods as the &ldquo;default block&rdquo; argument</li>
<li>in a method that&rsquo;s been called with a default block, <code>yield</code> zero or more arguments to it, getting a result</li>
<li>in a method that&rsquo;s been called with a default block, capture it into a <code>Proc</code> with an ampersand in the arguments list</li>
</ul></dd>
<dt>Proc</dt>
<dd>A <code>Proc</code> is an object. You can do just about anything with it, including turn it back and forth from a block.</dd>
</dl><h1>Developing with Blocks</h1>
<p>Using blocks with somebody else&rsquo;s API is intuitive:</p>
```ruby
(1..5).map{|n| n * 3} #=> [3, 6, 9, 12, 15]
```

<p>Making an API that uses blocks can be easy too:</p>

```ruby
module Object
  def with
    yield self
  end
end

# calling
"hello".with{|h| puts h}
# prints "hello"
```

<p>If you <code>yield</code> more than once, you run the block multiple times.</p>
<h1>Developing with Procs</h1>
<p>How do you even make a <code>Proc</code>?</p>

```ruby
doubler = proc{|n| n * 2}
#=> #<Proc:0x007f8e938d8618@(irb):1>
```

<p>There&rsquo;s a few methods, like <code>proc</code>, that turn a block into a <code>Proc</code>. <code>lambda</code> is very similar. They&rsquo;re easy to write:</p>

```ruby
def my_proc(&block)
  block
end

tripler = my_proc{|n| n * 3}
#=> #<Proc:0x007f8e9506d170@(irb):5>
```

<p>The ampersand in the argument list to <code>my_proc</code> is what transforms the block into a <code>Proc</code>, and then we simply return it.</p>

<p>You can use the ampersand to turn a <code>Proc</code> into a block too:</p>

```ruby
(1..5).map doubler
# ArgumentError: wrong number of arguments(1 for 0)

(1..5).map &doubler
#=> [2, 4, 6, 8, 10]
```

<p>And you can call a <code>Proc</code> directly without having to turn it into a block too:</p>

```ruby
doubler.call 4
#=> 8

doubler[5]
#=> 10

doubler.(6) # the dot before the parentheses is significant
#=> 12
```

<p>Blocks and <code>Proc</code>s can be confusing. I hope this helps!</p>
